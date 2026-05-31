---
title: "バックテストを78%削減した — LightGBMサロゲートモデルで仮想通貨Bot候補を事前フィルタリングする実装"
emoji: "🤖"
type: "tech"
topics: ["LightGBM", "機械学習", "仮想通貨", "バックテスト", "Python"]
published: true
---

<!-- sources: evaluation_report.json / train_categorical_surrogate.py / auto_bt_runner.py -->

バックテストは重い処理です。シグナルを15本生成して、それぞれをIS期間・OOS期間でテストすると、1回の夜間バッチで数時間かかります。

この記事では、過去のバックテスト結果から「合格しそうな候補」をLightGBMで予測し、**BT実行前に78%の候補を棄却する**仕組みの実装を紹介します。

## 問題: バックテストがボトルネック

自動売買Botの開発では、アルファ（利益機会）を探索する際に大量の候補をバックテストする必要があります。

筆者が運用しているGMO coin向けのシステムでは、毎夜15候補を生成してIS/OOS評価を行っています。合格率は約2.75%（2,036候補中56合格）です。

つまり、**97.25%の候補はBTを走らせても不合格**です。このムダを削れれば、同じ計算コストでより多くの候補を探索できます。

```
合格率: 56 / 2036 = 2.75%
→ 100候補BTしても2〜3本しか合格しない
→ 合格しない97本のBT時間が丸ごとムダ
```

## アプローチ: メタモデルで事前フィルタリング

アイデアはシンプルです。

**過去のBT結果をトレーニングデータとして、「この戦略タイプ × 通貨ペア × 時間足の組み合わせは合格しやすいか？」を学習させる。**

これが「サロゲートモデル（代理モデル）」と呼ばれるアプローチです。本物のBTを実行する代わりに、安価な予測で事前に候補を絞ります。

## 特徴量: たった3つのカテゴリ変数

驚くほどシンプルで、特徴量は以下の3つだけです。

| 特徴量 | 例 |
|--------|-----|
| strategy_type | mean_reversion, donchian_breakout など |
| symbol | BTC_JPY, ETH_JPY, XRP_JPY |
| interval | 15min, 1hour, 4hour |

「BT結果の数値（PF, Calmar率など）は使わないの?」と思われるかもしれません。

使いません。理由は2つあります。

1. BT前の時点では当然BT結果は存在しない（推論時に使えない）
2. 「どの組み合わせが歴史的に合格しやすいか」という構造的パターンを学びたい

## 実装コード

### 学習（train_categorical_surrogate.py）

```python
import csv, json, pickle
from pathlib import Path
import pandas as pd
import lightgbm as lgb
from sklearn.model_selection import StratifiedKFold
from sklearn.metrics import roc_auc_score
from sklearn.preprocessing import LabelEncoder
import numpy as np

ML_DIR = Path(__file__).parent
TRAINING_CSV = ML_DIR / "training_data.csv"
MODEL_PATH   = ML_DIR / "categorical_surrogate.pkl"

def train():
    df = pd.read_csv(TRAINING_CSV)
    cat_cols = ["strategy_type", "symbol", "interval"]
    
    # LabelEncoderでカテゴリ変数をエンコード
    encoders = {}
    for col in cat_cols:
        le = LabelEncoder()
        df[col + "_enc"] = le.fit_transform(df[col].astype(str))
        encoders[col] = le

    X = df[[c + "_enc" for c in cat_cols]]
    y = df["passed"].values

    # 不均衡対策: scale_pos_weight（合格2.75%→不合格97.25%）
    neg, pos = (y == 0).sum(), (y == 1).sum()
    
    # 5-fold Stratified CV
    skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    aucs = []
    for train_idx, val_idx in skf.split(X, y):
        model = lgb.LGBMClassifier(
            n_estimators=200, learning_rate=0.05, max_depth=4,
            scale_pos_weight=neg/max(pos, 1), random_state=42, verbose=-1
        )
        model.fit(X.iloc[train_idx], y[train_idx],
                  eval_set=[(X.iloc[val_idx], y[val_idx])],
                  callbacks=[lgb.early_stopping(20, verbose=False)])
        proba = model.predict_proba(X.iloc[val_idx])[:, 1]
        aucs.append(roc_auc_score(y[val_idx], proba))
    
    print(f"CV AUC: {np.mean(aucs):.3f} ± {np.std(aucs):.3f}")
    
    # 全データで最終モデルを学習
    final_model = lgb.LGBMClassifier(
        n_estimators=200, learning_rate=0.05, max_depth=4,
        scale_pos_weight=neg/max(pos, 1), random_state=42, verbose=-1
    )
    final_model.fit(X, y)
    
    # pickle保存（LabelEncoderも一緒に）
    with open(MODEL_PATH, "wb") as f:
        pickle.dump({
            "model": final_model,
            "encoders": encoders,
            "cat_cols": cat_cols,
            "enc_cols": [c + "_enc" for c in cat_cols],
        }, f)
```

### 推論（auto_bt_runner.py内）

```python
import pickle
from pathlib import Path

# モデル読み込み
ml_dir = Path(__file__).parent / "ml"
surrogate_path = ml_dir / "categorical_surrogate.pkl"

if surrogate_path.exists():
    with open(surrogate_path, "rb") as f:
        surrogate_data = pickle.load(f)  # lightgbmがpythonパスにないとここでエラー
    print("✅ サロゲートモデル読み込み完了")
else:
    print("⚠️  サロゲートモデル未存在。フィルタリング無効化。")
    surrogate_data = None

# 候補ごとに判定
SURROGATE_THRESHOLD = 0.30  # 低めに設定→高recall

for candidate in generated_candidates:
    if surrogate_data is not None:
        enc = surrogate_data["encoders"]
        feat_row = {
            "strategy_type_enc": enc["strategy_type"].transform([candidate["type"]])[0],
            "symbol_enc":        enc["symbol"].transform([candidate["symbol"]])[0],
            "interval_enc":      enc["interval"].transform([candidate["interval"]])[0],
        }
        import pandas as pd
        feat_df = pd.DataFrame([feat_row])
        prob = surrogate_data["model"].predict_proba(feat_df)[0][1]
        
        if prob < SURROGATE_THRESHOLD:
            print(f"🔵 SR棄却: {candidate['symbol']}/{candidate['interval']} (score={prob:.2f})")
            continue  # BTをスキップ
    
    # BTを実行
    run_backtest(candidate)
```

## 結果: 78.5%の候補を棄却

評価結果（evaluation_report.json）:

```
データ: 2,036件（合格=56, 合格率=2.75%）
5-fold CV AUC: 0.753 ± 0.048

閾値0.30での性能:
  precision:     12.8%（サロゲートが通過させた候補のうち、実際に合格する割合）
  recall:       100.0%（合格候補を1本も見逃さない）
  rejection_rate: 78.5%（候補の78.5%をBT前に棄却）
```

閾値0.30という低めの設定が重要なポイントです。

「合格しやすい」という確信がなくても合格可能性がある候補はすべてBTを実行します。**精度（precision）よりも再現率（recall）を最優先**した設計です。

合格候補を見逃すリスクよりも、ムダなBTを削減することが目的なので、この設計は理にかなっています。

実際のBT削減効果:
- 15候補 → 78.5%棄却 → 実際にBTするのは約3.2候補（15×(1-0.785)）
- 探索効率: 理論上約5倍（15÷3.225）

## 実装時のハマりポイント

### pickle + LightGBM の環境依存問題

`categorical_surrogate.pkl` はLightGBMモデルをpickle化したものです。このファイルをロードするには、**実行するPython環境にlightgbmがインストールされている必要があります**。

筆者のケースでは、学習環境（`.venv_ml`）とBT実行環境（`.venv_quant`）でvenvを分けていました。

```
学習: .venv_ml/bin/python3 → lightgbm ✅
実行: .venv_quant/bin/python3 → lightgbm ❌（インストール忘れ！）
```

この状態でcronが夜間に実行されると、以下のエラーが出て静かに失敗します:

```
ModuleNotFoundError: No module named 'lightgbm'
```

手動実行は別のシェルでvenvをactivateして動いていたため、「手動は成功・cron失敗」という謎の症状になりました。

教訓: **pickle化されたMLモデルを別venvで使う場合、依存ライブラリのインストールを忘れずに確認する。**

```bash
# cronで使うpythonにlightgbmがあるか確認
/path/to/.venv_quant/bin/python3 -c "import lightgbm; print(lightgbm.__version__)"
```

### 学習データの品質が肝

2,036件のサンプルで学習したとはいえ、合格率2.75%（56/2036）という極端な不均衡データです。

`scale_pos_weight = 不合格数 / 合格数 = 1980 / 56 ≈ 35` という重み付けがないと、モデルはすべてを「不合格」と予測するだけになります。

LightGBMの`LGBMClassifier`には`scale_pos_weight`パラメータがあり、これを設定することで不均衡データを扱えます。

### 未知のカテゴリ（新戦略追加時）

新しい戦略タイプを追加した場合、LabelEncoderが未知のカテゴリに遭遇してエラーになります。

```python
# 対策: 未知カテゴリをハンドリング
try:
    enc_val = encoder.transform([category])[0]
except ValueError:
    # 未知カテゴリ → サロゲートフィルタをスキップしてBT実行
    enc_val = None
```

実装では未知カテゴリが来た場合はサロゲートフィルタをパスする設計にしています。

## まとめ

3つのカテゴリ変数だけで**78%のBTをスキップ**できる、シンプルで効果的なサロゲートモデルの実装を紹介しました。

ポイントをまとめると:

- 特徴量は「戦略タイプ×通貨ペア×時間足」の3つのみ
- LGBMClassifier + scale_pos_weight（不均衡対策）
- 閾値は低め（0.30）でhigh-recall設計
- pickleロード時はlightgbmのvenv確認が必須

サロゲートモデルは探索効率化の初歩ですが、実際の運用では蓄積されたBTデータが増えるほど精度が向上します。2,036件でAUC=0.753でしたが、データが倍になればさらに改善が期待できます。

---

## 関連記事

- [仮想通貨Botの合格基準を統計学で検証してみた](https://zenn.dev/ashigaru4)
- [Zenn Vol.1: 11Botが全滅した日 — GMO移行の記録](https://zenn.dev/ashigaru4)
