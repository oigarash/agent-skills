# E*TRADE 操作手順 (playwright-cli)

## セッション起動

```bash
# 国税庁サイト用のheadedブラウザを先に起動（まだの場合）
playwright-cli open --headed --browser=chrome https://www.keisan.nta.go.jp/

# E*TRADEを追加タブで開く
playwright-cli tab-new https://us.etrade.com
```

> **重要**: `--headed` は国税庁サイト用の要件。E*TRADE自体はheadlessでも動作するが、同一セッションで操作するため同じブラウザを使用。

ログインはブラウザで手動実行（セキュリティのためパスワードはPlaywrightに渡さない）。

---

## BenefitHistory.xlsx ダウンロード手順

RSU/ESPP の全履歴データを含むExcelファイル。

> **重要**: 旧URL (`/etx/aw/benefithistory`) は404になる。以下のナビゲーション手順で辿ること。

```bash
# At Work → My Account → Benefit History タブ
# Step 1: At Workページに移動
playwright-cli click "At Work"  # ナビゲーションバーから

# Step 2: My Accountをクリック
playwright-cli click "My Account"

# Step 3: Benefit Historyタブをクリック
# URL: https://us.etrade.com/etx/sp/stockplan#/myAccount/benefitHistory
playwright-cli snapshot  # タブ要素のref確認
playwright-cli click <Benefit History tab ref>  # snapshotで確認したref番号を使用

# Step 4: Download Expanded をクリック
playwright-cli snapshot  # Downloadボタンのref確認
playwright-cli click <Download button ref>  # ドロップダウンが開く
playwright-cli click <Download Expanded menuitem ref>  # メニューアイテムをクリック
# ファイルは .playwright-cli/BenefitHistory.xlsx として保存される
```

直接URLでのアクセスも可能（ただしAt Workセッションが有効な場合のみ）:
```bash
playwright-cli goto "https://us.etrade.com/etx/sp/stockplan#/myAccount/benefitHistory"
```

**ダウンロードファイル**: `BenefitHistory.xlsx`
**主要シート**:
- `Restricted Stock` → RSU vesting履歴 (Date, Transaction Type, Shares, Price)
- `ESPP` → ESPP購入履歴 (Purchase Date, Shares Purchased, Purchase Price, FMV)
- `OSPS` → 追加の株式プラン履歴

**RSU データ抽出ポイント**:
- `Record Type` 列で `Grant` 行と `Event` 行がある
- `Event` 行の `Event Type` が `"Shares vested"` の行を使用
- `Date` = vest日 (MM/DD/YYYY形式)
- `Qty. or Amount` = 株数（gross。税源泉分も含む）
- **Grant紐付け**: Event行は直前のGrant行に属する。Grant行の `Grant Date` と `Grant Number` でグラントを識別

**ESPP データ抽出ポイント**:
- `Record Type = "Purchase"` の行を使用
- `Purchase Date` = 購入日 (DD-MMM-YYYY形式, 例: 30-JUN-2025)
- `Purchased Qty.` = 購入株数
- `Purchase Price` = 実際の購入価格（look-back適用後）
- `Purchase Date FMV` = 購入日の公正市場価値 (先頭に`$`がつく場合あり、要strip)

---

## 年次ステートメントPDF ダウンロード手順

配当所得・US源泉税額の年間合計を取得。

```bash
# Documents → Statements タブ (デフォルトで選択されている)
playwright-cli goto "https://us.etrade.com/etx/pxy/accountdocs"
# ページロードに数秒かかる場合がある。Loadingが消えるまで待つ

playwright-cli snapshot
# Statements タブ内のテーブルから12月分 (12/31/YY) の
# "Single Account Statement" リンクをクリック
# ファイル名例: ClientStatements-2784-123125.pdf (.playwright-cli/ に保存される)
```

> **注意**: 旧URL (`/etx/hw/docs/statements`) は動作しない場合がある。`/etx/pxy/accountdocs` を使用すること。

**ステートメントから読み取る値** (Page "Account Summary" の Income and Distribution Summary欄):

| 項目 | 欄名 | 説明 |
|------|------|------|
| 配当所得 | Qualified Dividends (This Year) | CSCO株の適格配当 |
| その他配当 | Other Dividends (This Year) | マネーマーケット等 |
| US源泉税 | Tax Withholdings (This Year) | 外国税額控除の根拠 |

12月ステートメントの `This Year` 列が全年合計値。

---

## Activity (トランザクション) ページ

配当の四半期別内訳が必要な場合に使用。

```bash
playwright-cli goto "https://us.etrade.com/etx/pxy/accounts/transactions"
playwright-cli screenshot
```

> **注意**: このページは "We're experiencing a technical issue" エラーが発生することがある。
> その場合はアカウントホームから再アクセスするか、月次ステートメントPDFを使用する。

フィルター操作:
```bash
playwright-cli snapshot  # フィルター要素のref確認
# Date Rangeを設定: 01/01/YYYY ～ 12/31/YYYY
# Transaction Type: "Dividends" を選択
```

---

## Stock Plan Holdings 確認

現在の保有状況・株価確認。

```bash
playwright-cli goto "https://us.etrade.com/e/t/eproxyj2ee/ajax/etws"
# または
playwright-cli goto "https://us.etrade.com/etx/aw/stockplansummary"
playwright-cli screenshot
```

---

## セッション管理

```bash
# タブ一覧確認
playwright-cli tab-list

# NTA (tab 0) と E*TRADE (tab 1) を切り替え
playwright-cli tab-select 0  # 国税庁
playwright-cli tab-select 1  # E*TRADE

# セッション状態保存（ログイン情報を保持）
playwright-cli state-save etrade-session.json
playwright-cli state-load etrade-session.json
```

---

## PDFからのデータ抽出 (Python)

```python
import pdfplumber

with pdfplumber.open("ClientStatements-XXXX-12MMYY.pdf") as pdf:
    for i, page in enumerate(pdf.pages):
        text = page.extract_text()
        if text and ("Qualified Dividends" in text or "Tax Withholdings" in text):
            print(f"--- Page {i+1} ---")
            print(text)
```

## BenefitHistoryの読み取り (Python)

```python
import pandas as pd

xl = pd.ExcelFile("BenefitHistory.xlsx")
print(xl.sheet_names)  # シート一覧

# RSU履歴
df_rsu = pd.read_excel("BenefitHistory.xlsx", sheet_name="Restricted Stock")
# vesting行のみ抽出
vested = df_rsu[df_rsu["Transaction Type"].str.contains("Vest|Release", case=False, na=False)]
print(vested[["Date", "Shares", "Price"]])

# ESPP履歴
df_espp = pd.read_excel("BenefitHistory.xlsx", sheet_name="ESPP")
purchases = df_espp[df_espp["Transaction Type"].str.contains("Purchase", case=False, na=False)]
print(purchases[["Purchase Date", "Shares Purchased", "Purchase Price", "FMV at Purchase"]])
```
