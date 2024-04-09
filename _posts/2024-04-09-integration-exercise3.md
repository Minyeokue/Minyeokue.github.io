---
title: 통합 실습 3 - NAT 포트포워딩을 통한 MariaDB, DNS, IIS, FTP 및 원격 데스크톱 연결
excerpt: "윈도우 2003 서버에서 NAT 포트포워딩을 진행해 사설망 안의 MariaDB, DNS, IIS, FTP 서버들에 접속하고 원격 데스크톱으로 연결한다."
author: minyeokue
date: 2024-04-09 19:08:51 +0900
last_modified_at: 2024-04-09 21:35:00 +0900
categories: [Exercise]
tags: [Linux, Windows, Firewall, MariaDB, Secure, Network, DNS]

toc: true
toc_sticky: true
---

<br>

윈도우 2003 서버에서 NAT 포트포워딩을 진행해 사설망 안의 MariaDB, DNS, IIS, FTP 서버들에 접속하고 원격 데스크톱으로 연결한다.

<br>

---

<br>

## 시나리오

<br>

- Win2003

    - [NAT 서버 설정](#nat-서버-설정)

        - 사설망 게이트웨이 : 10.10.10.253/24

        - 공인망 : 192.168.1.99  GW : 192.168.1.2  DNS : 8.8.8.8

            - [NAT 서버 포트포워딩](#nat-서버-포트포워딩) => [MariaDB](#mariadb-서버-테스트), [DNS](#dns-서버-테스트), [IIS](#윈도우-iis-서버-테스트), [FTP](#ftp-서버-테스트) 서버 접근 및 [원격 데스크톱 연결](#원격-데스크톱-연결-테스트)
            
- CentOS8

    - IP : 10.10.10.10/24  GW : 10.10.10.253

    - [DNS 서버 설정](#dns-서버-설정)

        - 도메인명 : minyeokue.gitblog

        - 호스트 생성 : www.minyeokue.gitblog
    
    - [MariaDB 서버 설정](#mariadb-서버-설정)

        - 로컬 DB 접속 시 root 계정 비밀번호 : 1234

        - 원격 DB 접속 시 root 계정 비밀번호 : 4321

        - 데이터베이스 생성 : gitblog

- DC1

    - IP : 10.10.10.20/24  GW : 10.10.10.253

    - [윈도우 IIS 서버 설정](#윈도우-iis-서버-설정)

    - [FTP 서버 설정](#ftp-서버-설정)

        - FTP 서버 홈 디렉토리 : C:\ftpdata

        - blogAdmin 사용자 생성 => FTP 서버 읽기, 쓰기 권한 부여

        - bloguser 사용자 생성 => FTP 서버 읽기 권한 부여
    
    - [원격 데스크톱 연결 설정](#원격-데스크톱-연결-설정)

        - 포트번호 3389에서 35000으로 변경

        - 방화벽 인바운드 규칙 생성

---

<br>

### NAT 서버 설정

<br>

먼저 공인망과 사설망 IP가 구분되어 있기 때문에, 랜카드를 하나 더 붙여준다.

![NAT 서버 랜카드 추가 1](/assets/img/2024-04-09/1.png)
_NAT 서버 랜카드 추가 1_

![NAT 서버 랜카드 추가 완료](/assets/img/2024-04-09/2.png)
_NAT 서버 랜카드 추가 완료_

위 사진처럼 *Add...* 버튼을 누른 뒤 *Network Adapter*를 선택하고 **Finish** 버튼을 누른다.

<br>

가상머신을 실행시키고 네트워크 설정을 변경한다.

<br>

![NAT 서버 IP 설정 1](/assets/img/2024-04-09/3.png)
_NAT 서버 IP 설정 1_

*내 네트워크 환경*을 우측 마우스 클릭 -> **속성** 메뉴를 선택해 *LAN 또는 고속 인터넷* 창을 연다.

<br>

먼저 ~~가상의~~ 공인망 IP를 설정한다. 

![NAT 서버 IP 설정 2](/assets/img/2024-04-09/4.png)
_NAT 서버 IP 설정 2_

로컬 영역 설정을 *더블 클릭* 혹은 *오른쪽 마우스 클릭* -> **속성** -> *인터넷 프로토콜(TCP/IP)* 더블 클릭

<br>

![NAT 서버 공인망 IP 설정](/assets/img/2024-04-09/5.png)
_NAT 서버 공인망 IP 설정_

위 사진처럼 설정해둔 뒤 **확인** -> **확인** 버튼을 연달아서 누른다.

<br>

![NAT 서버 IP 설정 3](/assets/img/2024-04-09/6.png)
_NAT 서버 IP 설정 3_

새로 붙인 랜카드에 대한 설정을 진행한다. 사설망의 게이트웨이로 설정할 것이다.

<br>

![NAT 서버 사설망 게이트웨이 설정](/assets/img/2024-04-09/7.png)
_NAT 서버 사설망 게이트웨이 설정_

위 사진처럼 설정하고 **확인** 버튼을 누르면 DNS 목록이 비어있어 기본 DNS 서버 주소로 설정된다는 알림창이 팝업된다. **확인** -> **확인** 버튼을 연달아서 누르면 된다.

<br>

IP 설정은 끝났지만, 알아보기 쉽도록 각각의 로컬 영역 연결의 이름을 "공인망", "사설망 게이트웨이"로 변경한다.

![로컬 영역 연결 이름 변경](/assets/img/2024-04-09/8.png)
_로컬 영역 연결 이름 변경_

<br>

NAT 서버의 역할을 할 수 있도록 기능을 추가하기 위해 *사용자 서버 관리* 창으로 향한다. 해당 메뉴의 위치는 다음 사진을 참조한다.

![사용자 서버 관리 위치](/assets/img/2024-04-09/9.png)
_사용자 서버 관리 위치_

<br>

![사용자 서버 관리 역할 추가](/assets/img/2024-04-09/10.png)
_사용자 서버 관리 역할 추가_

위 사진처럼 강조된 부분을 클릭해 *서버 구성 마법사*를 진행한다.

<br>

![서버 구성 마법사 1](/assets/img/2024-04-09/11.png)
_서버 구성 마법사 1_

**다음** 버튼을 누른다.

<br>

![서버 구성 마법사 2](/assets/img/2024-04-09/12.png)
_서버 구성 마법사 2_

NAT 및 일종의 라우팅 역할을 하게 하기 위해 *원격 액세스 및 VPN 서버*를 선택한 뒤, **다음** -> **다음** 버튼을 누른다.

<br>

![라우팅 및 원격 액세스 서버 설치 마법사 1](/assets/img/2024-04-09/13.png)
_라우팅 및 원격 액세스 서버 설치 마법사 1_

*라우팅 및 원격 액세스 서버 설치 마법사* 창이 팝업되며, **다음** 버튼을 눌러서 계속 진행한다.

<br>

![라우팅 및 원격 액세스 서버 설치 마법사 2](/assets/img/2024-04-09/14.png)
_라우팅 및 원격 액세스 서버 설치 마법사 2_

*네트워크 주소 변환(NAT)* 라디오 버튼을 선택한 뒤, **다음** 버튼을 누른다.

<br>

![라우팅 및 원격 액세스 서버 설치 마법사 3](/assets/img/2024-04-09/15.png)
_라우팅 및 원격 액세스 서버 설치 마법사 3_

*~~가상의~~ 공인망*을 선택한 뒤, **다음** 버튼을 누른다.

![라우팅 및 원격 액세스 서버 설치 마법사 완료](/assets/img/2024-04-09/16.png)
_라우팅 및 원격 액세스 서버 설치 마법사 완료_

<br>

![서버 구성 마법사 완료](/assets/img/2024-04-09/17.png)
_서버 구성 마법사 완료_

*라우팅 및 원격 액세스 서버 설치 마법사* 창에서 **마침** 버튼을 누른 뒤, 잠시 기다리면 *서버 구성 마법사*에서도 **마침** 버튼을 누를 수 있게 된다.

<br>

이제 NAT 서버 구성을 마쳤으니 사설망(10.10.10.0/24) 대역에서 인터넷 연결이 가능해진 상태이다.

<br>

---

<br>

### DNS 서버 설정

<br>

먼저 CentOS8의 IP 및 호스트 이름의 설정을 변경하겠다.

`nmtui` 명령어를 입력한다.

![Network Manager Text User Interface](/assets/img/2024-04-09/18.gif)
_Network Manager Text User Interface_

리눅스 서버의 IP 및 호스트 이름 설정을 변경했다.

네트워크 랜카드를 재활성화하는 명령어로는 `nmcli con up [랜카드 이름]` 명령어로 변경한 IP를 적용시킬 수 있다.

> 호스트 이름을 변경하는 다른 방법은 `hostnamectl set-hostname [변경할 이름]` 명령어를 입력한 뒤, `exit` 명령어로 다시 로그인하면 적용된다.
{: .prompt-tip }

<br>

NAT 서버의 게이트웨이를 통해 외부 인터넷과 통신이 가능한지 확인한다.

![NAT 게이트웨이를 통한 외부 인터넷 연결 테스트](/assets/img/2024-04-09/19.gif)
_NAT 게이트웨이를 통한 외부 인터넷 연결 테스트_

정상적으로 진행되니, 이제 DNS 서버를 구축하기 위한 패키지를 설치해보겠다.

<br>

`yum -y install caching-nameserver vim` 명령어를 입력한다.

DNS 서버 관련 패키지들과 vim 패키지가 설치될 것이다.

DNS 서버 설정 파일을 수정해주도록 한다. `vim /etc/named.conf` 명령어를 입력한다.

![/etc/named.conf 수정](/assets/img/2024-04-09/20.png)
_/etc/named.conf 수정_

<br>

`/etc/named.rfc1912.zones`{: .filepath } 파일을 수정해 영역 파일에 관련된 설정을 진행한다.

![/etc/named.rfc1912.zones 수정](/assets/img/2024-04-09/21.png)
_/etc/named.rfc1912.zones 수정_

<br>

영역 파일을 복사해서 수정할 것이다. 기본이 되는 영역 파일은 `/var/named/named.localhost`{: .filepath }으로 이 파일의 권한과 함께 복사해서 `minyeokue.gitblog.zone`{: .filepath } 파일을 만들 것이다.

![minyeokue.gitblog.zone 파일 생성](/assets/img/2024-04-09/22.png)
_minyeokue.gitblog.zone 파일 생성_

<br>

![minyeokue.gitblog.zone 파일 수정](/assets/img/2024-04-09/23.png)
_minyeokue.gitblog.zone 파일 수정_

DNS 서버는 사설망인 10.10.10.10/24에 있지만, 공인망 IP를 적은 이유는 **SNAT(Source Network Address Translation)**을 하기 위함이다.

> **SNAT(Source NAT)**는 사설망에 존재하는 서버의 주소를 NAT의 공인망 주소로 변경하는 것을 뜻하며, 외부에서는 사설망의 존재를 모르고 공인망 주소를 해당 서비스를 하는 서버로 인식하기 때문에 SNAT를 사용한다. 또한 **DNAT(Destination Network Address Translation)**와 함께 적용되어야 외부에서의 응답을 사설망의 서버로 보낼 수 있다. 그렇기 때문에 **DNAT**와 포트포워딩이 거의 같은 의미로 통한다.
{: .prompt-info }

<br>

영역 파일 설정이 완료되면, `systemctl enable --now named` 명령어를 입력해 시스템이 재부팅되어도 서비스를 자동으로 시작할 수 있도록 설정한다.

서비스를 재시작하고 싶다면 `systemctl restart [서비스명]` 명령어를 입력한다.

정상적으로 링크파일이 생성되었다면 DNS 설정은 끝나게 된다.

<br>

### MariaDB 서버 설정

<br>

MariaDB 서버 구축을 위한 패키지를 설치한다. `yum -y install mariadb-server` 명령어를 입력한다.

![MariaDB 설정 데이터베이스 확인](/assets/img/2024-04-09/24.png)
_MariaDB 설정 데이터베이스 확인_

현재 MariaDB의 관리 데이터베이스를 확인했다.

<br>

> MariaDB 기본 보안 설정은 `mysql_secure_installation` 명령어를 통해 진행할 수 있으며, 이후 계정명과 비밀번호을 입력해야 접속할 수 있다.
{: .prompt-info }

"gitblog" 데이터베이스를 생성하고, 내부 테스트를 위해 MariaDB 서버에 접속할 때 root 계정으로 암호 '1234'를 입력하고, 외부에서 root 계정으로 접속할 때 암호 '4321'을 입력하도록 설정한다.

권한은 "gitblog"의 모든 테이블로 한정한다.

![MariaDB 데이터 베이스 생성 및 접속 설정](/assets/img/2024-04-09/25.png)
_MariaDB 데이터 베이스 생성 및 접속 설정_

`FLUSH PRIVILEGES;` SQL문으로 변경된 권한 설정을 적용한다.

~~root 계정으로의 접속은 보안 관점에서 봤을 때 바람직하지 않지만, 실습 테스트를 위해 진행한다.~~

<br>

CentOS8에서 진행해야 할 설정은 끝났다.

<br>

### 윈도우 IIS 서버 설정

<br>

이 이후는 추후 작성한다.

<br>

### FTP 서버 설정

<br>

### 원격 데스크톱 설정

<br>

### NAT 서버 포트포워딩

<br>

### 테스트

<br>

#### MariaDB 서버 테스트

<br>

#### DNS 서버 테스트

<br>

#### 윈도우 IIS 서버 테스트

<br>

#### FTP 서버 테스트

<br>

#### 원격 데스크톱 연결 테스트

<br>