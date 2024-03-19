---
title: 리눅스 종합문제 2-1
author: minyeokue
date: 2024-03-15 14:36:11 +0900
last_modified_at: 2024-03-15 18:00:00 +0900
categories: [Exercise]
tags: [Linux, Window, Network, Secure]

toc: true
toc_sticky: true
---

보안 설계를 위한 방화벽 컴퓨터 실습

---

## 개요

+ 사설 IP : 10.10.10.0/24

+ 가상의 공인 IP : 192.168.1.0/24

- 사설망
<br>

  + CentOS8
    - IP  10.10.10.10/24  GW  10.10.10.1  DNS  8.8.8.8
<br>

  + CentOS8 - 2 (CentOS8의 클론)
    - IP  10.10.10.20/24  GW  10.10.10.1  DNS  8.8.8.8
<br>

- 방화벽 컴퓨터(CentOS7)
<br>

  + 사설망 GW  10.10.10.1/24
<br>

  + 공인 IP : 192.168.1.10/24
<br>

- 외부 인터넷(Win2003-1) --> 가상의 공인 IP
<br>

  + IP  192.168.1.50/24

<br>

---

## 환경 구축

<br>

### 1단계
---

#### CentOS7 IP 설정

CentOS7에 랜카드를 하나 추가한다.

NAT 추가
> 이미지 삽입

<br>

로그인 한 후 `nmtui` 명령어를 입력해 이더넷을 추가한다.

```zsh
[root@Linux01 ~]# nmtui
```
<br>

> 이미지 삽입
> 이미지 삽입
> 이미지 삽입

*`nmcli con up ensXXX`를 이더넷 수 만큼 반복해 입력하는 것이 정석적이지만, CentOS7에서는 `systemctl restart network` 명령어를 통해 한 번에 수행할 수 있다.*

```zsh
[root@Linux01 ~]# systemctl restart network
```
<br>

무리없이 진행된다면 다음으로 넘어간다.
<br>


#### CentOS8 IP 설정

<br>

CentOS8으로 로그인해서 `nmtui`를 입력하고 IP 10.10.10.10/24, GW 10.10.10.1, DNS 8.8.8.8 입력

<br>

```zsh
[root@centos8 ~]# nmtui
```
<br>

> 이미지 삽입

<br>

#### CentOS8 - 2 IP 설정

CentOS8으로 로그인해서 `nmtui`를 입력하고 IP 10.10.10.10/24, GW 10.10.10.1, DNS 8.8.8.8 입력

```zsh
[root@centos8 ~]# nmtui
```

> 이미지 삽입

<br>

#### ping으로 통신 확인

CentOS8, CentOS8-2

> 이미지 삽입

<br>

CentOS7이 통신 가능한 이유는 사설망 게이트웨이뿐 아니라 외부 네트워크와 통신이 가능한 NAT가 존재하기 때문이다.
<br>

`SNAT(Source NAT)` -> 외부 네트워크에서는 통신 요청을 방화벽 서버에서 한 줄 알기 때문에(NAT의 변환으로 인해), 외부 네트워크 에서 응답을 네트워크 방화벽 서버로 보내고, 다시 사설 IP로 변환해 실제 통신에 응답한 시스템에 전송한다.

<br>

`DNAT(Destination NAT)` -> 외부에서 사설 IP로 접속을 시도하면 방화벽 컴퓨터(네트워크 방화벽)으로 넘어오고, 사설 IP로 접속하기 위해 방화벽 컴퓨터의 공인 IP를 접속할 사설 IP로 변경하는 것이다.

<br>


결론적으로 사설 네트워크의 모든 컴퓨터가 외부 인터넷을 사용할 때는 방화벽의 공인 IP를 사용하게 되는데, 이것을 **마스커레이딩(Masquerading)**이라고 한다.


*마스커레이딩*을 하기 위해 방화벽을 활성화해보자.

```zsh
[root@centos8 ~] systemctl enable --now firewalld
[root@centos8 ~] firewall-config
```

> 이미지 삽입
> 이미지 삽입
> 이미지 삽입

마스커레이딩 영역 체크

<br>

이후에 CentOS8, CentOS8-2에서 인터넷이 가능해지고, 이제 CentOS8 서버들에 httpd 패키지를 설치한다.

<br>


