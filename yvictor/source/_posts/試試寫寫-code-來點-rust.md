---
title: 試試寫寫 code 來點 rust
date: 2023-03-05 19:56:13
tags:
    - code
    - rust
---

Rust 是一個現代的系統程式語言，具有高效的性能和強大的安全性。在 Rust 的設計中，強調的是安全性和可靠性，這使得 Rust 成為了很多領域的選擇，如區塊鏈、WebAssembly 等等。在這篇文章中，我們將嘗試用 Rust 編寫一些簡單的程式碼，了解一下 Rust 的一些基礎概念。

### 安裝 Rust

``` bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### 開新專案
``` bash
cargo new meet-serde
```

### 安裝 serde
```
cargo add serde serde_json
```

### 開始

這裡使用的是 serde_json 库來實現對 json 格式的支持，serde 库用於序列化和反序列化。

接著，定義一個自定義的結構 Person，並使用 serde 宏對其進行序列化和反序列化的支持：

``` rust
extern crate serde;

use serde::{Deserialize, Serialize};


#[derive(Debug, PartialEq, Deserialize, Serialize)]
struct Person {
    name: String,
    age: u32,
}
```

在這個結構中，我們使用了 `serde` 宏的 `Serialize` 和 `Deserialize` 屬性來實現對結構的序列化和反序列化的支持。

下面是一個對 `Person` 結構進行序列化和反序列化的例子：

`main.rs`

``` rust

fn main(){
    // 定義一個 Person 結構
    let person = Person {
        name: String::from("John"),
        age: 25,
    };

    // 將 Person 結構序列化為 json 格式
    let serialized = serde_json::to_string(&person).unwrap();
    println!("serialized = {}", serialized);

    // // 將 json 格式反序列化為 Person 結構
    let decoded_person: Person = serde_json::from_str(&serialized).unwrap();
    // // 輸出反序列化得到的 Person 結構
    println!("deserialized = {:?}", decoded_person);

}
```
在這個例子中，我們首先定義了一個 Person 結構，然後將其序列化為 json 格式並存儲到 serialized 中。接著，我們使用 serde_json 函数將其反序列化為 Person 結構。

Ok 第一篇有 code 的文章測試一下排版，打完收工。