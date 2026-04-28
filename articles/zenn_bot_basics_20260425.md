---
title: "GMO Coin APIで仮想通貨自動売買Botを動かす仕組み：Donchianチャネル戦略の実例"
emoji: "📈"
type: "tech"
topics: ["python", "仮想通貨", "自動売買", "bot", "gmocoin"]
published: true
---

# GMO Coin APIで仮想通貨自動売買Botを動かす仕組み：Donchianチャネル戦略の実例

仮想通貨の自動売買Botに興味はあるけれど、「どこから始めればいいのか分からない」という方は多いと思います。

この記事では、GMO Coin APIを使った自動売買Botの基本的な仕組みを、実際に稼働中のDonchianチャネル戦略を例にとって解説します。Python初〜中級者を対象に、概念とコードのイメージを掴んでいただくことを目標にしています。

---

## なぜGMO Coin APIを選ぶのか

国内主要取引所のAPIは複数ありますが、Bot開発においてGMO Coinが使いやすい理由がいくつかあります。

**1. APIが無料で使える**
口座を開設するだけで、REST APIが無料で利用できます。認証なしで使えるPublic API（現在価格・板情報・歩み値）と、認証が必要なPrivate API（残高照会・注文発注・ポジション管理）が揃っています。

**2. 公式ドキュメントが整備されている**
[GMO Coin公式APIドキュメント](https://api.coin.z.com/docs/)はリクエスト/レスポンスの仕様が詳しく、サンプルコードも掲載されています。

**3. 現物とレバレッジ取引の両対応**
現物取引だけでなく、レバレッジ取引にも対応しています。Botの戦略に合わせて選択できます。

**レート制限について**: Private APIの呼び出し上限は1秒あたり最大20回です（2023年12月改定）。1時間足ベースの戦略なら制限を気にする必要はほとんどありません。

---

## Bot運用の基本構成

シンプルなBotは以下の4ステップで動いています。

```
[1. 定期実行] → [2. データ取得] → [3. シグナル判定] → [4. 注文/ログ記録]
```

### 1. 定期実行

cronを使って1時間ごとにPythonスクリプトを実行します。

```bash
# 毎時10分に実行する例
10 * * * * /home/user/.venv/bin/python3 /home/user/strategies/donchian_trader.py run ETH_JPY
```

### 2. データ取得

GMO Coin APIからOHLCV（始値・高値・安値・終値・出来高）の1時間足データを取得します。

```python
import urllib.request
import json
from datetime import datetime

def get_ohlcv(symbol, interval="1hour", date=None):
    # date は "YYYYMMDD" 形式。省略すると最新データを取得
    date_str = date or datetime.now().strftime("%Y%m%d")
    url = f"https://api.coin.z.com/public/v1/klines?symbol={symbol}&interval={interval}&date={date_str}"
    with urllib.request.urlopen(url) as response:
        data = json.loads(response.read())
    return data["data"]
```

標準ライブラリのみで書けるため、追加インストール不要なのもメリットです。

### 3. シグナル判定

取得したデータを使って「買う／売る／保有継続」を判断します（Donchianチャネルの詳細は後述）。

### 4. 注文/ログ記録

判断結果に応じて注文APIを呼ぶか、ペーパートレード（仮想売買）としてログに記録します。開発初期はペーパートレードで安全に動作確認するのがおすすめです。

---

## Donchianチャネル戦略とは

Donchianチャネルは、一定期間の**最高値と最安値でチャネルを形成**し、価格がそれを突破したときにトレンドフォローするシンプルな戦略です。

```
上限チャネル = 過去N本の最高値
下限チャネル = 過去M本の最安値
```

実際に稼働中のETH/JPY Botでは、以下のパラメータを使っています。

| パラメータ | 値 | 意味 |
|-----------|-----|------|
| エントリー期間 | 48本（1時間足） | 過去48時間の最高値を上抜けで買い |
| 決済期間 | 15本（1時間足） | 過去15時間の最安値を割れで売り |
| 利確幅 | +3.0% | エントリー価格から3%上昇で利確 |
| 損切り幅 | -1.5% | エントリー価格から1.5%下落で損切り |
| 最大保有時間 | 120時間 | 5日間保有したら強制決済 |

判定ロジックのイメージは以下の通りです。

```python
def judge_signal(ohlcv_data):
    closes = [float(c["close"]) for c in ohlcv_data]
    highs  = [float(c["high"])  for c in ohlcv_data]
    current_price = closes[-1]

    # エントリー: 過去48本の最高値を上抜け
    entry_threshold = max(highs[-49:-1])  # 直近を除く48本

    if current_price > entry_threshold:
        return "BUY"
    return "HOLD"
```

シンプルな構造なのに、適切なパラメータ設定でバックテストPF（プロフィットファクター）1.2以上を達成できるケースがあります。

---

## ペーパートレードで安全に検証する方法

本番の資金を使う前に、**ペーパートレード**（仮想売買）で動作確認することを強くおすすめします。

ペーパートレードの仕組みは簡単です。

```python
DRY_RUN = True  # True: ペーパートレード, False: 本番注文

def execute_order(side, symbol, price):
    if DRY_RUN:
        # 実際には注文せず、ログに記録するだけ
        log(f"[PAPER] {side} {symbol} @ {price}")
        return
    # ここに実際の注文APIコールを書く
    ...
```

`DRY_RUN = True` の状態でcronを動かすことで、注文ロジックやシグナル判定の挙動をリアルタイムで確認できます。

ログはTSV形式で毎時記録されます。

```
2026-04-25 08:10:05	ETH_JPY	hold	上限=382,702 下限=368,395, 条件未達
2026-04-25 07:10:04	ETH_JPY	hold	上限=383,470 下限=368,395, 条件未達
```

このログを見ることで、「何時間に何回エントリー条件が発生したか」「利確と損切りの比率」などを把握できます。

---

## 実績の一端：ETH/JPY Donchianチャネルの場合

OSSのマルチエージェントシステムを活用して運用中のBot群では、ETH/JPYのDonchianチャネル戦略を含む複数のBotが現在ペーパートレード稼働中です。

バックテスト（2022年〜2024年のIS期間）では、手数料（Taker 0.05%）込みでPF 1.2以上を達成した戦略のみを候補に採用しています。

ペーパートレードの判定ログはGitHubなどで管理することで、後から「どの相場環境でどのシグナルが発生したか」を振り返ることができます。

BT合格後、一定期間のペーパー稼働で動作確認してから本番移行する流れです。焦らず積み上げることが長期運用のコツです。

---

## まとめ

GMO Coin APIを使った自動売買Botの基本構成は、

1. **cronで定期実行**（1時間足なら毎時）
2. **APIでOHLCVデータを取得**（標準ライブラリで実装可能）
3. **シグナル判定**（Donchianチャネルなどのルールベース）
4. **ペーパートレードで動作確認** → 本番移行

というシンプルな流れです。

「仕組みはわかった、次は実装の詳細を知りたい」という方に向けて、GMO Coin APIの認証設定・バックテストフレームワーク・複数Bot管理の実装まで、コードレベルで解説するZenn Booksも今後公開予定です。公開時はこちらの記事でご案内します。

---

*本記事のBotは教育目的のサンプルです。実際の投資はご自身の判断と責任のもとで行ってください。*
