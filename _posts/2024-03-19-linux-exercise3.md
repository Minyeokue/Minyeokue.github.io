---
title: 리눅스 종합문제 3
author: minyeokue
date: 2024-03-19 22:16:22 +0900
last_modified_at: 2024-03-20 02:30:55 +0900
categories: [Exercise]
tags: [Linux, Window, Network, Secure]

toc: true
toc_sticky: true
---

Bastion Host 구성 - HAProxy 서버(라운드로빈 부하 분산), Apache 웹서버, MariaDB 서버 등

<br>

---

<br>

## 시나리오

<br>

    centos8 => 10.10.10.10/24
    centos8-2 => 10.10.10.20/24

    win2003-1 => 10.10.10.30/24

    centos7 NGW
        공인망 192.168.1.111
        사설망 GW 10.10.10.1

<br>

1. centos8,8-2,win2003-1 웹서버 설치

2. haproxy 외부에서 접속 단 front 단 https://192.168.1.111

3. mariadb-server 설치 복제서버

4. haproxy 이용해서 로드벨런서 생성해서 확인
        단(centos8 / centos8-2) 방화벽 해제...

5. centos8 NFS 서버 구성 /var/www/html 공유
   centos8-2 NFS 클라이언트 /var/www/html 마운트

<br>

## 실습 환경 구축

<br>

#### CentOS7 (NGW;NAT GateWay), CentOS8 랜카드 설정, CentOS8-2 랜카드 설정

<br>

Bastion Host를 구축하기 위해 서버들이 존재하는 internal 네트워크(사설망)과 외부 인터넷(인터넷의 모든 컴퓨터들)과의 통신을 위한 공인 IP(가상의 공인 IP)로 설정하기 위해서는 2개의 랜카드가 필요하기 때문에 랜카드를 추가해준다.

<br>

![NAT 추가](/assets/img/2024-03-19/1.png)
_NAT 추가_

<br>

로그인 후 `hostnamectl` 명령어로 localhost라는 이름을 ngw로 변경한다.

```zsh
[root@localhost ~]# hostnamectl set-hostname ngw
[root@localhost ~]# exit
```

<br>

다시 로그인하면 호스트 네임이 바뀐다.

<br>

기존에 장착된 랜카드 이름은 'ens33', 장착한 랜카드를 설정하기 전에 랜카드의 이름을 확인하기 위해 `ip a` or `ip addr`을 입력한다.

```zsh
[root@ngw ~]# ip a

...
ensXX
...
...
ens33
...

# 이름 확인

[root@ngw ~]# nmtui
```

<br>

![nmtui ens36](/assets/img/2024-03-19/2.png)
_nmtui ens36_

<br>

> CentOS7 버전은 `systemctl restart network`이라는 명령어로 전체 랜카드들을 재시작할 수 있다.
{3: .prompt-tip }

정석은 `nmcli con up ens33`, `nmcli con up ens36`을 연달아 입력하는 것이다.

<br>

현재 다른 내부망 컴퓨터 CentOS8(10.10.10.10/24), CentOS8-2(10.10.10.20/24), win2003-1(10.10.10.30/24)는 통신이 안되는 상태이기 때문에, Masquerading을 해주기 위해 방화벽을 활성화 한다.

<br>

`systemctl start firewalld && systemctl enable firewalld`를 한 번에 실행하기 위해 `systemctl enable --now firewalld` 명령어를 입력

<br>

```zsh
[root@ngw ~]# systemctl enable --now firewalld
```

<br>

내부망과 공인망을 분리해주기 전 확인하고, 내-외부를 분리해주겠다.

<br>

```zsh
[root@ngw ~]# firewall-cmd --get-active-zones
public
  interfaces: ens33 ens36

# 분리
[root@ngw ~]# nmcli c mod ens33 connection.zone external
[root@ngw ~]# nmcli c mod ens36 connection.zone internal

[root@ngw ~]# firewall-cmd --get-active-zones
internal
  interfaces: ens36
external
  interfaces: ens33
```

<br>

이제 마스커레이딩을 해주기 위해서 다음 명령어를 입력한다.

```zsh
[root@ngw ~]# firewall-cmd --permanent --add-masquerade --zone=external
...
success

# 적용
[root@ngw ~]# firewall-cmd --reload

# 확인
[root@ngw ~]# firewall-cmd --zone=external --list-all
external (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33
  sources: 
  services: ssh
  ports: 
  protocols:
  masquerade: yes # 적용 완료
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
```

<br>

Bastion Host에서 ssh로 원격 접속하기 위해 `ssh-keygen` 명령어로 rsa키를 생성한다.

<br>

```zsh
[root@ngw ~]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): # Enter
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): # Enter
Enter same passphrase again: # Enter
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:fa9f4AuM0uoDTKmeGzpHOzv/tB/PIZnJcEQI3nwZXio root@ngw
The key's randomart image is:
+---[RSA 2048]----+
|      .. .o .    |
|     . o.o =     |
|      ..E *      |
|      o  =       |
|     +  S o . .  |
|    o o  = B o . |
|   o.o .o X + o .|
|  ..B. ..+ = + o |
|  .oo*.o=o. +.o  |
+----[SHA256]-----+
```

<br>

```zsh
[root@ngw ~]# cd ~/.ssh
[root@ngw .ssh]# ls
id_rsa  id_rsa.pub
```

키가 ~/.ssh 디렉토리 안에 생성되었다.

<br>

키 생성을 완료했으니 내부망 컴퓨터에 키를 복사하기 전 CentOS8, CentOS8-2가 켜져있고, IP 설정이 완료되어야 한다.

<br>

CentOS8 IP 설정

![CentOS8 nmtui](/assets/img/2024-03-19/3.png)
_CentOS8 nmtui_

<br>

```zsh
[root@centos8 ~]# nmcli con up ens160
```

<br>

CentOS8-2 IP 설정

![CentOS8-2 nmtui](/assets/img/2024-03-19/4.png)
_CentOS8 nmtui_

<br>

```zsh
[root@centos8-2 ~]# nmcli con up ens160
```

<br>

이제 `ssh-copy-id` 명령어로 ssh 키를 복사하겠다.

```zsh
[root@ngw .ssh]# ssh-copy-id 10.10.10.10
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host '10.10.10.10 (10.10.10.10)' can't be established.
ECDSA key fingerprint is SHA256:STBE61BxkMnkxeEbs3SFRI9CK09vBMWCdckwCRObvng.
ECDSA key fingerprint is MD5:65:6d:0d:59:7c:2e:be:d6:45:df:d7:bc:4b:1e:ab:fd.
Are you sure you want to continue connecting (yes/no)? yes # Enter
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@10.10.10.10's password: #111111

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh '10.10.10.10'"
and check to make sure that only the key(s) you wanted were added.

[root@ngw .ssh]# ssh-copy-id 10.10.10.20
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host '10.10.10.20 (10.10.10.20)' can't be established.
ECDSA key fingerprint is SHA256:STBE61BxkMnkxeEbs3SFRI9CK09vBMWCdckwCRObvng.
ECDSA key fingerprint is MD5:65:6d:0d:59:7c:2e:be:d6:45:df:d7:bc:4b:1e:ab:fd.
Are you sure you want to continue connecting (yes/no)? yes # Enter
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@10.10.10.20's password: #111111

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh '10.10.10.20'"
and check to make sure that only the key(s) you wanted were added.
```

<br>

이제 `ssh` 명령어로 키인증을 통해 비밀번호를 묻지 않고 원격 로그인이 원활히 진행될 것이다.

<br>

그 전에 `/etc/hosts`{: filepath}를 수정해 별칭으로 로그인해보겠다.

```zsh
[root@ngw ~]# vim /etc/hosts
```

```vim
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.10.10.10     web01
10.10.10.20     web02
10.10.10.30     web03

:wq
```

<br>

`ssh web01`와 `ssh web02`를 입력해 로그인이 가능해졌다.

<br>

---

<br>

#### 웹서버 구축

<br>

웹 서버와 MariaDB 패키지를 설치한다. (feat. vim)

<br>

```zsh
[root@centos8 ~]# yum -y install vim httpd mariadb-server

# 각 가상머신에서 실행

[root@centos8-2 ~]# yum -y install vim httpd mariadb-server
```

<br>

웹 서버 문서들의 기본 경로인 `/var/www/html`{: .filepath}

```zsh
[root@centos8 ~]# cd /var/www/html

[root@centos8 html]# echo web01 site centos8 > index.html

# 각 가상머신에서 진행

[root@centos8-2 ~]# cd /var/www/html

[root@centos8-2 html]# echo web01 site centos8-2 > index.html
```

<br>

각 머신의 웹 서버를 활성화

```zsh
[root@centos8 ~]# systemctl enable --now httpd
 
# 각 가상머신에서 진행

[root@centos8-2 ~]# systemctl enable --now httpd
```

---

<br>

윈도우 서버에서도 웹 서버를 가동하기 전 먼저 IP 설정을 해주겠다.

<br>

![윈도우 IP 설정](/assets/img/2024-03-19/5.png)
_윈도우 IP 설정_

<br>

설정 완료 후 파일 탐색기로 웹서버 기본 디렉토리로 이동한다.

<br>

![시작메뉴 파일탐색기](/assets/img/2024-03-19/6.png)
_시작메뉴 파일탐색기_

<br>

윈도우 2003 기준 웹서버 기본 경로는 `C:\Inetpub\wwwroot`{: .filepath}이며, 해당 경로에 index.htm 파일을 생성하겠다.

<br>

![윈도우 서버 웹서버 경로](/assets/img/2024-03-19/7.png)
_윈도우 서버 웹서버 경로_

<br>

![윈도우 서버 htm 파일 저장](/assets/img/2024-03-19/8.png)
_윈도우 서버 htm 파일 저장_

<br>

각 서버를 CentOS7에서 정상 작동 확인

![웹 서버 정상 작동](/assets/img/2024-03-19/9.png)
_웹 서버 정상 작동_

<br>

---

<br>

#### CentOS7 HAProxy 서버 구축

<br>

HAProxy 서버 패키지인 `haproxy`를 설치 후 haproxy configure 파일을 수정한다.
`/etc/haproxy/haproxy.cfg`{: .filepath}

```zsh
[root@ngw ~]# yum -y install haproxy

...
...

[root@ngw ~]# vim /etc/haproxy/haproxy.cfg
```
```vim
 ...
 ...
 63 frontend  main *:80 # 80으로 수정
 64     acl url_static       path_beg       -i /static /images /javascript /stylesheets
 65     acl url_static       path_end       -i .jpg .gif .png .css .js
 66 
 67     use_backend static          if url_static
 68     default_backend             app
 69 
 70 #---------------------------------------------------------------------
 71 # static backend for serving up images, stylesheets and such
 72 #---------------------------------------------------------------------
 73 backend static
 74     balance     roundrobin
 75     server      static 127.0.0.1:4331 check
 76 
 77 #---------------------------------------------------------------------
 78 # round robin balancing between the various backends
 79 #---------------------------------------------------------------------
 80 backend app
 81     balance     roundrobin
 82     server  app1 10.10.10.10:80 check   # 수정
 83     server  app2 10.10.10.20:80 check   # 수정
 84     server  app3 10.10.10.30:80 check   # 수정
~                                
~                                
:wq                                                      
```

<br>

저장 후 HAProxy를 가동하기 전, firewalld를 활성화했는데 포트가 열려있지 않다면 외부에서 Bastion Host에 공인 IP에 접근을 할 수 없을 것이다.

<br>

현재 NAT의 방화벽 상태를 확인하고, 해당 서비스(http)를 활성화하겠다.

```zsh
[root@ngw ~]# firewall-cmd --list-all --zone=external
external (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33
  sources: 
  services: ssh
  ports: 
  protocols: 
  masquerade: yes
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

[root@ngw ~]# firewall-cmd --permanent --add-service=http --zone=external
success
[root@ngw ~]# firewall-cmd --reload
success
[root@ngw ~]# firewall-cmd --list-all --zone=external
external (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33
  sources: 
  services: http ssh
  ports: 
  protocols: 
  masquerade: yes
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

```

<br>

이제 HAProxy를 활성화하고 외부에서 확인해보자.

```zsh
[root@ngw ~]# systemctl enable --now haproxy

[root@ngw ~]# systemctl status haproxy
```

> haproxy 서비스는 활성화할 때 에러 메시지를 출력하지 않기 때문에 반드시 `systemctl status haproxy` 명령어로 active 상태인지 확인하자.
{3: .prompt-tip }

<br>

이제 외부 컴퓨터에서 확인해본다.

![외부 컴퓨터에서 확인](/assets/img/2024-03-19/10.gif)
_외부 컴퓨터에서 확인_

<br>

---

<br>

#### CentOS8을 MariaDB master 서버, CentOS8-2를 MariaDB slave 서버로 만들고 실시간 복제 및 DB 이중화

<br>

mariadb 한글이 깨지지 않도록 진행하고 master 서버와 slave 서버로 이중화하겠다.

<br>

먼저 MariaDB 설정 파일들이 있는 `/etc/my.cnf.d/`{: .filepath}에 이동한다.

<br>

```zsh
[root@centos8 ~]# cd /etc/my.cnf.d/
[root@centos8 my.cnf.d]# ls
auth_gssapi.cnf  client.cnf  enable_encryption.preset  mariadb-server.cnf  mysql-clients.cnf
```

<br>

client.cnf, mariadb-server.cnf, mysql-clients.cnf 파일을 수정한다.

<br>

```zsh
[root@centos8 ~]# vim client.cnf
```

```vim
...
  6 
  7 [client]
  8 default-character-set=utf8 # 수정
  9 # This group is not read by mysql client library,
...
:wq
```

<br>

```zsh
[root@centos8 ~]# vim mysql-clients.cnf
```

```vim
  1 #
  2 # These groups are read by MariaDB command-line tools
  3 # Use it for options that affect only one utility
  4 #
  5 
  6 [mysql]
  7 default-character-set=utf8 # 수정
  8 [mysql_upgrade]
  9 
 10 [mysqladmin]
 11 
 12 [mysqlbinlog]
 13 
 14 [mysqlcheck]
 15 
 16 [mysqldump]
 17 default-character-set=utf8 # 수정
 18 [mysqlimport]
 19 
 20 [mysqlshow]
 21 
 22 [mysqlslap]
 23 

:wq
```

<br>

```zsh
[root@centos8 ~]# vim mariadb-server.cnf
```

```vim
...
...
# instructions in http://fedoraproject.org/wiki/Systemd
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mariadb/mariadb.log
pid-file=/run/mariadb/mariadb.pid
character-set-server=utf8           # 추가
collation-server=utf8_general_ci    # 추가
init_connect=set\ names\ utf8       # 추가
...

...
# MariaDB Master 서버 지정 -> 맨 아래로 이동 (shift + g)
...

...
# use this group for options that older servers don't understand
[mariadb-10.3]
log-bin=mysql-bin   # 추가
server-id=1         # 추가
binlog_format=row   # 추가
expire_logs_days=2  # 추가

:wq
```

<br>

cnf 파일을 수정하는 것은 끝났으니, mariadb-server를 활성화한다.

<br>

```zsh
[root@centos8 ~]# systemctl enable --now mariadb

[root@centos8 ~]# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 11
Server version: 10.3.28-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> status;
--------------
mysql  Ver 15.1 Distrib 10.3.28-MariaDB, for Linux (x86_64) using readline 5.1

Connection id:		10
Current database:	
Current user:		root@localhost
SSL:			Not in use
Current pager:		stdout
Using outfile:		''
Using delimiter:	;
Server:			MariaDB
Server version:		10.3.28-MariaDB-log MariaDB Server
Protocol version:	10
Connection:		Localhost via UNIX socket
Server characterset:	utf8    # 문자셋 설정 성공
Db     characterset:	utf8    # 문자셋 설정 성공
Client characterset:	utf8    # 문자셋 설정 성공
Conn.  characterset:	utf8    # 문자셋 설정 성공
UNIX socket:		/var/lib/mysql/mysql.sock
Uptime:			12 min 21 sec

Threads: 7  Questions: 12  Slow queries: 0  Opens: 17  Flush tables: 1  Open tables: 11  Queries per second avg: 0.016
--------------

MariaDB [(none)]> show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000002 |      342 |              |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.000 sec)

MariaDB [(none)]> grant replication slave on *.* to 'slave_db'@'%' identified by '1234';
Query OK, 0 rows affected (0.000 sec)
# 복제 서버를 위한 master DB 설정

MariaDB [(none)]> grant all privileges on *.* to root@'%' identified by '4321';
Query OK, 0 rows affected (0.000 sec)
# 원격 접속을 위한 유저 추가 완료

```

<br>

master 서버 설정이 완료되었으니 slave 서버를 설정한다.

<br>

한글이 깨지지 않도록 위와 같게 `/etc/my.cnf.d/client.cnf`{: .filepath}, `/etc/my.cnf.d/mysql-clients.cnf`{: .filepath}의 수정은 동일하게 진행하지만, 마지막 `/etc/my.cnf.d/mariadb-server.cnf`{: .filepath}의 맨 마지막 부분은 주의해서 적는다.

<br>

```zsh
[root@centos8-2 ~]# vim mariadb-server.cnf
```

```vim
...
...
# instructions in http://fedoraproject.org/wiki/Systemd
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mariadb/mariadb.log
pid-file=/run/mariadb/mariadb.pid
character-set-server=utf8           # 추가
collation-server=utf8_general_ci    # 추가
init_connect=set\ names\ utf8       # 추가
...

...
# use this group for options that older servers don't understand
[mariadb-10.3]
log-bin=mysql-bin   # 추가
server-id=2         # 추가 server id로 master와 slave를 구분한다.
binlog_format=row   # 추가
expire_logs_days=2  # 추가

:wq
```


