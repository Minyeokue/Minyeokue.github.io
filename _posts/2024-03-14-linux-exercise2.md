---
title: 리눅스 종합문제 2
author: minyeokue
date: 2024-03-14 20:00:00 +0900
last_modified_at: 2024-03-15 14:00:00 +0900
categories: [Exercise]
tags: [Linux, Window, Network, Secure]

toc: true
toc_sticky: true
---

DNS서버와 Apache 웹서버, DB서버, 프록시서버에 ssl 자체 서명 인증서를 통한 접속과 라운드로빈 실습

---
<br>

## 1. 개요
<br>

#### 1단계

<br>

- CentOS7(GUI)  [192.168.1.10]   => 주 DNS 서버, http 웹서버
- CentOS8       [192.168.1.20]   => 보조 DNS 서버, https 웹서버
- Window 2003   [192.168.1.50]   => 보조 DNS 서버
  
윈도우 서버에서 구현 사항 확인

<br>

#### 2단계

- CentOS7(GUI)  [192.168.1.10]  => HAproxy 서버, MariaDB 클라이언트
- CentOS8-2     [192.168.1.30]  => Apache 웹서버, MariaDB 서버
- CentOS7-2     [192.168.1.40]  => Apache 웹서버, MariaDB 서버

CentOS7에서 HAproxy로 구현한 라운드로빈 방식으로 웹서버와 DB서버 각각 접근해보겠다.

<br>

## 2. 1단계 실습


#### Linux01
---

먼저 CentOS7(이하 Linux01)에서 DNS 서버와 웹서버 패키지를 다운로드

<br>

    [root@Linux01 ~]# yum -y install caching-nameserver httpd

<br>

네임 서버에 관련된 설정(configure; -> conf)을 수정한다

<br>

    [root@Linux01 ~]# vim /etc/named.conf

<br>

![named.conf](/assets/img/2024-03-14/1.png)

<br>

주 서버의 영역을 설정해주기 위해 영역 영역 이름은 gitblog.vm으로 하겠다.

<br>

    [root@Linux01 ~]# vim /etc/named.rfc1912.zones

<br>

![named.rfc1912.zones](/assets/img/2024-03-14/2.png)

<br>

현재 주 DNS 서버에서 보조 DNS 서버로 보내기 위해

- type master;  => 주 영역 DNS 서버임을 의미
- file "gitblog.vm.zone"  => `/var/named/gitblog.vm.zone` 이라는 파일에 영역이 저장될 것이라는 의미 
- allow-transfer { 192.168.1.20; 192.168.1.50; };  => 보조 DNS 서버에 전달하겠다는 의미
<br>

해당 이유로 위 항목들을 설정해주었다.

<br>

    [root@Linux01 ~]# cd /var/named
    [root@Linux01 named]# ls

<br>

ls 명령어로 /var/named 아래에 있는 파일들을 확인하고, 그 중 named.localhost라는 파일의 권한(소유자, 소유그룹)과 함께 복사해서 gitblog.vm.zone이라는 파일을 만들겠다.

<br>

그렇게 하는 이유는 root 소유자와 named 그룹이 소유하지 않으면 해당 영역으로 쿼리를 할 수 없기 때문이다.

<br>

    [root@Linux01 named]# cp -p named.localhost gitblog.vm.zone

<br>

복사한 뒤 수정

<br>

![/var/named/ls-l](/assets/img/2024-03-14/3.png)

<br>

    [root@Linux01 named]# vim gitblog.vm.zone

<br>

![gitblog.vm.zone](/assets/img/2024-03-14/4.png)

<br>

이제 named 서비스를 시작하기 전에 `gitblog.vm` 영역의 `gitblog.vm.zone` 파일을 제대로 작성했는지 확인해본다.

<br>

    [root@Linux01 named]# named-checkzone gitblog.vm gitblog.vm.zone
    zone gitblog.vm/IN: loaded serial 0
    OK

    [root@Linux01 named]# systemctl enable --now named

<br>

`systemctl enable --now named` 라는 명령어로 `systemctl start named && systemctl enable named` 와 같은 효과를 볼 수 있다.

<br>

이제 Linux01에 웹서버가 잘 작동할 수 있도록 html 파일을 생성하고, 가상호스트를 구현해보겠다.
<br>

*가상호스트란 하나의 서버에 여러 대의 웹서버를 구현하는 것과 비슷한 효과를 볼 수 있다.*

<br>

먼저 /(최상위 디렉토리)로 이동한다.

<br>

    [root@Linux01 /]# mkdir /www1 /www2
    [root@Linux01 /]# echo www1.gitblog.vm site > www1/index.html
    [root@Linux01 /]# echo www2.gitblog.vm site > www2/start.html

<br>

이제 가상 호스트의 홈디렉토리와 이름을 지정하기 위해 설정 파일을 수정해보자

<br>

    [root@Linux01 /]# vim /etc/httpd/conf/httpd.conf

<br>

CentOS7 기준 다음 줄들을 수정한다.

<br>

    ...
    ...
    124 <Directory "/">
    125     AllowOverride None
    126     # Allow open access:
    127     Require all granted
    128 </Directory>
    ...
    ...
    163 <IfModule dir_module>
    164     DirectoryIndex index.html start.html
    165 </IfModule>
    ...
    ...
    355 <VirtualHost *:80>
    356     DocumentRoot /www1
    357     ServerName www1.gitblog.vm
    358 </VirtualHost>
    359 <VirtualHost *:80>
    360     DocumentRoot /www2
    361     ServerName www2.gitblog.vm
    362 </VirtualHost>

    :wq

<br>

이렇게 수정한 뒤 문법에 문제가 없는지 체크해보자

<br>

    [root@Linux01 ~]# httpd -t
    ...
    ...
    Syntax OK

<br>

문제가 없다면 구문 OK 라고 나올 것이다.

<br>

    [root@Linux01 ~]# systemctl enable --now httpd

<br>

`systemctl` 명령어로 http 데몬 서비스를 활성화한다.

<br>

<br>

#### CentOS8
---

보조 DNS 서버와 웹 서버를 구축하기 위해 필요한 패키지를 설치한다.

<br>

    [root@centos8 ~]# yum -y install caching-nameserver httpd

<br>

CentOS7(Linux01)에서 했던 것 처럼 `/var/named.conf` 파일을 수정한다. => any; any;

<br>

하지만 보조 영역을 위한 `/var/named.rfc1912.zones` 파일은 다르게 수정한다.

<br>

![보조-named.rfc1912.zones](/assets/img/2024-03-14/5.png)

<br>

위와 같이 수정한 뒤 named 서비스를 시작하면 `/var/named/slaves` 아래에 지정한 이름으로 읽기전용 영역 파일이 생성된다.

<br>

*`/var/named/slaves/gitblog.vm.slave.zone` 파일은 암호화되어 저장되기 때문에 자세히 알아볼 수 없다.*

<br>

![gitblog.vm.slave.zone](/assets/img/2024-03-14/6.png)

<br>

이제 2번째 웹서버를 구축한다. https로 구현하도록하고 먼저 `/var/www/html/index.html` 파일을 만들겠다.

<br>

    [root@centos8 ~]# cd /var/www/html
    [root@centos8 html]# echo ssl.gitblog.vm site > index.html

<br>

SSL(Secure Socket Layer) 인증으로 자체 서명한 인증서를 만들기 위해 관련 패키지를 설치한다.

<br>

    [root@centos8 ~]# yum -y install mod_ssl

<br>

설치한 뒤 /etc/ssl/ 폴더로 이동해 private라는 폴더를 생성해 그 안에 인증서 파일과 개인키를 생성하도록 하겠다.

<br>

    [root@centos8 ~]# cd /etc/ssl
    [root@centos8 ssl]# mkdir private
    [root@centos8 ssl]# cd private

<br>

openssl 명령어로 개인키와 인증서를 생성한다.

<br>

    [root@centos8 private]# openssl req -x509 -nodes -newkey rsa:2048 -keyout gitblog.vm.key -out gitblog.vm.crt

<br>

rsa 알고리즘으로 2048비트 암호화를 실행해 **gitblog.vm.key**라는 개인키와 **gitblog.vm.crt**라는 인증서 파일을 생성한다.

<br>

![openssl 생성](/assets/img/2024-03-14/7.png)

<br>

https 자체 서명 인증서가 작성되었다.

<br>

이제 80번 포트(http)로 들어와도 443 포트(https)로 들어오도록 `/etc/httpd/conf/httpd.conf` 파일을 수정한다.

<br>

    [root@centos8 ~]# vim /etc/httpd/conf/httpd.conf

    ...
    ...
    358 <VirtualHost *:80>
    359     ServerName ssl.gitblog.vm
    360     ServerAlias gitblog.vm
    361     Redirect permanent / https://ssl.gitblog.vm/
    362 </VirtualHost>
    :wq

<br>

제일 아래 줄에 위 처럼 작성한다.

<br>

이제 ssl 관련 설정 파일 `/etc/httpd/conf.d/ssl.conf` 파일을 수정하도록 한다.

<br>

    [root@centos8 ~]# vim /etc/httpd/conf.d/ssl.conf

    ...
    ...
    203
    204 <VirtualHost *:443>
    205     ServerAdmin admin@gitblog.vm
    206     ServerName ssl.gitblog.vm
    207     ServerAlias gitblog.vm
    208     DocumentRoot /var/www/html
    209     SSLEngine on
    210     SSLCertificateFile /etc/ssl/private/gitblog.vm.crt
    211     SSLCertificateKeyFile /etc/ssl/private/gitblog.vm.key
    212 </VirtualHost>
    :wq

<br>

저장한 뒤 `httpd -t` 명령으로 문법을 체크하면 에러가 발생하는데 이 이유는 ssl 가상 호스트의 개인키와 인증서 파일의 위치를 모르기 때문이다.

<br>

`systemctl start httpd` 명령으로 에러가 없다면 정상 작동할 것이다.

<br>

    [root@centos8 ~]# systemctl start httpd
    [root@centos8 ~]# systemctl enable httpd

<br>
<br>

#### window 2003
---

IP 설정

<br>

![window IP 설정](/assets/img/2024-03-14/8.png)

<br>

DNS 관리창 위치

<br>

![window DNS 설정 위치](/assets/img/2024-03-14/9.png)

<br>

영역 추가

<br>

![DNS 정방향 영역 추가](/assets/img/2024-03-14/10.png)

<br>

다음 -> 보조 영역 체크 -> `gitblog.vm` 영역 이름 설정 -> 주 영역 IP 주소 추가 -> 다음 -> 마침

<br>

잠시 기다리면 주 DNS 서버로부터 영역이 로드된다.

<br>

불러와진 정방향 조회 영역

<br>

![window DNS  gitblog.vm](/assets/img/2024-03-14/11.png)

<br>


이제 internet explorer로 확인해보자

<br>

이 때 주의할 점은 만약 명령 프롬프트로 `ping` 명령어를 통해 dns 정보가 기록된 상태라면 `ipconfig /flushdns` 명령어를 입력해 dns 정보를 지운 뒤 확인하는 것이 좋다.

<br>

dns 정보를 확인하고 싶을 때 `ipconfig /displaydns` 명령어로 cmd에서 확인할 수 있다.

<br>

*리눅스에서 dns 정보를 확인할 수 있는 파일은 `/etc/resolv.conf` 파일로, 해당 파일을 바꾸면 dns 서버를 즉각적으로 바꿀 수 있다.*

<br>

![www1.gitblog.vm](/assets/img/2024-03-14/12.png)

<br>

![www2.gitblog.vm](/assets/img/2024-03-14/13.png)

<br>

ssl은 explorer에서 확인할 수 없으니 firefox를 사용하겠다.

<br>

리눅스에서 확인할 때 네임서버 설정을 해주지 않으면 위치를 알 수 없기 때문에, `/etc/resolv.conf` 파일에서 네임서버의 ip를 설정해주자.

<br>

![/etc/resolve.conf](/assets/img/2024-03-14/14.png)

<br>

![ssl.gitblog.vm main](/assets/img/2024-03-14/15.png)

<br>

고급.. -> 위험을 감수하고 계속

<br>

![ssl.gitblog.vm site](/assets/img/2024-03-14/16.png)

<br>

## 3. 2단계 실습

곧 작성하도록 하겠다.
