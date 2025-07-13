---
title: rsolace 的進化：一次關於 Rust FFI、異步與記憶體安全的深度實踐
date: 2025-07-13 20:34:53
tags:
- Rust
- rsolace
- Solace
- CI/CD
- FFI
- Pin
- async/await
- tokio
- channel
- DashMap
---

開發一個高品質的 Rust 函式庫，尤其是當它需要與現有的 C 函式庫進行深度互動（FFI）時，往往是一條充滿挑戰的道路。在過去一週，我為 `rsolace`（一個 Solace PubSub+ 的 Rust 綁定）進行了一系列重大更新。本文將記錄這段歷程，不僅分享如何克服跨平台 CI/CD 的種種困難，更將深入探討我們如何運用 Rust 的核心特性（如 `Pin`）來解決棘手的記憶體安全問題，並引入異步（`async/await`）支援，使其架構更加現代化與穩固。

### 架構革新：用 `Pin` 深入解決 FFI 中的記憶體安全難題

在與 C 函式庫進行 FFI 互動時，最棘手的問題之一就是記憶體安全，尤其是在處理回呼（callback）時。在 `rsolace` 中，我們需要將一個指向 Rust 物件的指標（`user_p`）傳遞給 Solace 的 C API。C 函式庫會儲存這個指標，並在稍後觸發事件時，透過這個指標回呼到我們的 Rust 程式碼中。

**問題的核心：Rust 的所有權與 Move Semantics**

這個模式的危險之處在於 Rust 的所有權與移動語意（Move Semantics）。在 Rust 中，除非一個型別有實作 `Copy` trait，否則它的值在進行以下操作時，所有權會被轉移，其記憶體內容**會被移動**（`memcpy`）到新的位置：

1.  **變數賦值**：`let client_b = client_a;`
2.  **函式傳參**：`process_client(client_a);`
3.  **函式回傳**：`return client_a;`

**一個具體的危險情境**

讓我們想像一下 `rsolace` 如果沒有 `Pin` 會發生什麼事：

1.  我們建立了一個 `SolClient` 物件，它位於記憶體位址 `0x1000`。
2.  我們呼叫 `connect()`，將 `SolClient` 的位址 `0x1000` 作為 `user_p` 傳遞給 Solace 的 C 函式庫。C 函式庫高興地記下了這個位址。
3.  接下來，我們的應用程式邏輯可能需要將這個 `SolClient` 物件放進一個 `Vec` 中來管理多個連線：`let mut clients = Vec::new(); clients.push(sol_client);`
4.  `Vec::push` 這個操作會取得 `sol_client` 的所有權。如果 `sol_client` 原本是個堆疊上的變數，`push` 會將其內容**整個移動**到由 `Vec` 管理的、位於堆上的新記憶體位址。**這裡需要強調的是，移動的是 `SolClient` 這個 struct 物件本身。** 我們傳遞給 C 函式庫的，是指向這個 struct 物件的指標，因此這個指標就失效了。
5.  **災難發生**：Solace C 函式庫對這一切毫不知情。它仍然持有指向舊位址 `0x1000` 的指標。當它試圖透過這個懸空指標（dangling pointer）回呼我們的 Rust 程式碼時，就會讀取到無效或已釋放的記憶體，導致程式崩潰或難以追蹤的未定義行為。

**解決方案：`Pin` 的保證**

為了防止這種災難性的記憶體移動，我引入了 `Pin`。`Pin` 是 Rust 用來處理「自我參考結構」（self-referential structs）和固定記憶體位置需求的核心工具。透過 `Pin`，我們可以向編譯器做出一個承諾：**這個物件的記憶體位置將不會被改變**。

我的具體做法是將 `SolClient` 的核心邏輯和資料抽取到一個 `SolClientInner` 結構中，然後將 `SolClient` 的定義改為：

```rust
struct SolClient {
    inner: Pin<Box<SolClientInner>>,
}
```

這裡的 `Pin<Box<T>>` 是關鍵。`Box<T>` 會將 `SolClientInner` 配置在堆（heap）上，這本身就提供了一層間接性。而 `Pin` 則提供了更強的保證：它確保了 `Box` 所指向的記憶體**永遠不會被移動或替換**。

現在，當 `SolClient` 這個外層的 struct 物件被移動時（例如被 `push` 到 `Vec` 中），**改變的只有 `SolClient` 這個 struct 物件本身的位址**。它內部只包含一個指向 `Pin<Box<SolClientInner>>` 的指標（`inner`），這個指標所指向的 `SolClientInner` 資料，其在堆上的記憶體位址是**穩定不變**的。

因此，我們可以安全地將指向 `SolClientInner` 的指標傳遞給 C 函式庫，因為我們已經向編譯器保證了它不會再被移動。

### 引入異步支援：`async/await` 與 `tokio` 的整合

現代的網路應用程式，異步（asynchronous）操作是不可或缺的一環。為了讓 `rsolace` 能夠更好地融入 `async/await` 的生態系，我為其新增了基於 `tokio` 的異步支援。

原本的 `send_request` 方法是同步的，會阻塞當前的執行緒直到收到回覆。在這次的更新中，我引入了 `send_request_async` 方法，它會回傳一個 `Future`，讓使用者可以用 `async/await` 的方式來處理請求和回覆。

這個功能的實作，主要依賴 `kanal` 這個 `channel` 函式庫。當使用者呼叫 `send_request_async` 時，我們會：

1.  產生一個 `kanal` 的 `async` channel。
2.  將 `sender` 端存入一個 `DashMap` 中，以 `correlation_id` 作為 key。
3.  非同步地發送請求。
4.  在 C 函式庫的回呼函式中，當收到回覆時，我們會根據 `correlation_id` 從 `DashMap` 中找到對應的 `sender`，並將回覆的訊息傳送出去。

如此一來，使用者就可以用非常直觀的方式來撰寫異步的程式碼：

```rust
let response = client.send_request_async(&request_msg).await?;
```

這個改動讓 `rsolace` 的 API 更加現代化，也更容易與其他基於 `tokio` 的函式庫整合。

### 提升執行緒安全

除了記憶體安全，執行緒安全（thread safety）也是 `rsolace` 的一個重要考量。在這次的更新中，我做了以下幾點改進：

*   **`&self` 取代 `&mut self`**：我將 `disconnect()` 和 `send_request_async()` 等方法的接收者從 `&mut self` 改為 `&self`。這意味著這些方法現在可以被多個執行緒同時呼叫，提升了 `rsolace` 的並行性。
*   **`SolMsg` 的執行緒安全測試**：我新增了一個執行緒安全測試，確保 `SolMsg` 可以在不同的執行緒之間安全地傳遞。在這個測試中，我在一個執行緒中建立 `SolMsg`，並將它傳遞給另一個執行緒，然後在接收端驗證其內容，確保 `SolMsg` 的所有權轉移是安全的。

### 克服跨平台的 CI/CD 挑戰

在開發一個像 `rsolace` 這樣的函式庫時，確保它能在各種不同的作業系統和硬體架構上穩定運行是至關重要的。然而，在設定 CI/CD 流程時，我遇到了一些棘手的問題，特別是在 Windows 和 ARM 架構的 Linux 環境下。

為了解決這些問題，我進行了以下調整：

*   **`bindgen` 的相容性問題**：`bindgen` 是我們用來自動產生 Rust FFI (Foreign Function Interface) 綁定的工具。在 Windows 環境下，我遇到了 `bindgen` 和 Clang 之間的一些相容性問題，導致了 panic。為了解決這個問題，我將 `bindgen` 的版本從 0.65.1 升級到了 0.72.0，並對 Clang 的參數進行了微調，以確保在 Windows x86 和 x64 環境下都能順利產生綁定。
*   **Linux ARM 環境的設定**：在 Linux ARM 環境下，我遇到了 `pip` 的 `externally-managed-environment` 錯誤。為了解決這個問題，我將 CI 環境從 Ubuntu 20.04 升級到了 22.04，並使用了 `uv` 來管理 Python 的虛擬環境，取代了原本的 `apt` 安裝方式。
*   **`docs.rs` 的支援**：`docs.rs` 是 Rust 社群的官方文件託管網站。為了讓 `rsolace` 能夠在 `docs.rs` 上成功建立文件，我調整了 `build.rs`，讓它在偵測到 `DOCS_RS` 環境變數時，會跳過下載 Solace C 函式庫和產生綁定的步驟，改為使用預先產生的綁定。

### 結論

經過這次的迭代，`rsolace` 不再僅僅是一個功能的實現。透過引入 `Pin`，我們從根本上解決了 FFI 中最危險的記憶體安全問題；藉由整合 `async/await`，我們賦予了它現代化的非同步處理能力；而對 API 的精心調整與全面的 CI/CD 流程，則確保了其在多平台上的執行緒安全與穩定性。這些改進共同將 `rsolace` 推向了一個新的高度，使其成為一個更加可靠、高效且易於整合的生產級函式庫，為 Rust 生態系與 Solace 的結合提供了堅實的基礎。