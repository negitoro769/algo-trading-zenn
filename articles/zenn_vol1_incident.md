---
title: "11Botをサイレント停止させた2時間"
emoji: "🤖"
type: "tech"
topics: ["python", "cron", "自動売買", "運用", "claudecode"]
published: true
---

:::message
この記事は仮想通貨Bot運用連載の **Vol.1** です。連載の概要については [Vol.0（序章）](https://zenn.dev/ai_jidoka_lab/articles/zenn_intro_overview) をご覧ください。
:::

## はじめに

2026年4月11日の午後、週次レビューのドキュメントを読み返していたとき、違和感を覚えました。

「18Bot稼働中、累計収支+1,770円」そう書かれているのに、なぜか実感と合わない。

各Botのログファイルの最終更新時刻を直接確認してみると

| Bot番号 | 最終実行日時 |
|---------|------------|
| #2 ETH Donchian | 2026-04-05 00:10 |
| #4 BTC Donchian | 2026-04-04 23:30 |
| #5 ETH BB+ADX | 2026-04-04 23:40 |
| #7 CCI XRP | 2026-04-04 23:55 |
| ... | ... |

18本中11本が、4月4日〜5日を最後に沈黙していました。約6.5日間（156時間）、11本のBotがサイレント停止していたのです。

この記事では、その原因調査から初動対応、再発防止設計の検討まで約2時間の記録を書きます。

---

## 状況：稼働システムの構成

筆者はOSSのマルチエージェントシステム（[multi-agent-shogun](https://github.com/yohey-w/multi-agent-shogun)）をforkして活用し、仮想通貨・FXの自動売買システムを運用しています。PythonスクリプトをWSL2上のcronで定期実行し、GMOコインのペーパートレードAPIに注文を投げる構成です。

エージェントシステムはClaudeで動いており、通常はKaro（家老）が全体を管理し、Ashigaru（足軽）が個別タスクをこなす形で動いています。

稼働中のBotは18本。

| 区分 | Bot | 状態 |
|------|-----|------|
| **停止11本** | #1 KKW AUD/JPY、#2 ETH Donchian48、#3 ETH Volume Breakout、#4 BTC Donchian96、#5 ETH BB+ADX+MA100、#6 BTC Keltner Breakout、#7 CCI XRP、#8 CCI ETH、#9 CCI BTC、#10 Donchian48 ETH、#11 Donchian48 XRP | 4/5以降 実行ゼロ |
| **稼働7本** | multipos_incr_eth_jpy、multipos_dca_btc 等 | 通常稼働継続 |

---

## 原因調査：なぜ動いていないのか

最初に疑ったのは「APIが落ちたのか？」でした。しかしGMOコインのAPIステータスページに障害履歴はありません。

次に各Botスクリプトを **手動実行** してみました。するとエラーなく動作します。

「スクリプト自体は壊れていない。では、なぜcronからは起動されないのか」

`crontab -l`を確認したところ

```bash
$ crontab -l | grep cron_gmo
10 * * * * /home/user/projects/gmo/strategies/cron_gmo_donchian.sh
...
```

停止している11本のエントリが **見当たりません** 。crontabから消えていました。

根本原因は明快でした。 **crontabから11本のBotエントリが消失していた** のです。

cron daemonは存在しないエントリを実行しません。手動では動くのに、cronからは一切起動されない——これが6.5日間、誰にも気づかれなかったサイレント停止の正体でした。

消失の推定時刻は2026-04-05 00:00〜09:00の深夜帯です（最後の正常ログが4/4 23:20〜4/5 00:10であることから）。何らかのcrontab編集が行われ、11本を残したまま新しい状態で上書きされた疑いがありますが、直接の犯人ログは現時点で特定できていません。

---

## なぜ6.5日も気づかなかったか：3層の盲点

調査を通じて、より深い問題が浮かび上がりました。監視の仕組みに構造的な盲点があったのです。

### 盲点①：ヘルスチェックスクリプトの監視範囲が3本のみ

`scripts/check_cron_health.sh`には、GMO Botのログをチェックするためのリストが **ハードコードされていました** 。

```bash
GMO_BOT_LOGS=(
    /home/multi-agent-shogun/projects/gmo/multipos_incr_eth_jpy.log
    /home/multi-agent-shogun/projects/gmo/multipos_dca_btc.log
    /home/multi-agent-shogun/projects/gmo/pyramid_dry_run.log
)
```

監視対象はわずか3本。停止した11本はそもそも監視されていなかったため、ログのmtimeがいくら古くなっても警告が出ませんでした。

### 盲点②：cron_registry.txtに11本が未登録

`cron_registry.txt`はcronエントリの管理ドキュメントとして機能していましたが、 **4月1日のBot量産時から4月11日の復旧まで、11本が一度も登録されていませんでした** 。

レジストリに存在しないエントリが消えても、差分チェックには何も映りません。

### 盲点③：「All healthy」の誤った安全宣言

ヘルスチェックスクリプトは正直に結果を返していました。

```
[OK] All cron jobs are healthy
```

しかし実態は「監視対象3本のうち3本が正常」だっただけです。正確には`[OK] 3/3 monitored bots healthy (NOTE: 15 bots not monitored)`と表現すべきでした。

**結果：監視3本 + レジストリ未登録 = 11本が二重に監視対象外** 。エージェントも家老も、スクリプトの出力を信頼して「正常」と判断し続けていました。

---

## 復旧手順

13:15、crontabを手動で復旧しました。消失していた11本のエントリを戻し、`scripts/cron_registry.txt`も同時に更新。

```bash
# 復旧前にバックアップ取得
crontab -l > logs/crontab_backup_before_restore_20260411.txt

# エントリを復旧後、13:15:04に#9 CCI BTCが初発火
# → 復旧成功を確認
```

`logs/crontab_backup_before_restore_20260411.txt`に復旧前の状態を記録。後日の調査に備えました。

---

## 再発防止策：即時対応と中長期計画

復旧後、約2時間かけて再発防止の設計を検討しました。

### 即時対応

**A. `check_cron_health.sh`の監視対象を自動探索化**

ハードコードされたリストを廃止し、`cron_registry.txt`または`crontab -l`から動的に監視対象を抽出する。

```bash
# 案: cron_registry.txtから対象を動的抽出
mapfile -t GMO_BOT_LOGS < <(
  grep -E '^\s*[0-9*/,-]+ ' "$REGISTRY" \
    | grep -oE 'projects/gmo/[^ ]+\.sh' \
    | while read sh; do
        log="${sh%.sh}.log"
        echo "/home/multi-agent-shogun/${log}"
      done
)
```

**B. `cron_registry.txt`をsource of truthとして強制**

全cronはレジストリ経由で管理。レジストリにないエントリをcrontabに直接書くことを禁止。

**C. dashboardにBotのheatbeatウィジェットを追加**

「18/18稼働（直近2h以内に最終発火）」のような指標を冒頭に表示。18本中17本以下になれば自動的に🚨要対応として通知。

### 中長期対応

**bot_registry.yaml統合計画（A-2）**

crontabの自動生成・ログパス・ステータス・BTの合否をすべて1つのYAMLで管理する設計です。段階的に整備を進めています。

```yaml
# bot_registry.yaml の設計（概要）
bots:
  - id: eth_donchian48
    name: "#2 ETH Donchian 48h"
    script: projects/gmo/strategies/cron_gmo_donchian.sh
    cron: "10 * * * *"
    log: projects/gmo/strategies/cron_donchian.log
    symbol: ETH_JPY
    execution_type: single
    expected_interval_min: 60
    stale_h: 2
    status: active
```

`expected_interval_min`と`stale_h`を各Botに設定し、「毎時実行Botが2時間更新されていない」ようなケースを自動検知できる設計です。

bot_registry.yamlが完全稼働すれば、今回のような「crontabからエントリが消えたのに気づかない」問題は構造的に解消されます。ただし現時点では計画段階であり、運用はcheck_cron_health.sh改善とcron_registry.txt強制化が先行しています。

---

## 学んだこと

### 1. 「OK」を出すだけのヘルスチェックは盲点を生む

カバレッジを明示しないヘルスチェックは、実質的に誤った安全宣言になります。「3/3 healthy」と「18/18 healthy」は意味がまったく違います。

### 2. Botを増やすときは監視も一緒に増やす

4月のBot量産時に、ヘルスチェックスクリプトの監視リスト拡張を忘れました。「Botを追加するときは監視設定も必ず同時更新する」というルールをCONTRIBUTING.mdに明記しました。

### 3. サイレント失敗は最悪の失敗

エラーで止まるBotはすぐ気づけます。しかし「cronからは起動されないが、スクリプト自体は壊れていない」状態は発見が遅れます。6.5日間、誰も気づけませんでした。「無音＝正常」という暗黙の前提を疑う設計が重要です。

### 4. 発覚は人間の「違和感」だった

ダッシュボードを見て何かおかしいと感じ、ログのmtimeを直接確認したことで発見できました。エージェントも監視スクリプトも「All healthy」を返し続けていた中で、人間の直感が最後の防衛線でした。自動検知にレジリエンスを依存しすぎている状態は技術負債です。

---

## おわりに

「18本のBotを動かしています」と書けば聞こえはいいですが、実態は「11本が6.5日間サイレント停止していたことを誰も知らなかった」状態でした。

自動化を積み重ねると、ある時点で「システムの健全性を管理するシステム」が必要になります。今回のインシデントは、その設計を後回しにしていたツケが来た出来事です。

check_cron_health.shの改善とcron_registry.txt強制化はすでに着手しました。bot_registry.yamlによる統合管理は段階的に整備を進めています。

次回は、アルファマイニングパイプラインの設計——Bot候補を自動でバックテストして合格基準を満たしたものだけを本番に昇格させる仕組みについて書く予定です。

フォローしていただけると次の記事もお届けできます。

---

:::message
本システムはおしおさん（yohey-w）が作成・公開しているOSSのマルチエージェントシステム（[multi-agent-shogun](https://github.com/yohey-w/multi-agent-shogun)）をforkして構築・運用しています。ペーパートレードで検証中であり、実際の投資を推奨するものではありません。
:::
