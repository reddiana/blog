---
title: CANONICAL Multipass
description: 리눅스 서버 마련이 제일 쉬웠어요
layout: post
categories: [Ubuntu, multipass]
toc: true
---

## CANONICAL Multipass

- [Multipass orchestrates virtual Ubuntu instances](https://multipass.run/)
- [[Ubuntu\] multipass를 이용한 우분투 서버 가상환경 구축 :: 또치의 삽질 보관함 (tistory.com)](https://ddochea.tistory.com/68)

## SSH Client로 접속

계정/비전으로는 접속이 안된다.

`id_rsa.pub` 내용을 `authorized_keys` 파일에 추가하고 인증서로 로그인하자

```bash
cat >> ~/.ssh/authorized_keys
```

## Static IP 설정

vm이 dhcp를 써서 ssh client로 접속할 때 무진장 귀찮다.

```bash
#!/bin/bash
MY_IP=$(ip a | awk '/inet.*eth0/{print $2}')
MY_GW=$(ip r | awk '/default/{print $3}')
MY_MAC=$(ip a show dev eth0 | awk '/link\/eth/{print $2}')

cat << EOF > /etc/netplan/50-cloud-init.yaml
network:
 version: 2
 renderer: networkd
 ethernets:
   enp0s8:
     dhcp4: no
     addresses: [$MY_IP]
     gateway4: $MY_GW
     nameservers:
       addresses: 
         - 8.8.8.8
         - 8.8.4.4
#         - 70.10.98.4
#         - 203.241.132.34
#         - 203.241.132.85
#         - 203.241.135.130
#         - 203.241.135.135         
     match:
       macaddress: $MY_MAC
EOF
netplan apply
# curl www.google.com
```

## multipass cmd Windows alias 설정

#### doskey

multipass라는 명령이 은근히 매번 타이핑 귀찮다. Windows에도 `doskey`라는 alias 명령어가 있다.

- alias.cmd

```cmd
@echo off

doskey m       = multipass $*
:: commands
doskey alias   = doskey $*
doskey cat     = type $*
doskey clear   = cls
doskey grep    = find $*
doskey history = doskey /history
doskey man     = help $*
::
doskey kill    = taskkill /PID $*
doskey ls      = dir $*
doskey ll      = dir $*
::
doskey cp      = copy $*
doskey cpr     = xcopy $*
doskey mv      = move $*
doskey rm      = del $*
doskey rmr     = deltree $*
::
doskey ps      = tasklist $*
doskey pwd     = cd
::
doskey sudo    = runas /user:administrator $*
```

#### 시작 시 doskey 적용 .reg

위 alias.cmd를 레지스트리에 등록하여 도스창 열릴 때 마다 적용 시킨다.

[alias - Aliases in Windows command prompt - Stack Overflow](https://stackoverflow.com/questions/20530996/aliases-in-windows-command-prompt)

- alias_auto_set.reg

```
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\Microsoft\Command Processor]
"AutoRun"="%USERPROFILE%\\alias.cmd"
```
