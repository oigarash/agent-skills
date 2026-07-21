# 確定申告 計算根拠

## RSU (制限付き株式) の給与所得

### 計算式

```
課税所得(JPY) = FMV(USD) × 株数 × USD/JPY(vest日)
```

- **FMV**: vest日（権利確定日）の終値
- **USD/JPY**: vest日の市場終値（yfinance `USDJPY=X`）。正式には銀行TTM仲値を推奨
- **株数**: gross株数（税源泉前）を使用。E*TRADEが源泉分を会社に代わって処理するため、gross株数 × FMVが所得となる

### データソース

- vest日・株数: `BenefitHistory.xlsx` の `Restricted Stock` シート
- 株価: yfinance `CSCO` の終値（休日は翌営業日を使用）
- 為替: yfinance `USDJPY=X` の終値

### 申告区分

**給与所得** として申告。源泉徴収票に含まれていないため、確定申告書等作成コーナーで「その他給与」として追加入力。

---

## ESPP (従業員株式購入計画) の給与所得

### 計算式

```
課税所得(USD) = (FMV at Purchase - Purchase Price) × 株数
課税所得(JPY) = 課税所得(USD) × USD/JPY(購入日)
```

- **FMV at Purchase**: 購入日の公正市場価値（E*TRADEが明示）
- **Purchase Price**: look-back規定適用後の実際購入価格
- **USD/JPY**: 購入日の為替レート

### Cisco ESPP Look-back規定（重要）

Ciscoのオファリング期間は24ヶ月（例: 2022-07-01 開始）。

```
Purchase Price = 85% × min(Offering Date FMV, Purchase Date FMV)
```

オファリング開始時のFMV < 購入日FMV の場合:
- Purchase Price = 85% × Offering Date FMV
- 例: $47.52 × 85% = $40.392

これはE*TRADEのBenefitHistoryの `Purchase Price` 欄に既に反映されているため、**E*TRADEの値をそのまま使用**すること。

### 申告区分

**給与所得** として申告（RSUと同様）。

---

## 配当所得 (外国株式配当)

### 計算式

```
配当所得(JPY) = 配当合計(USD) × USD/JPY(受取日 or 年末)
外国税額控除 = US源泉税(USD) × USD/JPY
```

### 米日租税条約の適用

- US源泉税率: **10%**（日米租税条約第10条）
- 通常の米国内税率30%から減免される
- E*TRADEが自動的に10%を源泉徴収

### 四半期配当のUSD/JPY

より正確な計算には、各支払日の為替レートを使用:

| CSCO支払月 (概算) | 適用USD/JPY |
|------------------|------------|
| 1月下旬 (Q1) | 受取日の終値 |
| 4月下旬 (Q2) | 受取日の終値 |
| 7月下旬 (Q3) | 受取日の終値 |
| 10月下旬 (Q4) | 受取日の終値 |

簡便法として年末(12/31)の為替レートを一括使用することも可。差異は通常数千円以内。

### Treasury Liquidity Fund (TLF) 配当

マネーマーケットファンドの配当。`Other Dividends` として計上される。
US源泉税はCSCO配当のみに課せられている（TLFは源泉徴収なし）。

### 申告区分

**配当所得** として申告（申告分離課税 or 総合課税を選択）。外国税額控除も同時に申告。

---

## 為替レートの取得方法

```python
import yfinance as yf
from datetime import datetime, timedelta

def get_close_price(ticker: str, date_str: str):
    """指定日の終値取得。休日は翌営業日にフォールバック。"""
    dt = datetime.strptime(date_str, "%Y-%m-%d")
    for offset in range(5):
        check = dt + timedelta(days=offset)
        end = check + timedelta(days=2)
        df = yf.download(ticker, start=check.strftime("%Y-%m-%d"),
                         end=end.strftime("%Y-%m-%d"), progress=False, auto_adjust=True)
        if not df.empty:
            return float(df["Close"].iloc[0]), df.index[0].strftime("%Y-%m-%d")
    raise ValueError(f"No data for {ticker} near {date_str}")

# 使用例
csco_price, actual_date = get_close_price("CSCO", "2025-08-10")
usdjpy, fx_date = get_close_price("USDJPY=X", "2025-08-10")
```

> **正式な為替レート**: 税務上は銀行の電信売買相場の仲値(TTM)を使用するのが原則。
> yfinanceの終値は参考値として実務上許容範囲。大きな差異が出る場合は三菱UFJ等の公示レートを確認。

---

## 確定申告書等作成コーナー 入力手順

### 1. 給与所得 (RSU + ESPP 合計)

源泉徴収票入力後:
- 「他の所得がある方へ」→「給与・退職所得以外の所得」は不要
- RSU/ESPPは **給与所得** なので源泉徴収票の「支払金額」に加算して入力
- または会社に確認して修正源泉徴収票を発行してもらう方法もある

> **注意**: RSU/ESPPの申告方法については税理士に確認することを推奨。

### 2. 配当所得

「配当所得・株式等の譲渡所得等の申告」→「外国株式等の配当」:
- 配当の支払者: 証券会社名 (E*TRADE / Morgan Stanley)
- 配当金額: ¥XXX,XXX (JPY換算後)
- 外国税額: ¥XX,XXX (US源泉税のJPY換算)
- 国名: アメリカ合衆国

### 3. 外国税額控除

「外国税額控除等」→「外国税額控除」:
- 外国税額(USD): $103.53（例）
- 適用条約: 日米租税条約

---

## 出力例 (参照用)

スクリプト実行後、以下のような形式で結果が出力される:

| 項目 | 値 |
|------|-----|
| RSU合計課税所得 | ¥X,XXX,XXX |
| ESPP合計課税所得 | ¥X,XXX,XXX |
| 給与所得加算合計 | ¥X,XXX,XXX |
| CSCO配当(USD) | $X,XXX.XX |
| TLF配当(USD) | $XX.XX |
| 配当合計(JPY) | ¥XXX,XXX |
| US源泉税(USD) | $XXX.XX |
| 外国税額控除(JPY) | ¥XX,XXX |
