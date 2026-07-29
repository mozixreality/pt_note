---
title: Help
description: 
tags:
  - GraphQL
  - Blind SQLi
  - 任意檔案上傳
  - 老舊 Linux 核心提權
date: 2026-04-20
---

# Help

- 練習日期:   2026/04/20
- 考點:       GraphQL、Blind SQLi、任意檔案上傳、老舊 Linux 核心提權

## GraphQL

發現 3000 port 有運行 GraphQL，開始撈資料。詳細可以參考 Exploitation -> graphql.md。

## Blind SQLi

透過使用者帳號登入後，我們發現可以自行上傳檔案及下載檔案，透過下載檔案的鏈結可以發現，該鏈結存在 SQLi 的漏洞，sqlmap 插一插，隨便撈出來就可以發現幾組帳號密碼。

## 任意檔案上傳

這個我沒有嘗試。HelpDeskZ 上傳的命名規則是 md5(filename + time)，我們可以嘗試上傳一個 webshell，並窮舉上傳前後幾秒的實間做 md5 hash，並存取 `http://<ip>:<port>/uploads/<md5_hash>`，就可以使用 webshell。

## 老舊 Linux 核心提權

透過列舉幾個 username 及 password 後，發現有一組帳密可以成功登入。登入後使用 `uname -a` 發現 Linux 核心版本為 4.4.0-116-generic，在 exploit-db 上找到適合腳本，丟上去跑一跑就拿到 root 了。