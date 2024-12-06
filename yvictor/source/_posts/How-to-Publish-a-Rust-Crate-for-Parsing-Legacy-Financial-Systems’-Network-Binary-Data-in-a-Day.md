---
title: >-
  一天內發佈一個解析舊有金融系統網路二進位資料的 Rust Crate
date: 2024-12-06 17:22:32
tags:
    - code
    - rust
---


在本教學中，我們將使用 Rust 建立並發佈一個 crate，用於解析舊有金融系統的二進位網路資料。我們會採用測試驅動開發（TDD）方法：先撰寫測試，再實作功能。每個小步驟都有測試作為品管，確保正確性。我們也會討論如何利用 [Cursor 編輯器](https://www.cursor.so/) 與 Claude 這類大型語言模型（LLM）來加速開發。有了 TDD 作為安全防護，您可以自信地使用 AI 建議來快速迭代開發。

**完整程式碼存放於：** [https://github.com/Yvictor/binary_mirror](https://github.com/Yvictor/binary_mirror)

## 為什麼使用 TDD 與 AI 協助？

使用 TDD 時，每個新功能的開發流程是：先寫一個預期失敗的測試（定義需求），再寫出剛好能通過該測試的程式碼，避免過度設計。同時搭配 Cursor 編輯器與 Claude AI，可快速產生程式碼片段與建議。TDD 確保即使 AI 給出不完全正確的建議，我們也能透過測試來驗證、修正，最終得到符合預期的結果。

## 第 1 步：初始化專案並撰寫首個測試

**概念：** 從建立一個 library crate 開始，並在實作任何功能前先撰寫一個會失敗的測試。例如，我們期望能有一個 `BinaryMirror` 派生巨集以及解析 `str` 欄位的功能，但尚未實作。

**指令：**
```bash
cargo new binary-mirror --lib
cd binary-mirror
mkdir tests
touch tests/derive_tests.rs
```

### 初始測試（tests/derive_tests.rs）：
``` rust
use binary_mirror::BinaryMirror;

#[repr(C)]
#[derive(Debug, BinaryMirror)]
struct TestStruct {
    #[bm(type = "str")]
    name: [u8; 10],
}

#[test]
fn test_str_field() {
    let record = TestStruct {
        name: *b"Hello    ",
    };
    assert_eq!(record.name(), "Hello");
}
```
現在執行 cargo test 會失敗，因為還沒有實作 BinaryMirror。這個測試明確告訴我們下一步該做什麼：實作 str 字串解析。

## 第 2 步：實作最基本的 str 欄位解析

透過 TDD：測試告訴我們需要解析 str 欄位。您可在 Cursor 中詢問 AI 如何建立基本的 proc macro，再根據測試結果進行微調。

加入必要套件：
```bash
cargo add syn --features full,extra-traits
cargo add quote
cargo add proc-macro2
```

src/lib.rs 實作：
``` rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput, Data, Fields};

#[proc_macro_derive(BinaryMirror, attributes(bm))]
pub fn binary_mirror_derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;
    let fields = match input.data {
        Data::Struct(ref data) => match data.fields {
            Fields::Named(ref fields) => &fields.named,
            _ => panic!("只支援有命名欄位的結構"),
        },
        _ => panic!("只支援結構"),
    };

    let mut methods = Vec::new();
    for f in fields {
        let field_name = f.ident.as_ref().unwrap();
        let bm_type = f.attrs.iter().find_map(|attr| {
            let tokens = attr.tokens.to_string();
            if tokens.contains("\"str\"") {
                Some("str".to_string())
            } else {
                None
            }
        });

        if let Some("str") = bm_type.as_deref() {
            methods.push(quote! {
                pub fn #field_name(&self) -> String {
                    String::from_utf8_lossy(&self.#field_name).trim().to_string()
                }
            });
        }
    }

    let expanded = quote! {
        impl #name {
            #(#methods)*
        }
    };

    expanded.into()
}
```

再次測試：
```bash
cargo test
```
現在測試會通過。由於有 TDD 驗證，我們可確定 AI 協助的程式碼符合需求。

## 第 3 步：為整數欄位（i32）撰寫測試，再實作

接著我們需要解析數字欄位。在實作前先寫測試：

``` rust
#[repr(C)]
#[derive(Debug, BinaryMirror)]
struct NumericStruct {
    #[bm(type = "i32")]
    value: [u8; 4],
}

#[test]
fn test_i32_field() {
    let rec = NumericStruct {
        value: *b"123 ",
    };
    assert_eq!(rec.value(), Some(123));
}
```
cargo test 現在失敗了，我們必須實作 i32 解析。

### 第 4 步：實作 i32 解析

更新 src/lib.rs 程式碼，加入 i32 支援：
``` rust
if let Some("i32") = bm_type.as_deref() {
    methods.push(quote! {
        pub fn #field_name(&self) -> Option<i32> {
            String::from_utf8_lossy(&self.#field_name).trim().parse::<i32>().ok()
        }
    });
}
```
再次 `cargo test`，現在測試通過。

## 第 5 步：為別名（alias）功能撰寫測試，再實作
先寫測試，預期有 alias 能將方法命名改為 exchange()：
``` rust
#[repr(C)]
#[derive(Debug, BinaryMirror)]
struct AliasStruct {
    #[bm(type = "str", alias = "exchange")]
    exh: [u8; 10],
}

#[test]
fn test_alias_field() {
    let rec = AliasStruct {
        exh: *b"CME       ",
    };
    assert_eq!(rec.exchange(), "CME");
}
```
cargo test 現在失敗了，我們必須實作 alias 功能。
告訴 cursor 我們需要實作 alias 功能，直到 cargo test 通過。

## 第 6 步：為日期與時間解析撰寫測試，再實作
加上 chrono 套件：
```bash
cargo add chrono
```
寫測試解析日期與時間：
``` rust
use chrono::{NaiveDate, NaiveTime};

#[repr(C)]
#[derive(Debug, BinaryMirror)]
struct DateTimeStruct {
    #[bm(type = "date", format = "%Y%m%d")]
    date_field: [u8; 8],
    #[bm(type = "time", format = "%H%M%S")]
    time_field: [u8; 6],
}

#[test]
fn test_date_time_fields() {
    let rec = DateTimeStruct {
        date_field: *b"20240101",
        time_field: *b"123456",
    };
    assert_eq!(rec.date_field(), Some(NaiveDate::from_ymd_opt(2024,1,1).unwrap()));
    assert_eq!(rec.time_field(), Some(NaiveTime::from_hms_opt(12,34,56).unwrap()));
}
```
跑測試失敗後，再讓 AI 幫你實作 date/time 格式解析，透過 TDD 確保功能正確。

## 第 7 步：為合併日期與時間（datetime_with）寫測試，再實作
撰寫測試期望有 datetime_with 能將日期與時間合併成 NaiveDateTime。測試失敗後實作此功能，再測試應可通過。

## 第 8 步：為錯誤資料寫最後的測試
測試無效資料能安全返回 None。通過後即可確信穩健性

## 第 9 步：發佈 Crate
所有測試通過且功能完整，準備發佈：
```bash
cargo login
cargo publish --dry-run
cargo publish
```
您的 crate 已經上架到 crates.io，其他人可用 `cargo add binary_mirror` 來使用。

## 結論

透過 TDD 與 AI 協助，我們在一天內成功發佈了 BinaryMirror crate。這不僅展示了 Rust 的強大功能，也展示了如何利用 AI 工具來加速開發過程。希望這個範例能激勵您在開發過程中探索更多可能性。

完整程式碼存放於： [https://github.com/Yvictor/binary_mirror](https://github.com/Yvictor/binary_mirror)
