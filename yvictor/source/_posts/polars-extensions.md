---
title: Polars Rust Expression Extension 開發：串接 ta-lib 讓指標瞬間多核平行化運算
date: 2024-02-08 18:26:04
tags:
    - code
    - rust
    - python
    - polars
    - pyo3
    - ta-lib
    - polars-extension
---

這篇文算是記錄一下，去年 PyCon 分享完 Polars 後，有一個比較難的環節，當時就沒有特別提，加上那時候 Polars extension 的 api 還沒有完全 expose，去年底發現這部分已經很完整了，我就決定要來寫這個我想做很久的 extension，我看到的時候是 12 月 28 左右，本來想要在過年之前寫出來，結果整個元旦都在寫，之前寫完了先開始爽爽用，又快要到農曆過年了，想說就趕快來記錄一下。

用 Polars Expression Extension 有什麼好處呢？
- 可讀性一致性: 提供了一個統一的框架，讓開發者可以通過相同的介面進行擴展開發，從而保持代碼的一致性和易讀性。
- 最佳化: 利用 Polars 的優化框架，進行底層效能的最佳化。
- 平行化: 利用 Polars 的平行運算框架，將運算任務指派到多個 CPU 核上執行，大幅提升運算速度。
- rust原生效能: rust效能，釋放python gil及效能限制。

我這次 extension 要做的範例就直接用串接 talib 來分享。本來在做多檔商品的技術指標，在 pandas 用起來真的很不舒服，寫起來很髒，效能又不好，自從用過 polars 的 over 語法後，整個回不去。本來想要直接用 Polars 語法上重新實作幾個在用的指標，但我才實作第三個，就覺得想放棄了，公式算起來的跟 talib 的不一樣，所以直接串 c 的 talib 是我覺得最簡單快速的解法。

舉個簡單的例子如果串接完的 extension 有多好用

``` python
pl.col("close").ta.ema(5).over("symbol").alias("ema5")
```

`over("symbol")` 這邊就是說 ema 的計算要根據 symbol 商品代碼分開計算，所以每檔商品 ema 就分開獨立計算，但他們所有資料可以放在一張 dataframe 中不需要分開，並且因為 polars 的關係，所以商品的指標計算都是平行多核心在跑的。

