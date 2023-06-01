---
title: 如何建立一個具備可測試性的 Python 交易 package 使用 Shioaji (Part 1)
date: 2023-06-01 23:13:53
tags:
    - trading
    - python
    - shioaji
    - pytest
categories: shioaji
---

#### 前言

因為是第一篇所以先簡單介紹一下台灣第一套可以跨平台的金融交易 API solution，Shioaji 目前主要語言為 Python，提供開發者進行證期選交易及即時歷史行情獲取，其高效且易於使用的特性受到開發者的青睞，它能夠與像 NumPy、pandas 或 PyTorch 等流行的 Python 套件進行整合，讓使用者能夠創建出專屬於自己的跨平台交易模型。

特色:

- 高效率：C++ 作為核心邏輯，並且利用 FPGA 進行訊息交換，從而提高整體效能和執行速度。
- 簡單易用：設計為易於使用和學習，提供了直觀且符合 Pythonic 風格的介面，讓開發者能夠快速上手並開始進行金融交易。

- 快速開發：與原生 Python 緊密集成，充分利用生態系統中的豐富套件，從而加速開發過程並提高效率。

- 跨平台支援：作為台灣第一個兼容 Linux 與 Mac 的交易 API，讓開發者能夠在不同的作業系統上使用相同的介面進行量化金融交易。

好了廢話不多說[Shioaji](https://sinotrade.github.io/) 的官方文件在這邊，關於怎麼開通等等的都在文件裡面有，這邊就從當你已經準備好 API 權限後開始教起。

#### 開新專案

方法一 : 當你今天已經有 Python 環境了我這邊選擇 flit 來快速開啟一個新專案

1. 確保你已經安裝了 Flit：在終端機中執行以下命令來安裝 Flit：
``` bash
pip install flit
```
2. 執行 Flit 命令：在終端機中初始化 python project 這系列的範例就用 sjtrade
``` bash
mkdir sjtrade
cd sjtrade
flit init
```
他會出現一個一個要你填的你就跟著填入就可以了
```
Module name [temp]: sjtrade
Author [yvictor]: 
Author email [yvictor3141@gmail.com]: 
Home page [https://github.com/Yvictor/sjtrade]: 
```
這邊填完你就會看到他幫你創建好的 pyproject.toml 這個檔案產生了

3. Flit有一個淺規則要在你的 python module 中加入 doc string跟 `__version__`
``` bash
mkdir sjtrade
```
新增一個 `__init__.py` 在 sjtrade folder 底下並加入下面這段
``` python 
""" sjtrade
trading with shioaji 
"""
__version__ = "0.1"
```
這個就是 flit 的淺規則
詳細的部分可以去看 [flit](https://flit.pypa.io/en/stable/index.html) 的文件這邊就不多說