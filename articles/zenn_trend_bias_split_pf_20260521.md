---
title: "CFD自動売買のトレンドバイアス問題：IS分割PFテストで見抜く過学習"
emoji: "📈"
type: "tech"
topics: ["自動売買", "バックテスト", "claudecode", "python"]
published: true
---

## はじめに

CFD（差金決済取引）の自動売買において、バックテストが優秀なのにライブで失敗するケースのひとつに**トレンドバイアス問題**があります。

これは「テスト期間中にたまたまトレンドが強い相場が続いており、それに乗っかっただけで好成績を出したロング戦略が、過学習した戦略として誤検知されにくい」という問題です。

この記事では、AIを活用して開発・運用しているCFDバックテストフレームワーク `auto_bt_runner_cfd.py` に実装した2つの検出手法を紹介します。

- **E2（trend_bias_suspect）**: シンボルごとのIS期間トレンド強度を定数管理し、ロング比率>80%で警告フラグを立てる
- **E1（is_first_half_pf）**: ISを前半・後半に分割し、相場が比較的横ばいだった前半でもPF≥1.0を要求する

---

## 問題の背景：XAUUSDのIS期間は+42.6%上昇していた

バックテストフレームワークでは、以下の期間設定を採用しています。

```python
IS_START  = "2022-01-01"
IS_END    = "2024-12-31"
IS_FIRST_HALF_START = "2022-01-01"
IS_FIRST_HALF_END   = "2023-06-30"
OOS_START = "2025-01-01"
OOS_END   = "2025-12-31"
```

IS期間（2022〜2024年）はXAUUSD（金）にとって強烈な上昇相場でした。コードで確認できるように、IS期間の価格変化率を定数として管理しています。

```python
# IS期間(2022-01-01〜2024-12-31)のシンボル別価格変化率（trend_bias_suspect判定用）
SYMBOL_IS_TREND = {
    "XAUUSD": 0.426,   # +42.6%（強い上昇トレンド）
    "XAGUSD": 0.248,   # +24.8%（銀も貴金属上昇トレンド）
    "NATGAS": -0.031,  # -3.1%
    "UKOIL":  -0.041,  # -4.1%
    "USOIL":  -0.041,  # UKOIL近似
}
```

XAUUSDがIS期間に+42.6%上昇していたということは、単純に「常にロング」するだけで大きな利益が出る環境だったことを意味します。このような相場でバックテストをすると、「ロング偏重の戦略」が優秀に見えてしまいます。

---

## E2：trend_bias_suspect フラグ

### 仕組み

バックテスト結果を集計する段階で、以下の判定ロジックを実行しています。

```python
is_long_count = is_metrics.get("long_trade_count", 0)
long_trade_pct = (is_long_count / is_trades) if is_trades > 0 else 0.5
trend_bias_suspect = (
    long_trade_pct > 0.8 and SYMBOL_IS_TREND.get(symbol, 0.0) > 0.20
)
```

条件は2つの組み合わせです。

1. **IS期間のロング比率が80%超**（`long_trade_pct > 0.8`）
2. **そのシンボルのIS期間トレンドが+20%超**（`SYMBOL_IS_TREND.get(symbol, 0.0) > 0.20`）

XAUUSDで「ロング取引が全体の80%以上を占める戦略」は、+42.6%の上昇トレンドに乗っかっているだけの可能性が高い。そのような戦略は `trend_bias_suspect: True` として報告書にフラグが立ちます。

### なぜ「即不合格」ではなく「フラグ」なのか

`trend_bias_suspect` はあくまでも**警告フラグ**であり、合格/不合格の判定には直接使いません。なぜなら：

- ロング比率が高くても、純粋にロング優位な市場構造を捉えた戦略が存在する
- 実際の排除は次に説明するE1（IS前半PF）に委ねる方が客観的

フラグは人間がレビューするときの注意喚起として機能し、レポートの `trend_bias_suspect` フィールドに記録されます。

---

## E1：IS前半分割PFテスト

### 問題意識

IS期間を通しでバックテストすると、後半（2023年7月〜2024年12月）の急激な上昇相場が成績を押し上げてしまいます。前半（2022年1月〜2023年6月）のXAUUSDは比較的横ばいで推移していました。

「横ばい期にも利益を出せるか？」を確認することで、「単なるトレンドライダー」を事前排除できます。

### 実装

IS前半期間のバックテストを追加で実行しています。

```python
# E1: IS前半期間(2022-2023H1)で分割PFテスト
first_half_metrics = _run_single_period_bt_cfd(
    compute_fn, make_fn,
    symbol, timeframe, IS_FIRST_HALF_START, IS_FIRST_HALF_END, init_cap, lot_size,
    direction=direction,
)
is_first_half_pf = first_half_metrics.get("profit_factor", 0.0) if first_half_metrics else 0.0
is_first_half_trades = first_half_metrics.get("total_trades", 0) if first_half_metrics else 0
if is_first_half_trades < 10:
    print(f"    ⚠ {symbol}: IS前半取引数={is_first_half_trades}件（統計信頼性低）", flush=True)
```

前半期間の取引数が10件未満の場合は統計信頼性が低いとして警告を出します。

### 合格基準への組み込み

IS前半PF≥1.0を合格条件に追加しています。

```python
sym_passed = (
    is_pf >= PASS_PF and           # IS全体 PF≥1.20
    oos_pf >= PASS_PF_OOS and      # OOS PF≥0.90
    oos_ratio >= PASS_OOS_RATIO and # OOS/IS比率≥0.7
    total_trades >= PASS_TRADES and
    max_dd_combined <= sym_max_dd and
    (min_calmar == float("inf") or min_calmar >= PASS_CALMAR) and
    oos_trades >= oos_min and
    (is_conc_pf == float("inf") or is_conc_pf >= 1.0) and
    is_ir >= PASS_IR and
    is_first_half_pf >= 1.0        # ← E1: IS前半でもPF≥1.0
)
```

この条件によって、**IS全体のPFが1.20以上でも、前半だけで負けていた戦略は不合格**になります。

---

## 2つの手法の使い分け

| 手法 | 判定方法 | 使い方 |
|------|----------|--------|
| E2（trend_bias_suspect） | ロング比率×トレンド強度 | 警告フラグ（人間レビュー用） |
| E1（is_first_half_pf） | IS前半期間のPF | 合格基準（自動排除） |

E2は「この戦略はトレンドバイアスの疑いがある」と気づくための**人間向けシグナル**です。E1は「前半横ばい期にも機能するか」を**定量基準で自動チェック**します。

組み合わせることで、トレンドライダー疑いの戦略を多角的に排除できます。

---

## レポート出力の確認

バックテスト実行後のレポートには、以下のフィールドが含まれます。

```python
entry = {
    ...
    "long_trade_pct": round(long_trade_pct, 3),         # ロング比率（例: 0.847）
    "trend_bias_suspect": trend_bias_suspect,            # True/False
    "is_first_half_pf": round(is_first_half_pf, 3) if is_first_half_pf != float("inf") else 999.0,
    "is_first_half_trades": is_first_half_trades,        # 前半取引数
}
```

`trend_bias_suspect: True` の戦略は、たとえ他の指標が合格ラインを超えていても、その成績が「XAUUSDの上昇トレンドにただ乗りしただけではないか」と人間が確認する起点になります。

---

## まとめ

CFDバックテストにおけるトレンドバイアス問題への対応として、2つのアプローチを実装しました。

1. **SYMBOL_IS_TREND定数管理**：シンボルごとのIS期間トレンド強度を一元管理し、保守性を確保
2. **trend_bias_suspect フラグ**：ロング比率>80% かつ トレンド>20% で警告。即排除ではなく人間レビューに活用
3. **is_first_half_pf 合格基準**：IS前半（横ばい期）でもPF≥1.0を要求。トレンドライダーを自動排除

「バックテストで良い成績が出た戦略を信じる」だけでは不十分で、「その成績がどんな相場環境で生まれたか」を分解することが重要です。この2つの手法は、相場環境依存のバイアスを定量的に検出するための実践的なアプローチです。

