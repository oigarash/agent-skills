---
name: etrade-kakutei-sinkoku
description: E*TRADE (Morgan Stanley at Work) の証券口座から日本の確定申告に必要なデータを収集・計算・レポート出力するスキル。Cisco社員向けに、RSU（制限付き株式）のvesting所得、ESPP（従業員株式購入計画）の購入割引所得、CSCO株式配当の外国源泉税額控除を計算する。playwright-cli を使ったブラウザ操作、yfinance による株価・為替データ取得、Excel/HTML/Markdown出力までの一連の手順を提供する。対象年度のデータを収集して確定申告書等作成コーナーへの入力値を算出したいとき、またはこのワークフローを実行・更新したいときに使用。
---

# E*TRADE 確定申告データ収集・計算スキル

## ワークフロー概要

```
Step 1: E*TRADEからデータ収集 (playwright-cli)
  ├─ BenefitHistory.xlsx ダウンロード (RSU/ESPP履歴)
  └─ 年次ステートメントPDF ダウンロード (配当・源泉税)

Step 2: 計算スクリプト実行
  └─ scripts/fetch_tax_data.py (yfinance で株価・為替取得 → Excel/HTML/Markdown出力)

Step 3: 確定申告書等作成コーナーへ入力
  └─ references/nta_entry_guide.md を参照
```

## Step 1: E*TRADEデータ収集

### playwright-cli 起動 (必須: --headed フラグ)

```bash
# E*TRADEを新しいタブで開く (既存セッションがあれば既存ブラウザに追加)
playwright-cli tab-new https://us.etrade.com
```

> **注意**: 国税庁サイト用に `playwright-cli open --headed --browser=chrome` で起動済みのセッションに追加タブとして開く。

詳細な操作手順は `references/etrade_navigation.md` を参照。

### 収集するデータ

| データ | 取得場所 | ファイル形式 |
|--------|----------|------------|
| RSU/ESPP 履歴 | At Work → My Account → Benefit History タブ → Download Expanded | BenefitHistory.xlsx |
| 年次ステートメント | Documents → Statements → 対象年12月分 | ClientStatements-XXXX-12MMYY.pdf |

## Step 2: 計算スクリプト実行

### 事前準備: データ入力

`scripts/fetch_tax_data.py` の先頭にある定数セクションを、Step 1で収集したデータで更新する:

```python
# RSU_EVENTS: vest日・株数をBenefitHistory.xlsxから転記
RSU_EVENTS = [
    {"date": "YYYY-MM-DD", "shares": N, "grant_ref": "Grant名"},
    ...
]

# ESPP_EVENTS: 購入日・株数・購入価格・FMVをBenefitHistory.xlsxから転記
ESPP_EVENTS = [
    {"date": "YYYY-MM-DD", "shares": N, "purchase_price": X.XXX, "fmv": YY.YY},
    ...
]

# DIVIDEND: ステートメントPDFのIncome Summary欄から転記
DIVIDEND = {
    "qualified_usd": XXXX.XX,   # Qualified Dividends
    "other_usd":     XX.XX,     # Other Dividends (money market等)
    "us_tax_withheld_usd": XXX.XX,  # Tax Withholdings (全年合計)
}
```

計算ロジックの詳細は `references/tax_calculation.md` を参照。

### 実行

```bash
cd /path/to/working/directory
python3 scripts/fetch_tax_data.py
# → tax_summary_YYYY.xlsx / .html / .md が生成される
```

### 出力ファイル

3形式で同一内容を出力:

| ファイル | 用途 |
|---------|------|
| `tax_summary_YYYY.xlsx` | Excel (色分け付き5シート: 申告サマリー / RSU計算根拠 / ESPP計算根拠 / 配当計算根拠 / 色の凡例) |
| `tax_summary_YYYY.html` | HTML (印刷対応、色分け凡例・NTA入力手順付き) |
| `tax_summary_YYYY.md`   | Markdown (GitやNotesでの共有・記録用) |

## Step 3: 確定申告書等作成コーナー入力値

スクリプト出力のサマリー例:

```
【給与所得の申告 (RSU + ESPP)】
  RSU  課税所得合計: ¥X,XXX,XXX  → 源泉徴収票の給与所得に加算
  ESPP 課税所得合計: ¥X,XXX,XXX
  給与所得加算額合計: ¥X,XXX,XXX

【配当所得の申告 (外国株式)】
  配当収入(円換算): ¥XXX,XXX
  外国税額控除額:   ¥XX,XXX  (米国源泉税10%)
```

入力フロー:
1. **給与所得**: 源泉徴収票入力後に「その他給与」として上記合計を追加
2. **配当所得**: 外国株式配当として配当収入を入力
3. **外国税額控除**: 「外国税額の控除」でUS源泉税額を入力

## 確定申告の実施（国税庁サイトへの入力）

このスキルはE*TRADEからのデータ収集・計算までをカバーする。実際に確定申告書等作成コーナーで申告書を作成・提出する際は、以下のスキルを利用すること:

- **確定申告書作成スキル**: https://zenn.dev/kazukinagata/articles/83fe82191db01b

## 参照ファイル

- `references/etrade_navigation.md` - E*TRADE画面の操作手順詳細
- `references/tax_calculation.md` - RSU/ESPP/配当の計算根拠と注意事項
