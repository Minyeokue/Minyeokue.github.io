---
title: 리눅스 수업 정리
author: minyeokue
date: 2024-03-08 18:00:00 +0900
last_modified_at: 2024-03-19 21:20:54 +0900
categories: [Linux Class]
tags: [Linux, Network, Permission]
render_with_liquid: false

toc: true
toc_sticky: true
---

리눅스 패스워드 저장 위치 및 그룹 설정과 권한, **wheel 그룹**, **가상호스트**와 DNS 설정에 대한 학습
<br>

---

## cat 명령어를 사용해 /etc/shadow 파일을 확인해보자.

<br>

먼저 cat 명령어가 어디에 있는지 확인해야 한다.

<br>

```zsh
[root@localhost ~]# which cat
/usr/bin/cat
[root@localhost ~]# ls -l /usr/bin/cat
-rwxr-xr-x. 1 min users 68568 Jan  2 13:54 /usr/bin/cat
```

<br>

현재 사용자의 권한은 755로 소유자인 min은 모든 권한, users 그룹은 읽고 실행하는 권한, 다른 사용자들은 읽고 실행하는 권한을 가지고 있다.

<br>

cat 명령어에 특수 권한을 부여해 일반사용자가 /etc/shadow를 확인할 수 있다면 보안에 정말 치명적일 것이다. 연습해보며 조심하자.

<br>

```zsh
[root@localhost ~]# chmod 4775 /usr/bin/cat
```

<br>

chmod 명령어로 /usr/bin/cat 명령의 소유자인 root의 권한으로 실행하도록 할 수 있다.

<br>

일반적으로는 chmod 755 [파일 또는 디렉토리명]이라는 명령어를 입력한다.

<br>

이 때 문제는 cat 명령어를 통해 일반적인 상황에서는 확인할 수 없는 /etc/shadow 파일 등을 보려면 특수한 권한이 필요한데,

<br>

권한에 대한 숫자를 지정하는 방법과, 특정 문자로 심볼릭하게 권한을 부여하는 방법이 있다.

<br>

```zsh
[root@localhost ~]# chmod u+s /usr/bin/cat
```
<br>

numeric한 방법인 4755, symbolic한 방법인 u+s가 있고 효과는 동일하다.

<br>

이제 cat 명령으로 원래는 볼 수 없었던 /etc/shadow 등의 파일을 확인할 수 있다.

<br>

## **setUID, setGID**

<br>

|numeric|symbolic|효과|결과|
|:---:|:---:|:---:|:---:|
|chmod 4755 [파일명]|chmod u+s [파일명]|소유자 권한으로 해당 파일(바이너리 명령)을 실행|-rwsr-xr-x.|
|chmod 2755 [파일명]|chmod g+s [파일명]|그룹 권한으로 파일을 실행|-rwxr-sr-x.|

<br>

setGID의 경우 setUID에 비해 잘 사용되지 않는다..

<br>

## 주 그룹과 부 그룹

<br>

    [root@localhost ~]# usermod -g [그룹명] [사용자명]

    # 그룹명 혹은 번호를 지정해 사용자의 주 그룹을 변경하는 명령어이다.

    [root@localhost ~]# usermod -G [그룹명] [사용자명]

<br>

그룹명 혹은 번호를 지정해 사용자의 부 그룹을 추가하는 명령어이다. 해당 사용자의 부 그룹을 한꺼번에 추가하고 싶다면 **","** 로 구분한다.

<br>

e.g. [root@localhost ~]# usermod -G wheel,games kk1

<br>

kk1 유저의 부 그룹으로 wheel과 games 그룹을 추가한다.

<br>

## 특정 사용자(wheel 그룹의 사용자들)만 su 명령어 사용

<br>

    [root@localhost ~]# vim /etc/pam.d/su

```vim
...
#auth       required    pam_wheel.so    use_uid
...
```

<br>

해당 부분의 주석을 해제한다. 그러면 su 명령어는 wheel 그룹에 속한 사용자들만 사용할 수 있게된다.

<br>

root를 제외한 다른 그룹이 su 명령을 소유하더라도 **/etc/pam.d/su**를 통해 위의 방법으로 수정한다면 root와 wheel 그룹만 su 명령을 사용할 수 있게 된다.

<br>

    [root@localhost ~]# vim /etc/group

    ...
    ...
    wheel:x:10:root,hong,kim
    ...
    ...

<br>

그룹 파일을 수정하는 방법으로도 그룹에 추가할 수 있다.

<br>

`usermod -G wheel [사용자명]` 으로도 추가할 수 있다.

<br>

이 방법은 해당 사용자의 부 그룹으로 추가하는 방법이고, 주 그룹을 변경하는 방법은 옵션에 소문자 g를 사용하는 방법이다.

<br>

    [root@localhost ~]# usermod -g wheel kk1

<br>

다른 방법으로는 su 명령어에 특수권한을 부여하는 방법이 있다.

<br>

```zsh
[root@localhost ~]# which su
/bin/su
[root@localhost ~]# ll /bin/su
-rwsr-xr-x. 1  root root 32096 Apr 14 05:43 /bin/su
```

<br>

위의 권한을 갖고 있다면

<br>

```zsh
[root@localhost ~]# chgrp wheel /bin/su
[root@localhost ~]# ll /bin/su
-rwxr-xr-x. 1  root wheel 32096 Apr 14 05:43 /bin/su
```

<br>

chgrp 명령어로 사용 그룹 권한을 변경할 수 있지만. **-rwsr-xr-x에서 -rwxr-xr-x로 바뀌게 된다.**

<br>

su 명령을 소유주 권한(root)로 실행할 수 없게 된다. 다시 특수권한을 부여해줘야 한다.

<br>

```zsh
[root@localhost ~]# chmod 4750 /bin/su
[root@localhost ~]# ll /bin/su
-rwsr-xr-x. 1  root wheel 32096 Apr 14 05:43 /bin/su
```

<br>

위의 명령 "chmod 4750 /bin/su"와 같은 효과를 내는 명령은 다음과 같다.

<br>

**chmod u+s /bin/su**

<br>

### **포트 번호를 확인하는 명령**

<br>

    [root@localhost ~]# netstat -nlp | grep [원하는 서비스]

<br>

혹은 다음의 명령을 사용한다.

<br>

    [root@localhost ~]# ss -ant

<br>

## PAM을 사용한 ssh 보안 강화

<br>

PAM은 Pluggable Authentication Module;착탈형 인증 모듈

<br>

사용자를 인증하고 그 사용자의 서비스에 대한 엑세스를 제어하는 모듈화된 방법을 일컫는다.

<br>

해당 모듈이 모인 폴더가 `/etc/pam.d/`이고 이 아래에 su 파일을 조작한다.

<br>

/etc/ssh/sshd_config 에서 root로의 접근을 막고, ssh 포트 번호를 변경한다.

<br>

-> ssh 설정 변경 후에는 systemctl restart sshd

<br>

그 이후 /etc/pam.d/su 에서 주석 해제해 활성화.

<br>

## 가상 호스트 방식 (Virtual Host)

<br>

하나의 IP와 80번 포트로 여러 개의 웹사이트를 운영할 수 있는 기술이다.

<br>

    시나리오

        - 가상 도메인 : kosa.vm
        - 네임 서버 : ns1.kosa.vm

        - IP : 192.168.1.20/24
        - GW : 192.168.1.2
        - DNS1 : 192.168.1.20
        - DNS2 : 8.8.8.8    

        - 호스트 생성
        -> www1.kosa.vm    => 홈 디렉토리 /www1/index.html => www1 site
        -> www2.kosa.vm    => 홈 디렉토리 /www2/start.html => www2 site


가상호스트를 구현해서 사이트를 구성하자..

```zsh
[root@localhost ~]# yum -y install caching-nameserver
[root@localhost ~]# vim /etc/named.conf
```

```vim
options {
    listen-on port 53 { any; };
    listen-on-v6 port 53 { ::1; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file   "/var/named/data/named.secroots";
    recursing-file  "/var/named/data/named.recursing";
    allow-query     { any; };
    ...
}
```

<br>

```zsh
[root@localhost ~]# vim /etc/named.rfg1912.zones  
```

```vim
...
zone "kosa.vm" IN {
    type master;
    file "kosa.vm.zone";
    allow-update { none; };
};

:wq
```

<br>

=> 도메인 영역 생성

<br>

**주의사항으로는 zone 파일의 소유권이 root.named 로 유지되어야 정상적으로 작동할 수 있다.**

<br>

=> 존 파일 생성을 /var/named 디렉토리 안에서 진행

<br>

/var/named 안에 파일 중 named.localhost 라는 파일을 복사해 우리가 만든 사이트 **kosa.vm**의 zone 파일을 만드는데,

<br>

**cp 명령어에 -p 옵션으로 소유자와 소유그룹에 대한 정보까지 함께 복사해야 한다**

<br>

다음 명령을 수행

```zsh
[root@localhost ~]# cd /var/named
[root@localhost named]# cp -p named.localhost kosa.vm.zone
```