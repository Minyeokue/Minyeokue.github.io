---
title: 리눅스 실습문제 4
author: minyeokue
date: 2024-03-21 16:47:38 +0900
last_modified_at: 2024-03-22 19:55:47 +0900
categories: [Exercise]
tags: [Linux, MariaDB, Mail, DNS, DHCP]

toc: true
toc_sticky: true
---

<br>

사내 네트워크에 Bastion Host와, DNS 서버 2대(주 DNS, 보조 DNS), DB 서버와 메일 서버 역할의 리눅스 1대, DHCP 서버 1대를 구축한다.

사내 네트워크 안에 2대의 윈도우 클라이언트가 DHCP로부터 자동으로 주소를 받아오고, 메일 서버의 클라이언트로 적용된다.

<br>

---

## 시나리오

<br>

- Bastion Host(NGW) 구축 -> **CentOS7**

    - 공인 IP : 192.168.1.111/24

    - 사설 GW : 10.10.10.1/24

    - DNS : 8.8.8.8

- DNS 서버 2대 구축 => CentOS8, CentOS8-2

    - 주 DNS 서버 -> **CentOS8**

        - IP : 10.10.10.10/24

        - GW : 10.10.10.1
    
    - 보조 DNS 서버 -> **CentOS8-2**

        - IP : 10.10.10.20/24

        - GW : 10.10.10.1

- 웹 메일 서버 + DB 서버 1대 구축 -> **Linux01**

    - IP : 10.10.10.30/24

    - GW : 10.10.10.1

- DHCP 서버 -> **Linux01-2**

    - IP : 10.10.10.40/24

    - GW : 10.10.10.1

<br>

- 윈도우 클라이언트 2대

<br>

---

#### 실습 환경 구축

<br>

---

<br>

##### Bastion Host(NGW)

<br>

`nmtui` 명령어를 사용해 2개의 랜카드를 다음의 사진 2장과 같이 설정한다.

~~ensXX 이라는 이름은 바뀔 수 있으니, ip a 명령어로 랜카드 이름을 확인한 후 진행하도록 한다.~~

<br>

![Bastion Host 공인 IP 랜카드 설정](/assets/img/2024-03-21/1.png)
_Bastion Host 공인 IP 랜카드 설정_

![Bastion Host 사설 GW 랜카드 설정](/assets/img/2024-03-21/2.png)
_Bastion Host 사설 GW 랜카드 설정_

<br>

###### 최상단에 있는 시나리오와 같이 `nmtui` 혹은 GUI를 통해 랜카드 설정을 마친 상태라고 생각하고 이후 작업을 진행한다.

<br>

> 리눅스의 경우 `/etc/sysconfig/network-scripts/ifcfg-XXXXXX` 혹은 `/etc/sysconfig/network` 파일에 직접 입력해 설정을 변경해줄 수도 있다.
{: .prompt-info }

<br>

`systemctl restart network` 또는 `service network restart` 명령어를 입력해 바뀐 사항들을 적용한다.

~~혹은 `nmcli con up [랜카드명]` or `reboot`로 전부 적용~~

<br>

사설망 안의 서버 혹은 클라이언트들이 Bastion Host를 통해 네트워크가 가능하도록 **Masquerading** 해주기 위해 firewall 데몬 서비스를 활성화한다.


> `systemctl enable --now firewalld` 혹은 `systemctl start firewalld && systemctl enable firewalld` 명령어를 입력해 활성화하는 동시에 시스템을 재부팅해도 서비스가 자동으로 활성화되도록 한다.
{: .prompt-tip }

<br>

2개의 랜카드가 달린 Bastion Host에 공인 IP와 사설 IP를 분리하기 위해 현재 랜카드의 영역이 어떤지 확인하고 분리를 진행해주도록 한다.

<br>

```zsh
[root@bastion ~]# firewall-cmd --get-active-zones
public
  interfaces: ens33 ens36
[root@bastion ~]# nmcli con mod ens33 connection.zone external
[root@bastion ~]# nmcli con mod ens36 connection.zone internal
[root@bastion ~]# firewall-cmd --get-active-zones
internal
  interfaces: ens36
external
  interfaces: ens33
```

<br>

**Masquerading**을 진행해줄텐데, 공인 IP를 담당하는 ens33에 "masquerade : yes"인 상태로 만들어줘야 사설망안의 서버 혹은 클라이언트들이 공인 IP로 **위장**해서 외부 인터넷으로 나갈 수 있다.

<br>

```zsh
[root@bastion ~]# firewall-cmd --permanent --add-masquerade --zone=external
...
success
```

<br>

이제 사설망 안의 서버들은 Bastion Host의 사설 GW와 외부와 연결된 랜카드의 **Masquerading**으로 인터넷이 가능한 상태가 되었다.

<br>

Bastion Host에서 리눅스 서버들을 관리하기 위해 ssh로 접속할 수 있도록 Bastion Host의 키를 생성해 리눅스 서버들에 생성한 키를 전송하겠다.

<br>

```zsh
[root@bastion ~]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:PG0GLs10/9L7AB7c5a0RF1zjY5TCpkEmQYsKu7fWm+0 root@bastion
The key's randomart image is:
+---[RSA 2048]----+
|        .+oo. .++|
|        . +. +ooo|
|   .   .o.. + o+o|
|    o .* + + ..=o|
|   . .. S + = o o|
|    .  . + . = o |
|   . ..     o =  |
|    ....o    . o |
|    .. ooE    ...|
+----[SHA256]-----+
```

```zsh
[root@bastion ~]# ssh-copy-id 10.10.10.10
...
...
[root@bastion ~]# ssh-copy-id 10.10.10.20
...
...
[root@bastion ~]# ssh-copy-id 10.10.10.30
...
...
[root@bastion ~]# ssh-copy-id 10.10.10.40
...
...
```

<br>

생성된 키를 각 리눅스 서버들에 복사했고, ssh로 해당 서버들에 접속할 수 있게 되었다.

이 컴퓨터에서 별칭으로 접속할 수 있도록 `/etc/hosts`{: .filepath }를 수정해주자.

<br>

```vim
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.111   bastion ngw
10.10.10.10     dns1
10.10.10.20     dns2
10.10.10.30     mail    db
10.10.10.40     dhcp
...
...
:wq
```

<br>

---

<br>

##### 주 DNS 서버 구축

<br>

DNS 서버에 관련된 패키지를 설치한다. `yum -y install caching-nameserver` or `yum -y install bind bind-chroot`

<br>

`/etc/named.conf`{: .filepath }를 수정하는데, CentOS8 기준 11번 줄, 19번 줄의 중괄호 안을 **any;**로 바꿔준다.

```vim
...

options {
        listen-on port 53 { any; };     # 수정
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { any; };       # 수정

...
...
}
:wq
```

<br>

`/etc/named.rfc1912.zones`{: .filepath }를 수정하는데, 영역의 이름은 gitblog.io로 하겠다.

```vim
...
# 바로 아래 블록을 수정
zone "gitblog.io" IN {
        type master;
        file "gitblog.io.zone";
        allow-transfer { 10.10.10.20; };    # 보조 DNS로 전달하겠다는 의미
};

# localhost 영역 위에 있는 중괄호로 감싸진 영역을 수정
zone "localhost" IN {
        type master;
        file "named.localhost";
        allow-update { none; };
};
...
...
:wq
```

<br>

이제 영역 파일을 하기 위해 `/var/named/`{: .filepath }로 이동해서, **named.localhost**라는 파일을 복사해 **gitblog.io.zone**이라는 파일을 생성한다.

> 파일 혹은 디렉토리를 복사할 때 소유자, 소유그룹, 부여된 권한까지 복사하려면 **-p 옵션을 추가한 `cp` 명령어**를 사용한다.
{: .prompt-tip }

<br>

만들어진 **gitblog.io.zone** 파일을 수정한다.

```vim
$TTL 1D
@       IN SOA  ns1.gitblog.io. minyeokue.gitblog.io. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      ns1.gitblog.io.
        NS      ns2.gitblog.io.
        MX 10   mail.gitblog.io.
        A       10.10.10.10
ns1     A       10.10.10.10
ns2     A       10.10.10.20
mail    A       10.10.10.30
db      A       10.10.10.30
dhcp    A       10.10.10.40  
...
:wq
```

<br>

> 영역 파일의 문법에 이상이 없는지 체크하려면 `named-checkzone gitblog.io gitblog.io.zone` 명령어로 확인할 수 있다. 일반적인 상황에서는 `named-checkzone [영역 이름] [영역 파일 이름]`이다.
{: .prompt-tip }

> 영역 파일을 검사하는 `named-checkzone`이외에도 `named-checkconf /etc/named.conf`, `named-checkconf /etc/named.rfc1912.zones` 명령어로 설정 파일의 문법에는 이상없는지 검사할 수 있다. => 기본적으로 `named-checkconf`만 입력해도 된다.
{: .prompt-info }

<br>

주 네임 서버를 가동시키기 전에, 보조 네임 서버를 설정해주도록 하겠다.

---

##### 보조 DNS 서버 구축

<br>

`/etc/named.conf`{: .filepath }를 주 네임 서버에서 수정했듯이 11번 줄과 19번 줄을 "any;"로 수정하고 저장한다.

`/etc/named.rfc1912.zones`{: .filepath }는 보조 DNS 서버에 맞게 수정해주도록 하겠다.

<br>

```vim
...

zone "gitblog.io" IN {
        type slave;
        file "slaves/gitblog.io.slave.zone";    # 보조 DNS 서버의 영역파일은 /var/named/slaves/ 안으로!
        masters { 10.10.10.10; };               # 주 DNS 서버의 IP
};

zone "localhost" IN {
        type master;
        file "named.localhost";
        allow-update { none; };
};

...
:wq
```

<br>

![주 DNS에서 보조 DNS로 영역 파일 전달](/assets/img/2024-03-21/3.gif)
_주 DNS에서 보조 DNS로 영역 파일 전달_

<br>

이렇게 영역 전송이 끝난 뒤에는 주 서버에서 `systemctl status named` 명령어를 입력하면 다음과 같은 로그를 얻을 수 있다.

```vim
...
...
Mar 21 20:05:51 dns1 named[10975]: client @0x7f3ca00b0870 10.10.10.20#43301 (gitblog.io): transfer of 'gitblog.io/IN': AXFR started (serial 0)
Mar 21 20:05:51 dns1 named[10975]: client @0x7f3ca00b0870 10.10.10.20#43301 (gitblog.io): transfer of 'gitblog.io/IN': AXFR ended
```

> 주 DNS 서버 입장에서 보조 DNS 서버(클라이언트)로 전체 영역 전송인 **AXFR**이 시작되고 끝났다는 로그를 확인. 만약 스텁 영역을 복사했다면 **IXFR**이 시작되고 끝났다는 로그가 출력되었을 것이다.
{: .prompt-info }

<br>

---

##### Roundcube 웹 메일서버 + DB 서버 구축

<br>

Roundcube를 활용한 웹 메일 서버를 구축하기 전에, 필수 패키지를 설치하고 설정 파일을 수정한다.

<br>

```zsh
[root@mail_db ~]# yum -y sendmail*
```

<br>

Sendmail이라는 패키지는 인터넷을 통해 이메일을 전송하는데 사용되는 SMTP(Simple Mail Transfer Protocol)의 약자로, 수많은 종류의 메일 전송 및 전달 방식을 지원하는 범용 목적 인터네트워크 이메일 라우팅 기능을 가진 패키지이다.

<br>

SMTP 인증 기능 추가하기 위해 **vim** `/etc/mail/sendmail.mc`{: .filepath }의 52번, 53번 줄의 맨 앞 `dnl`을 지운다.

<br>

![/etc/mail/sendmail.mc 수정](/assets/img/2024-03-21/4.png)
_/etc/mail/sendmail.mc 수정_

=> 그 이후 `m4 /etc/mail/sendmail.mc > /etc/mail/sendmail.cf` 명령어를 입력

<br>

![/etc/mail/sendmail.cf 수정1](/assets/img/2024-03-21/5.png)
_/etc/mail/sendmail.cf 수정1_

![/etc/mail/sendmail.cf 수정2](/assets/img/2024-03-21/6.png)
_/etc/mail/sendmail.cf 수정2_

<br>

릴레이 허용 파일을 수정하는데, 기본적으로는 local만 허용하지만, 웹 메일 서비스를 구축한다고 생각하고 `/etc/mail/access`{: .filepath } 파일을 수정한다.

![릴레이 허용 파일 수정](/assets/img/2024-03-21/7.png)
_릴레이 허용 파일 수정_

<br>

위 사진 처럼 수정한 뒤 `makemap hash /etc/mail/access < /etc/mail/access` 명령어를 실행한다.

<br>

이후 메일을 수신할 호스트 이름을 등록하기 위해 **vim** `/etc/mail/local-host-names`{: .filepath } 명령을 실행한다.

=> ~~DNS의 MX(메일교환기)를 위해서인지?~~

<br>

```vim
# local-host-names - include all aliases for your machine here.
gitblog.io
~
~
...
:wq
```

<br>

저장 후 보안 인증에 관한 서비스인 `saslauthd`와 메일을 보내는데 필수적인 서비스인 `sendmail` 서비스를 활성화한다.

```zsh
[root@mail_db ~]# systemctl enable --now saslauthd
[root@mail_db ~]# systemctl enable --now sendmail

# sendmail은 서비스를 활성화하는 데 시간이 오래 걸린다.
```

---

<br>

이제 이메일을 보내는 기능은 준비되었으니, 이메일을 받을 수 있는 프로토콜이 포함된 패키지를 받아야 한다. 해당 패키지 이름은 `dovecot`이며 `dovecot`은 기존의 `pop3d`와 `imapd`의 역할을 대신해주며, **주 목적은 메일 저장 서버 역할**을 하는 것이다.

<br>

포트 번호| 서비스명 | 기능 차이
:---:|:---:|:---:
110|pop3|서버에서 클라이언트로 전자메일을 다운로드하는 것만 허용
993|pop3s|pop3에 ssl 인증서를 사용해 보안기능 추가
143|imap|여러 전자 메일 클라이언트에서 접속 가능, 메일이 서버에 저장
995|imaps|imap에 ssl 인증서를 사용해 보안기능 추가

<br>

`yum -y install dovecot` 명령어를 실행해 패키지를 설치한다.

<br>

설치가 완료되면 `/etc/dovecot/dovecot.conf`{: .filepath }에서 사용할 프로토콜과 허용할 IP 대역에 대해 설정한다.

<br>

![/etc/dovecot/dovecot.conf](/assets/img/2024-03-21/8.png)
_/etc/dovecot/dovecot.conf_

<br>

`/etc/dovecot/conf.d/10-mail.conf`{: .filepath }를 수정해 메일이 저장될 장소를 지정한다. => `/var/spool/mail/`{: .filepath } 사용자 폴더에 저장

<br>

![/etc/dovecot/conf.d/10-mail.conf](/assets/img/2024-03-21/9.png)
_/etc/dovecot/conf.d/10-mail.conf_

<br>

`/etc/dovecot/conf.d/10-ssl.conf`{: .filepath }에서 ssl 인증을 required에서 yes로 변경

![/etc/dovecot/conf.d/10-ssl.conf](/assets/img/2024-03-21/10.png)
_/etc/dovecot/conf.d/10-ssl.conf_

<br>

`/etc/dovecot/conf.d/10-auth.conf`{: .filepath }에서 disable_plaintext_auth yes에서 no로 변경

![/etc/dovecot/conf.d/10-auth.conf](/assets/img/2024-03-21/11.png)
_/etc/dovecot/conf.d/10-auth.conf_

<br>

이제 `dovecot`을 활성화하기 위한 준비가 끝났다. `systemctl enable --now dovecot` 명령어를 실행한다.

<br>

Roundcube를 활용한 웹 메일 서버 구축에 필요 패키지를 설치한다.

```zsh
[root@mail_db ~]# yum -y install vim mariadb-server php php-mysqlnd php-gd php-mbstring php-pecl-zip php-xml php-json php-intl httpd
```

<br>

설치 후, Roundcube 깃허브 레포지토리의 1.3.7 버전을 `wget` 명령어를 통해 받도록 하겠다.

<br>

```zsh
[root@mail-db ~]# wget -c https://github.com/roundcube/roundcubemail/releases/download/1.3.7/roundcubemail-1.3.7-complete.tar.gz
```

<br>

.gz 압축 파일을 풀기 위해 `tar xfz roundcubemail-1.3.7-complete.tar.gz` 명령어를 입력하고, 압축이 풀리면 나오는 폴더를 `/var/www/html`{: .filepath }로 옮기도록 하겠다.

```zsh
[root@mail-db ~]# tar xfz roundcubemail-1.3.7-complete.tar.gz
[root@mail-db ~]# mv roundcubemail-1.3.7 /var/www/html
[root@mail-db ~]# cd /var/www/html
[root@mail-db html]# ls
roundcubemail-1.3.7
```

<br>

`/var/www/html/roundcube`{: .filepath} 로 접근해도 `/var/www/html/roundcubemail-1.3.7`{L .filepath}에 접근할 수 있도록 `ln -s` 명령어를 사용해 심볼릭 링크파일을 생성하겠다.

<br>

```zsh
[root@mail-db html]# ln -s /var/www/html/roundcubemail-1.3.7 /var/www/html/roundcube
[root@mail-db html]# ls
roundcube  roundcubemail-1.3.7
```

<br>

Roundcube 임시 파일과 로그 파일들이 저장되는 디렉토리에 모든 권한을 부여하겠다.

```zsh
[root@mail-db html]# chmod 777 /var/www/html/roundcube/temp/
[root@mail-db html]# chmod 777 /var/www/html/roundcube/logs/
[root@mail-db html]# ls -l roundcube/
total 232
drwxr-xr-x  2 501 80    331 Mar 21 20:26 bin
-rw-r--r--  1 501 80 154039 Jul 30  2018 CHANGELOG
-rw-r--r--  1 501 80   1046 Jul 30  2018 composer.json-dist
drwxr-xr-x  2 501 80     97 Mar 21 20:26 config
-rw-r--r--  1 501 80  12726 Jul 30  2018 index.php
-rw-r--r--  1 501 80  10901 Jul 30  2018 INSTALL
drwxr-xr-x  3 501 80    123 Mar 21 20:26 installer
-rw-r--r--  1 501 80  35147 Jul 30  2018 LICENSE
drwxrwxrwx  2 501 80     23 Mar 21 20:26 logs
drwxr-xr-x 35 501 80   4096 Mar 21 20:26 plugins
drwxr-xr-x  8 501 80     92 Mar 21 20:26 program
drwxr-xr-x  3 501 80     83 Mar 21 20:26 public_html
-rw-r--r--  1 501 80   3876 Jul 30  2018 README.md
drwxr-xr-x  4 501 80     34 Mar 21 20:26 skins
drwxr-xr-x  7 501 80    206 Jul 30  2018 SQL
drwxrwxrwx  2 501 80     23 Mar 21 20:26 temp
-rw-r--r--  1 501 80   3579 Jul 30  2018 UPGRADING
drwxr-xr-x  8 501 80    110 Mar 21 20:26 vendor
```

<br>

웹 메일 서비스인 Roundcube를 MariaDB와 연동시켜 서비스를 구축하기 위해 http 데몬 서비스와 mariadb 서버 데몬을 활성화한 뒤 필수적인 작업을 진행한다.

<br>

```zsh
[root@mail-db ~]# systemctl enable --now httpd          # "웹" 메일
[root@mail-db ~]# systemctl enable --now mariadb        # 웹 메일 DB 연동을 위함
```

> 같은 포트를 사용하고 있을 경우 서비스가 활성화되지 않을 수 있다. 예를 들어, 80번 포트를 사용하고 있는 서비스를 찾고 싶다면 `netstat -nltp | grep ':80'`을 입력해 해당 서비스 혹은 프로세스를 종료하자.
{: .prompt-tip }

<br>

MariaDB에 웹 메일 서비스가 사용할 Database를 생성하고 Roundcube에 로그인하고 서비스를 Initialize할 수 있도록 유저를 생성하며 권한을 부여한 뒤 DBMS에 변경된 권한을 적용하겠다.

<br>

```zsh
[root@mail-db ~]# mysql
Welcome to-the MariaDB monitor.  Commands end with ; or \g.
Your Maria-B connection id is 2
Server ver-ion: 5.5.68-MariaDB MariaDB Server

Copyright -c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help-' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(-one)]> create database emailDB;
Query OK, - row affected (0.00 sec)

MariaDB [(-one)]> GRANT ALL ON emailDB.* TO 'emailAdmin'@'localhost' IDENTIFIED BY '1234';
Query OK, - rows affected (0.00 sec)

MariaDB [(-one)]> FLUSH PRIVILEGES;
Query OK, - rows affected (0.00 sec)

MariaDB [(-one)]> exit
Bye
```

<br>

Roundcube -웹 메일 서비스를 설정하기 위해 `http://mail.gitblog.io/roundcube/installer`{: .filepath }웹페이지에-접속하자. 

<br>

Linux01(10-10.10.30)에서 Firefox로 Roundcube 초기 설정을 진행하도록 하겠다.

![Roundcub- 초기설정 웹 페이지](/assets/img/2024-03-21/12.png)
_Roundcube-초기설정 웹 페이지_

<br>

빨간 박스- 있는 항목들이 전부 OK가 되어야 Roundcube 서비스를 DB 서버와 연동하고 시작할 수 있다.

![Roundcub- 초기설정 웹 페이지2](/assets/img/2024-03-21/13.png)
_Roundcube-초기설정 웹 페이지2_

해당 페이-의 최하단으로 스크롤을 내려 이동한 다음 NEXT 버튼을 클릭하자.

<br>

![Roundcub- 설정 웹 페이지1](/assets/img/2024-03-21/14.png)
_Roundcube-설정 웹 페이지1_

<br>

위의 사진 -처럼 product_name을 Web_Mail로 설정하고 스크롤을 내려 Database setup 항목을 수정한다.

<br>

![Roundcub- 설정 웹 페이지2](/assets/img/2024-03-21/15.png)
_Roundcube-설정 웹 페이지2_

<br>

위 사진처-, MariaDB에서 설정해줬던대로 빨간 박스 안의 내용을 수정한 뒤 스크롤을 내려 최하단으로 이동

<br>

![Roundcub- 설정 웹 페이지3](/assets/img/2024-03-21/16.png)
_Roundcube-설정 웹 페이지3_

<br>

빨간 박스 -안의 CREATE CONFIG 버튼을 클릭한다.

<br>

CREATE CON-IG를 눌러 나온 페이지에서 노란 박스안에 내용은 config.inc.php 파일이 `/var/www/html/roundcubem-il-1.3.7/conf/`{: .filepath }에 저장되어야 한다는 의미이다.

![Roundcub- 설치 웹 페이지1](/assets/img/2024-03-21/17.png)
_Roundcube-설치 웹 페이지1_


노란 박스 -안에 잘 보이는 Download 버튼을 눌러 저장해 메일 서버로 옮겨야 한다.



<br>

해당 파일- `/var/www/html/roundcube/config/`{: .filepath }이동시켰다면 스크롤을 조금 내려서 Continue 버튼을 누-다.

<br>

![Roundcub- 설치 웹 페이지2](/assets/img/2024-03-21/18.png)
_Roundcube-설치 웹 페이지2_

<br>

위 사진에- 첫 번째 빨간 박스안의 Check config file의 2가지 항목이 OK인 상태를 확인하고, 두 번째 빨간 박스인 버-을 누른다.

<br>

![Roundcub- 설치 웹 페이지3](/assets/img/2024-03-21/19.png)
_Roundcube-설치 웹 페이지3_

<br>

그럼 위 사-진과 같이 모두 OK인 상태가 될 것이다. 이제 스크롤을 조금 아래로 내린다.

<br>

![Roundcub- Test 웹 페이지1](/assets/img/2024-03-21/20.png)
_Roundcube-Test 웹 페이지1_

<br>

SMTP(Simpl- Mail Transfer Protocol) 25번 포트를 사용하는 메일 전송 프로토콜 테스트로 사전에 설정한 영역이름(gitblog.io)을 뒤로 적어서 아래의 **Send test mail** 버튼을 클릭한다.

메일 서버에 해당 유저가 생성되어야 username을 인식하기 때문에 아래 명령어를 입력해 새로운 유저를 생성한다.

<br>

```zsh
[root@mail-db ~]# useradd testuser
[root@mail-db ~]# passwd testuser
```

<br>

![Roundcub- Test 웹 페이지2](/assets/img/2024-03-21/21.png)
_Roundcube-Test 웹 페이지2_

<br>

아래의 Test IMAP config는 `dovecot` 서비스가 활성화되어 있어야 가능하다.

<br>

이후에 Test IMAP config 창에 방금 만든 사용자와 비밀번호를 입력한다.

<br>

![Roundcube Test 웹 페이지3](/assets/img/2024-03-21/22.png)
_Roundcube Test 웹 페이지3_

<br>

정상적으로 테스트가 끝났다. 위 사진에서 빨간 박스로 강조하는 내용은 `/var/www/html/roundcube/installer`{: .filepath } 디렉토리를 제거하거나, `/var/www/html/roundcube/config/config.inc.php`{: .filepath }에서 enable_installer 옵션이 disable 되어있는지 확인하라는 것이다.

roundcube에서 installer 디렉토리를 삭제하도록 하고, window에서 웹메일 클라이언트로 접속하기 전, firefox로 확인한다.

<br>

![Roundcube 접속 페이지](/assets/img/2024-03-21/23.png)
_Roundcube 접속 페이지_

<br>

![Roundcube 접속 페이지2](/assets/img/2024-03-21/24.png)
_Roundcube 접속 페이지2_

<br>

위 사진처럼 에러가 발생한다면, `/var/www/html/roundcube/logs/error`{: .filepath }를 확인해보자.

<br>

```vim
...
...
[22-Mar-2024 02:58:14 UTC] PHP Warning:  strtotime(): It is not safe to rely on the system's timezone settings. You are *required* to use the date.timezone setting or the date_default_timezone_set() function. In case you used any of those methods and you are still getting this warning, you most likely misspelled the timezone identifier. We selected the timezone 'UTC' for now, but please set date.timezone to select your timezone. in /var/www/html/roundcubemail-1.3.7/program/lib/Roundcube/rcube_session_db.php on line 104
~
~
~
```

<br>

`http://[도메인네임]/roundcube/installer/`{: .filepath } 웹 페이지에서 맨 아래의 NEXT 버튼을 무심코 눌렀을 때 발생할 수 있는 오류이다. 필자는 `dovecot`과 `sendmail` 설정 파일도 제대로 설정하지 않았어서 더 해결하기 쉽지 않았다. 

![http://[도메인네임]/roundcube/installer/ 주의사항](/assets/img/2024-03-21/25.png)
_http://[도메인네임]/roundcube/installer/ 주의사항_

<br>

시스템의 시간대를 믿는 것이 안전하지 않다는 경고메세지이다. 이 에러는 `/etc/php.ini`{: .filepath }에서 date.timezone이라는 변수를 수정해야 한다.

<br>

vim에서 /date.timezone 엔터 입력 후 다음 사진처럼 수정한다.

![/etc/php.ini 수정](/assets/img/2024-03-21/26.png)
_/etc/php.ini 수정_

<br>

`/etc/php.ini`{: .filepath }를 위 사진처럼 수정한 뒤 `dovecot`, `sendmail`, `httpd` 서비스를 다시 시작하니 해결되었다.

<br>

이제 테스트용 유저를 삭제하고 실제 서비스한다고 생각하고 minyeokue라는 유저를 생성하는데, 이 유저는 gitblog.io의 관리자 계정의 메일을 관리할 것이다.

<br>

![CentOS8 주 네임서버 영역파일](/assets/img/2024-03-21/27.png)
_CentOS8 주 네임서버 영역파일_

<br>

SOA 레코드의 빨간 박스는 관리자 이메일이라는 의미이다. => **minyeokue.gitblog.io => minyeokue@gitblog.io**

<br>

> 에러가 발생한 상태에서 생성된 사용자는 Roundcube에 로그인한 순간, 에러를 발생시킬 가능성이 매우 높다. 따라서 testuser를 삭제한다.
{: .prompt-info }

<br>

이제 각 유저를 생성했을 때 마지막으로 진행하는 **신원 편집**을 해야하는데, 먼저 관리자 이메일인 minyeokue 유저로 진행하겠다.

<br>

![관리자 이메일 설정1](/assets/img/2024-03-21/28.png)
_관리자 이메일 설정1_

![관리자 이메일 설정2](/assets/img/2024-03-21/29.png)
_관리자 이메일 설정2_

![관리자 이메일 설정3](/assets/img/2024-03-21/30.png)
_관리자 이메일 설정3_

<br>

메일 서버 구축을 위한 기본 설정이 끝났다.

---

##### DHCP 서버 구축

<br>

곧 진행할 예정..