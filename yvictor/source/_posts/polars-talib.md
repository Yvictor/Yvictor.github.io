---
title: 擁抱未來：為什麼你應該從 Pandas 轉換到 Polars
date: 2024-06-06 14:43:39
tags:   
    - code
    - rust
    - python
    - polars
    - pyo3
    - ta-lib
    - polars-extension
---
在不斷演變的數據分析世界中，提供高效、快速和易用性的工具至關重要。雖然多年来 Pandas 一直是 Python 中數據操作的首選庫，但現在是考慮一個強大替代方案的時候了：Polars。Polars 提供了顯著的性能提升和更一致的語法，使其成為現代數據分析任務的更佳選擇。其最突出的特點之一是原生 Rust 擴展，與 Pandas 相比，提供了無與倫比的速度和效率。

### 讓我們直接從實例開始感受
#### 準備數據
我們將下載 NASDAQ 市值最高的 2000 家公司的股票數據，並將其保存為 Parquet 文件。
``` python
import yfinance as yf
import polars as pl

tickers = pl.scan_csv("nasdaq_screener.csv").sort("Market Cap", descending=True).filter(
    pl.col("Market Cap").is_not_null() & (pl.col("Market Cap") > 0)
).select(pl.col("Symbol").str.strip_chars(" "), pl.col("Market Cap")).collect()["Symbol"].to_list()

data = yf.download(tickers[:2000], start='2001-01-01', end='2024-12-31', group_by='ticker', threads=32).stack(level=0).reset_index()
df = pl.from_pandas(data).with_columns(
    pl.col("Date").cast(pl.Date),
)
df.write_parquet("nasdaq_market_cap2000.parquet")
```

下面我們用 Polars 來讀檔並把命名轉換成像 talib abstract的用法
``` python
p = pl.scan_parquet("nasdaq_market_cap2000.parquet").select(
    pl.col("Date"), pl.col("Ticker").alias("Symbol"),
    pl.selectors.float().name.to_lowercase()
)
```

#### 如何使用 polars_talib
使用 over 語法，可以快速的對每檔商品計算SMA，這個操作，包括讀取文件、轉換和計算，僅需 139 毫秒。

``` python
%%timeit
df = p.with_columns(
    plta.sma(timeperiod=5).over("Symbol").alias("sma5"),
).filter(
    pl.col("Symbol") == "NVDA"
).collect()

139 ms ± 5.76 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

使用 Pandas，僅讀取文件並將列名轉換為小寫就需要 1.21 秒。
``` python
%%timeit
df = pd.read_parquet("nasdaq_market_cap2000.parquet").set_index(["Ticker", "Date"]).rename(
    columns={c: c.lower() for c in ["Open", "High", "Low", "Close"]}
)

1.21 s ± 62.5 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

接著使用talib有兩種計算方法：transform 和 apply。transform 方法更快，因此我們將在可能的情況下使用 transform。對於不能使用 transform 的情況，我們將使用 apply。計算速度的差異可以在以下結果中看到。

``` python
%%timeit
df["sma5"] = df.groupby("Ticker")["close"].transform(lambda x: ta.SMA(x, timeperiod=5))

1.84 s ± 62.3 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

``` python
%%timeit
df["sma5"] = df.groupby("Ticker").apply(lambda x: ta.SMA(x, timeperiod=5)).droplevel(0)

3.15 s ± 56.3 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### 效能小結
	- Pandas 使用 transform：1.84 秒 + 1.21 秒 = 3.05 秒（慢 22 倍）
	- Pandas 使用 apply：3.15 秒 + 1.21 秒 = 4.36 秒（慢 31 倍）
	- Polars 使用 over 語法並經查詢計劃優化：0.139 秒

在這些操作中，包括讀取文件和執行分析，Polars 明顯比 Pandas 更快。

#### 更多不同類型的指標
接著讓我們探索不同的 talib 函數及其各種輸入和輸出，並比較它們在 Polars 和 Pandas 中的用法。有些函數具有多個輸出而不是單一系列，我們將展示 Polars 如何提供一致的語法來方便地使用這些函數。

``` python
%%timeit
df = p.with_columns(
    plta.sma(timeperiod=5).over("Symbol").alias("sma5"),
    plta.macd(fastperiod=10, slowperiod=20, signalperiod=5).over("Symbol").alias("macd"),
    plta.stoch(pl.col("high"), pl.col("low"), pl.col("close"), fastk_period=14, slowk_period=7, slowd_period=7).over("Symbol").alias("stoch"),
    plta.wclprice().over("Symbol").alias("wclprice"),
).with_columns(
    pl.col("macd").struct.field("macd"),
    pl.col("macd").struct.field("macdsignal"),
    pl.col("macd").struct.field("macdhist"),
    pl.col("stoch").struct.field("slowk"),
    pl.col("stoch").struct.field("slowd"),
).select(
    pl.exclude("stoch")
).filter(
    pl.col("Symbol") == "AAPL"
).collect()

135 ms ± 5.6 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```
這些操作，包括讀取文件、轉換和計算輸出，經過了Polars最佳化，只需 135 毫秒


``` python
%%timeit
df["sma5"] = df.groupby("Ticker")["close"].transform(lambda x: ta.SMA(x, timeperiod=5))
df["macd"] = df.groupby("Ticker")["close"].transform(lambda x: ta.MACD(x, fastperiod=10, slowperiod=20, signalperiod=5)[0])
df["macdsignal"] = df.groupby("Ticker")["close"].transform(lambda x: ta.MACD(x, fastperiod=10, slowperiod=20, signalperiod=5)[1])
df["macdhist"] = df.groupby("Ticker")["close"].transform(lambda x: ta.MACD(x, fastperiod=10, slowperiod=20, signalperiod=5)[2])
df["slowk"] = df.groupby("Ticker").apply(lambda x: ta.STOCH(x, fastk_period=14, slowk_period=7, slowd_period=7)).droplevel(0)["slowk"] 
df["slowd"] = df.groupby("Ticker").apply(lambda x: ta.STOCH(x, fastk_period=14, slowk_period=7, slowd_period=7)).droplevel(0)["slowd"]
df["wclprice"] = df.groupby("Ticker").apply(lambda x: ta.WCLPRICE(x)).droplevel(0)
df.loc["AAPL"]

19.2 s ± 367 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```
在 Pandas 中，處理來自 talib 函數的多個輸出跟單輸出需要更多步驟和不同的語法，這可能會導致不一致和混淆。
並且以上執行大約需要 19.2 秒，顯示了 Pandas 語法的低效和不一致。

Polars 和 Pandas 的性能比較顯示出顯著的速度差異。以下是詳細比較：

 - Pandas 使用 transform 和 apply：
 - 讀取文件：1.2 秒
 - 執行計算：19.2 秒
 - 總時間：1.2 秒 + 19.2 秒 = 20.4 秒
 - Polars 使用 over 語法並經查詢計劃優化：
 - 總時間：0.135 秒

因此，在這些操作中，包括讀取文件和執行計算，Polars 配上封裝好的 talib extension 比 Pandas + talib 大約快 150 倍。
通過比較這些方法，可以明顯看出 Polars 在這類計算中相比 Pandas 具有顯著的性能優勢。Polars 提供了一致且簡化的語法，減少了混淆，使代碼更易於維護。

#### Rust 原生擴展的威力
Polars 的一大亮點是其 Rust 原生擴展，這使得它與 Pandas 不同。這個擴展利用了 Rust 的高效內存管理和並發能力，為 Polars 帶來了無與倫比的速度和性能。相比之下，Pandas 依賴於 Python 的原生功能，這可能會更慢且資源消耗更大，而 Polars 的 Rust 基礎確保了操作的最佳效率。

#### Expression Plugin
Polars 引入了表達式插件，這是創建用戶自定義函數的首選方式。這些插件允許你編譯 Rust 函數並將其註冊為 Polars 庫中的表達式。這意味著你的自定義函數可以像原生表達式一樣快速運行，並具有以下幾個顯著優點：

 - 優化：Polars 引擎會優化這些表達式，確保它們盡可能高效。
 - 並行性：Polars 充分利用 Rust 的並發特性，允許表達式的並行執行，加快數據處理速度。
 - Rust 原生性能：通過避免 Python 的全局解釋器鎖（GIL），Polars 中的表達式插件無需受到 Python 的干擾，確保高性能和高效資源使用。

#### Polars 原生表達式擴展包裝 Ta-Lib polars_talib
polars_talib 正是我用 polars expression plugin 去封裝 talib 的 c library 來讓 polars 在 python 中可以輕鬆的當成 expression 來使用並且享受 rust c 的高效率。

#### 其他值得關注的 Polars Extension
- polars_ds_extension: 專門針對 Data Science 的 expression 非常方便

### 為什麼 Polars 優於 Pandas
- 性能：得益於其 Rust 基礎，Polars 比 Pandas 提供了顯著的速度提升，特別是在處理大型數據集時。
- 一致性：Polars 提供了更一致和用戶友好的語法，減少了學習曲線，使數據操作更加直觀。
- 可擴展性：Polars 設計高度可擴展，擁有如 polars_talib 和 polars_ds_extension 等強大的擴展，增加了顯著的功能和性能增強。

總之，儘管 Pandas 一直是 Python 中可靠的數據操作工具，但數據分析的未來屬於 Polars。其 Rust 原生擴展、創新的表達式插件功能，以及通過 polars_talib 與 Ta-Lib 的集成，以及其他強大的擴展如 polars_ds_extension，使得 Polars 成為任何希望迎接數據分析未來的人的最佳選擇。