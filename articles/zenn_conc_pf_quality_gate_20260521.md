---
title: "過学習Botを見抜く品質ゲート：conc_PF（利益集中度PF）の設計と実践"
emoji: "📊"
type: "tech"
topics: ["claudecode", "python", "自動売買", "バックテスト"]
published: true
---

## はじめに：高PFでも信用できないバックテスト

自動売買Botを開発していると、バックテストで高いPF（プロフィットファクター）を記録したにもかかわらず、本番環境で全く機能しない戦略に出くわすことがあります。

原因の一つが「利益集中」です。100回のトレードで合計PF=1.5を達成していても、そのうち5〜10回の大当たりトレードが利益のほぼ全てを担っている場合、残りの90回は実はほぼ損益ゼロか小損失というケースが起きます。こうした戦略は、たまたま相場の大きな動きをキャッチしたときだけ機能する「過学習Bot」である可能性が高いです。

この記事では、こうした過学習Botを見抜くために実装した「conc_PF（利益集中度PF）」という品質ゲートの設計思想・計算方法・実際の効果をコードベースで解説します。

自動売買システムの開発・QA・運用を行う中で生まれた実践的な知見です。

---

## conc_PFとは何か

**conc_PF（利益集中度PF）** は、「上位10%の高利益トレードを除外した後のプロフィットファクター」です。

通常のPF計算では、トレード全体の総利益÷総損失を計算します。これに対してconc_PFは、

1. 全トレードをPnL（損益）の大きい順にソート
2. 上位10%（少なくとも1件）のトレードを除外
3. 残り90%のトレードでPFを再計算

という手順で算出します。

もし上位10%を除外してもPF≥1.0を維持できるなら、その戦略は「大当たりに頼らない安定した利益構造」を持っていると判断できます。逆にconc_PFが1.0を下回る場合、利益の大部分が一部のトレードに集中しており、汎化性能への懸念が生じます。

---

## 実装コード：gmo_backtest.py より

実際の実装を見てみましょう。バックテストエンジン（`gmo_backtest.py`）の `calculate_metrics()` 関数内に以下のコードが組み込まれています。

```python
# 利益集中度チェック: top10%取引除外後PF
sorted_trades = sorted(trade_log, key=lambda t: t["pnl"], reverse=True)
top_n = max(1, len(sorted_trades) // 10)
trimmed = sorted_trades[top_n:]
trimmed_wins   = sum(t["pnl"] for t in trimmed if t["pnl"] > 0)
trimmed_losses = abs(sum(t["pnl"] for t in trimmed if t["pnl"] <= 0))
if trimmed_losses == 0:
    profit_concentration_pf = float("inf") if trimmed_wins > 0 else 0.0
else:
    profit_concentration_pf = trimmed_wins / trimmed_losses
```

ポイントを整理します。

**`top_n = max(1, len(sorted_trades) // 10)`**
取引数が少ない場合でも必ず1件は除外します。10件のトレードなら1件、100件なら10件を除外します。

**`trimmed_losses == 0` の場合は `inf`**
除外後のトレードで損失が一切ない場合、`inf`（無限大）を返します。これは「全損失ゼロ」を意味し、合格とみなします。

**返り値は辞書に格納**

```python
return {
    # ... 他の指標 ...
    "profit_concentration_pf": round(profit_concentration_pf, 4),
    "passed": passed,
}
```

BTエンジンのメトリクス辞書に含まれ、後続の合格判定で参照されます。

---

## 合格基準：conc_PF ≥ 1.0

バックテストランナー（`auto_bt_runner_cfd.py`）では、複数の品質条件を組み合わせて戦略の合格を判定しています。conc_PFの合格条件は以下の通りです。

```python
sym_passed = (
    is_pf >= PASS_PF and                    # IS PF ≥ 1.20
    oos_pf >= PASS_PF_OOS and               # OOS PF ≥ 0.90
    oos_ratio >= PASS_OOS_RATIO and          # OOS/IS比 ≥ 0.70
    total_trades >= PASS_TRADES and          # 総取引数 ≥ 50
    max_dd_combined <= sym_max_dd and        # 最大DD ≤ 25%
    (min_calmar == float("inf") or min_calmar >= PASS_CALMAR) and  # Calmar ≥ 1.0
    oos_trades >= oos_min and
    (is_conc_pf == float("inf") or is_conc_pf >= 1.0) and  # conc_PF条件
    is_ir >= PASS_IR and                     # IS_IR ≥ 1.0
    is_first_half_pf >= 1.0                  # IS前半PF ≥ 1.0
)
```

注目すべきは、conc_PFが `inf` の場合は無条件で合格扱いとなっていることです。除外後に損失がゼロということは、残りのトレードが全て勝ちか引き分けであり、これは高品質の証拠です。

また、**救済パス（rescue path）にも同じ条件が適用されています**。

```python
if not sym_passed and is_pf >= 1.0 and is_calmar >= 0.5:
    if (oos_trades >= 15 and total_trades >= 80 and oos_pf >= 1.5
            and max_dd_combined <= sym_max_dd
            and (is_conc_pf == float("inf") or is_conc_pf >= 1.0)  # 救済パスも必須
            and is_ir >= PASS_IR
            and is_first_half_pf >= 1.0):
        sym_passed = True
        conditional_pass = True
```

救済パスはIS PFがメイン基準に届かない場合に適用される緩和条件ですが、conc_PFの条件はここでも免除されません。利益集中度は戦略品質の根本的な指標であるため、いかなる条件緩和にも例外を設けていません。

---

## IS Information Ratio との組み合わせ

conc_PFと並んで実装されているもう一つの過学習防止指標が **IS Information Ratio（IS_IR）** です。

```python
# IS Information Ratio = IC × √BR
# IC = (PF-1)/(PF+1)、BR = IS取引数
if is_pf == float("inf"):
    is_ic = 1.0
elif is_pf <= 0:
    is_ic = -1.0
else:
    is_ic = (is_pf - 1) / (is_pf + 1)
is_ir = is_ic * math.sqrt(is_trades) if is_trades > 0 else 0.0
```

IS_IRは「シグナルのエッジ強度（IC）× 取引回数の平方根（√BR）」で計算されます。取引回数が少ない場合はペナルティが大きく、取引回数が多い高PF戦略ほど高いIS_IRを示します。

合格基準は `PASS_IR = 1.0` です。既存の稼働Botを全件チェックした結果、最小IS_IRは1.626であり、全件がこの基準をクリアしています。

conc_PFが「利益の分散（集中でないこと）」を検証するのに対し、IS_IRは「PFの統計的信頼性（取引数の十分さ）」を検証します。両者は異なる角度から過学習を検出するため、組み合わせることで単独では捕捉できないリスクをカバーできます。

---

## 実際の効果：本番運用での実績

conc_PF品質ゲートの導入後、以下のような結果が得られています。

### 停止に至った戦略の例

OANDAのCFD（差金決済）Botを対象にconc_PFをチェックした結果：

| EA名 | 銘柄 | IS PF | conc_PF | 判定 |
|------|------|-------|---------|------|
| CFD#1 Donchian NATGAS v1 | NATGAS | 1.5881 | 0.6785 | ❌ FAIL |
| CFD#2 Donchian NATGAS Both | NATGAS | 1.2810 | 0.4091 | ❌ FAIL |
| CFD#3 VB NATGAS Long | NATGAS | 1.4078 | 0.4110 | ❌ FAIL |

上記3本はIS PFが1.2〜1.6と一見良好に見えますが、conc_PFが0.4〜0.68と著しく低いです。これは、利益の大部分が少数の大当たりトレードに集中していることを意味します。この結果を受け、3本全てを本番から停止しました。

### 継続稼働中の戦略の例

一方、`CFD#6 Donchian XAUUSD`（4時間足、long_only）はconc_PF=1.013でPASSし、現在もLIVE稼働中です。conc_PFが1.0をわずかに超える水準であっても、継続的に稼働に値すると判断する根拠になっています。

### NATGAS戦略とXAUUSD戦略の違いが示すもの

NATGASのlong_onlyにおけるconc_PF達成率は約20%、XAUUSDは約50%という調査結果も得られています。XAUUSDは長期的な上昇トレンドが安定しており、トレンドフォロー戦略の利益が特定の大相場に集中しにくい構造的な理由が考えられます。

---

## まとめ：品質ゲートを重ねる意味

conc_PFの設計思想は「バックテストのPFは嘘をつく可能性がある、ならば嘘がつきにくい別角度の指標で補完する」というものです。

- **通常PF** → 全トレードの粗利益率
- **conc_PF** → 上位10%を除いた後の真の粗利益率
- **IS_IR** → PFの統計的有意性（取引数考慮）
- **OOS PF** → 過学習の直接検証
- **IS前半PF** → 特定相場への過依存の排除

これらを組み合わせることで、「たまたまうまくいったバックテスト」が本番稼働に紛れ込むリスクを大幅に低減できます。

BT実行・品質チェック・稼働判定を自動化することで、人的なバイアスを排除しつつ厳格な品質基準を維持する運用が実現できています。

品質ゲートを増やすことは「通過する戦略が減る」ことを意味しますが、それは同時に「本番稼働に値する確度の高い戦略だけが残る」ことを意味します。過学習Botの排除は、長期的な収益安定性のために不可欠な投資です。

---

## 関連コード・記事

- `projects/gmo/gmo_backtest.py`：conc_PF計算（`calculate_metrics()`）
- `projects/cfd/auto_bt_runner_cfd.py`：品質ゲート全体（`_evaluate_single_direction()`）
- `projects/cfd/rebt_cfd_ea_conc_check.py`：稼働Bot全件conc_PFチェックスクリプト
