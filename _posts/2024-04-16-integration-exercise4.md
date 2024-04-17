---
title: 윈도우 서버 실습 1 - AD Certificate Services를 활용한 보안 웹 및 도메인 그룹 정책, VPN, 공유 폴더 등 실습
excerpt: Window Server Active Directory Certificate Services를 활용해 보안 웹 인증서를 발급받고, Default Domain Policy, Site to Site VPN, RAID, 원격 데스크톱 연결을 실습한다.
author: minyeokue
date: 2024-04-16 16:40:57 +0900
last_modified_at: 2024-04-17 17:43:45 +0900
categories: [Exercise]
tags: [Windows, Firewall, Network, Policy, RAID, VPN, Active Directory]

toc: true
toc_sticky: true
---

<br>

Window Server Active Directory Certificate Services를 활용해 보안 웹 인증서를 발급받고, Default Domain Policy, Site to Site VPN, RAID, 원격 데스크톱 연결을 실습한다.

<br>

---

## 시나리오

<br>

- DC1 => Active Directory Domain Controller

    - [DC1 서버 설정](#dc1-서버-설정)

        - [Domain Group 설정](#domain-group-설정)

            - [Organization Unit 설정](#organization-unit-설정)

        - [Domain Policy 설정](#domain-policy-설정)

        - [공유폴더 설정](#공유폴더-설정)

        - [Window Server 방화벽 설정](#window-server-방화벽-설정)

        - [원격 데스크톱 설정](#원격-데스크톱-설정)

        - [RAID 설정](#raid-설정)

        - [보안 웹 서버 설정](#보안-웹-서버-설정)

        - SVR1의 VPN 클라이언트
        
- [Site to Site VPN 설정](#site-to-site-vpn-설정)

    - SVR1

        - [SVR1 VPN 서버 설정](#svr1-vpn-서버-설정)

            - DC1과 연결된 VPN 서버
    
    - SVR2

        - [SVR2 VPN 서버 설정](#svr2-vpn-서버-설정)

            - Window 2003 서버와 연결된 VPN 서버

- 테스트

    - [Site to Site VPN 테스트](#site-to-site-vpn-테스트)

    - [Domain Policy 테스트](#domain-policy-테스트)

    - [공유폴더 테스트](#공유폴더-테스트)

    - [원격 데스크톱 연결 테스트](#원격-데스크톱-연결-테스트)

    - [RAID 및 보안 웹 서버 테스트](#raid-및-웹-서버-테스트)

<br>

---

### DC1 서버 설정

<br>

AD DS(Active Directory Domain Service)의 설치는 이 블로그의 [통합 실습 2](https://minyeokue.github.io/posts/integration-exercise2/)를 참고하도록 한다.

<br>

위 링크에서 진행한 AD DS가 설치된 상태에서 이 도메인 컨트롤러를 인증 기관으로써 활용하고, 보안 웹 서버에서 인증서 발급 요청을 할 수 있도록 **AD CS(Active Directory Certificate Services)**를 설치한다.

![역할 및 기능 추가 위치](/assets/img/2024-04-16/1.png)
_역할 및 기능 추가 위치_

위 사진과 같게 *역할 및 기능 추가* 메뉴를 선택한다.

<br>

![역할 및 기능 추가 - AD CS 1](/assets/img/2024-04-16/2.png)
_역할 및 기능 추가 - AD CS 1_

*서버 역할* 단계까지 **다음** 버튼을 누르고, **Active Directory 인증서 서비스** 체크박스를 체크한다.

![역할 및 기능 추가 - AD CS 2](/assets/img/2024-04-16/3.png)
_역할 및 기능 추가 - AD CS 2_

팝업된 창에서 **기능 추가** 버튼을 눌러 진행한다.

<br>

![역할 및 기능 추가 - AD CS 3](/assets/img/2024-04-16/4.png)
_역할 및 기능 추가 - AD CS 3_

위 사진의 상태와 같다면 **다음** 버튼을 누른다.

<br>

![역할 및 기능 추가 - AD CS 4](/assets/img/2024-04-16/5.png)
_역할 및 기능 추가 - AD CS 4_

*AD CS* 단계의 *역할 서비스* 선택 단계까지 **다음** 버튼을 누른 뒤, **인증 기관 웹 등록** 체크박스를 선택한다.

![역할 및 기능 추가 - AD CS 5](/assets/img/2024-04-16/6.png)
_역할 및 기능 추가 - AD CS 5_

팝업되는 창에서 **기능 추가** 버튼을 눌러서 진행한다.

![역할 및 기능 추가 - AD CS 6](/assets/img/2024-04-16/7.png)
_역할 및 기능 추가 - AD CS 6_

위 사진과 같은 상태인지 확인하고 **다음** 버튼을 누른다.

<br>

![역할 및 기능 추가 - AD CS 7](/assets/img/2024-04-16/8.png)
_역할 및 기능 추가 - AD CS 7_

함께 설치되는 서비스인 웹 서버의 *보안* 토글 메뉴의 **IIS 클라이언트 인증서 매핑 인증**과 **클라이언트 인증서 매핑 인증** 체크박스를 체크하고 **다음** 버튼을 누른다.

<br>

![역할 및 기능 추가 - AD CS 8](/assets/img/2024-04-16/9.png)
_역할 및 기능 추가 - AD CS 8_

현재까지 설정에 의해 어떤 서비스가 설치되는지 확인하는 단계이다. **설치** 버튼을 누른다.

<br>

잠시 기다리면 다음 사진과 같은 상태가 된다.

![역할 및 기능 추가 - AD CS 설치 완료](/assets/img/2024-04-16/10.png)
_역할 및 기능 추가 - AD CS 설치 완료_

**대상 서버에서 Active Directory 인증서 서비스 구성** 링크를 선택하거나, **닫기** 버튼을 누른 뒤, *관리* 메뉴의 옆에 있는 깃발 아이콘(알림)을 눌러 구성할 수 있다.

<br>

![AD CS 구성 1](/assets/img/2024-04-16/11.png)
_AD CS 구성 1_

*AD 인증서 서버*가 되기 위한 조건에 대해 나와있다. 해당 자격 증명을 진행할 서버(DC)를 선택할 수도 있지만, 현재 DC1만 DC(Domain Controller)이기 때문에 **다음** 버튼을 누른다.

<br>

![AD CS 구성 2](/assets/img/2024-04-16/12.png)
_AD CS 구성 2_

위 사진과 같은 상태가 되도록 체크박스를 선택한 뒤, **다음** 버튼을 누른다. ~~역할 및 기능 추가에서 체크한 서비스만 선택 가능하다.~~

<br>

![AD CS 구성 3](/assets/img/2024-04-16/13.png)
_AD CS 구성 3_

*엔터프라이즈 CA(Certificate Authority)*에서 진행하는 것이 좀 더 간단하게 인증서를 발급받을 수 있다. 하지만 이번 포스트에서는 **독립 CA**로 구성하도록 한다. **다음** 버튼을 누른다.

<br>

![AD CS 구성 4](/assets/img/2024-04-16/14.png)
_AD CS 구성 4_

유일하게 존재하는 DC이기 떄문에 **루트 CA**를 선택하고 **다음** 버튼을 누른다.

<br>

![AD CS 구성 5](/assets/img/2024-04-16/15.png)
_AD CS 구성 5_

CA의 개인키로 암호화하고, 클라이언트가 소지한 CA의 공개키로 복호화가 진행된다. **새 개인 키 만들기** 라디오 버튼을 선택한 뒤, **다음** 버튼을 누른다.

<br>

![AD CS 구성 6](/assets/img/2024-04-16/16.png)
_AD CS 구성 6_

필요한 수준의 보안을 위해 *키 길이*를 변경하거나 *해시 알고리즘*을 변경해 진행할 수 있지만, 위 사진처럼 기본값으로 진행한다. **다음** 버튼을 누른다.

<br>

![AD CS 구성 7](/assets/img/2024-04-16/17.png)
_AD CS 구성 7_

CA(인증 기관)를 식별하는 이름을 설정하는 단계이다. 바로 **다음** 버튼을 눌러 기본값으로 진행한다. 

<br>

![AD CS 구성 8](/assets/img/2024-04-16/18.png)
_AD CS 구성 8_

인증서의 유효 기간을 설정하는 단계이다. 이 역시 기본값으로 진행하기 위해 바로 **다음** 버튼을 누른다.

<br>

![AD CS 구성 9](/assets/img/2024-04-16/19.png)
_AD CS 구성 9_

인증서 서비스의 데이터베이스를 지정하는 단계이다. 기본값으로 진행한다. **다음**

<br>

![AD CS 구성 10](/assets/img/2024-04-16/20.png)
_AD CS 구성 10_

지금까지 진행한 구성 설정을 검토하는 단계이다. **구성** 버튼을 눌러서 설정값을 적용한다.

<br>

![AD CS 구성 완료](/assets/img/2024-04-16/21.png)
_AD CS 구성 완료_

AD CS의 설치 및 구성이 완료되었다. **닫기** 버튼을 누른다.

<br>

---

#### Domain Group 설정

<br>

이제 도메인 사용자 및 그룹을 만들고 도메인 구성원들의 설정을 변경해보도록 한다.

*blogAdmin*이라는 사용자를 생성하고, *Domain Admins* 그룹에 편입시킨 뒤, 해당 사용자의 *로그온 시간* 및 *로그온 대상*을 설정한다.

> AD 구성 후, *Active Directory 사용자 및 컴퓨터* 메뉴(실행창 `dsa.msc`)의 루트 도메인 관리자 그룹으로 *Domain Admins* 그룹과 *Enterprise Admins*가 생성된다. 이 그룹들은 AD 빌드 및 재해 복구에만 사용해야 하며, 이 그룹의 구성원으로 *Administrator* 범주 안에 들어가지 않는 사용자 계정을 포함하면 안된다.
{: .prompt-warning }

~~일반적인 상황은 아니지만, 실습 단계에서 관리자 계정을 추가한다는 개념입니다.~~

<br>

![명령 프롬프트 - 사용자 추가 및 확인](/assets/img/2024-04-16/22.png)
_명령 프롬프트 - 사용자 추가 및 확인_

*명령 프롬프트*를 실행(실행창 `cmd`)한다.

`net user [사용자명] [비밀번호] /add` 명령어를 입력해 사용자를 추가하고, `net user` 명령어를 입력해 사용자가 정상적으로 추가되었는지 확인한다.

<br>

![AD 사용자 및 컴퓨터 위치](/assets/img/2024-04-16/23.png)
_AD 사용자 및 컴퓨터 위치_

위 사진처럼 *도구* 탭의 강조된 메뉴를 선택하거나, 실행창에서 `dsa.msc`를 입력해 *Active Directory 사용자 및 컴퓨터* 창을 실행한다.

<br>

![AD 사용자 및 컴퓨터 - Domain Admins 위치](/assets/img/2024-04-16/24.png)
_AD 사용자 및 컴퓨터 - Domain Admins 위치_

루트 도메인 확장 -> Users에서 방금 생성한 계정이 추가되었고, *Domain Admins* 그룹이 존재하는 것을 확인했다.

*Domain Admins* 그룹 선택, 마우스 우클릭 -> **속성** 메뉴를 선택한다.

<br>

![AD 사용자 및 컴퓨터 - Domain Admins 구성원 추가](/assets/img/2024-04-16/25.png)
_AD 사용자 및 컴퓨터 - Domain Admins 구성원 추가_

원하는 사용자를 *Domain Admins* 그룹에 추가한다. **추가** 버튼을 누른다.

![AD 사용자 및 컴퓨터 - Domain Admins 구성원 선택](/assets/img/2024-04-16/26.png)
_AD 사용자 및 컴퓨터 - Domain Admins 구성원 선택_

추가를 원하는 사용자의 이름 중 일부를 입력한 뒤, **이름 확인** 버튼을 누른다.

![AD 사용자 및 컴퓨터 - Domain Admins 구성원 검색 및 추가 확인](/assets/img/2024-04-16/27.gif)
_AD 사용자 및 컴퓨터 - Domain Admins 구성원 검색 및 추가 확인_

위와 같은 과정으로 진행해 그룹에 편입시킨다.

<br>

이제 **SVR1 및 SVR2를 도메인에 가입시킨 상태**라고 가정하고 진행한다.

blogAdmin 사용자를 오전 9시부터 오후 10시까지만 로그온 가능하도록 설정한다.

![사용자 로그온 시간 설정 위치](/assets/img/2024-04-16/28.png)
_사용자 로그온 시간 설정 위치_

위 사진처럼 강조된 부분을 설정하고, **로그온 시간** 버튼을 누른다.

![사용자 로그온 시간 설정 완료](/assets/img/2024-04-16/29.gif)
_사용자 로그온 시간 설정 완료_

위 사진의 과정처럼 진행하면 특정 시간에만 로그인 가능하도록 설정할 수 있다.

<br>

*blogAdmin* 사용자는 SVR1에만 로그온 가능하도록 설정한다.

![사용자 로그온 대상 설정 위치](/assets/img/2024-04-16/30.png)
_사용자 로그온 대상 설정 위치_

![사용자 로그온 대상 설정 완료](/assets/img/2024-04-16/31.gif)
_사용자 로그온 대상 설정 완료_

위 사진의 과정처럼 진행하면 로그온할 대상을 지정할 수 있다.

<br>

이후에 진행할 실습에서 활용할 그룹과 사용자들을 생성한다.

Share라는 공유 폴더를 *Share*라는 그룹만 모든 권한을 가지도록 설정하고, *Share* 그룹의 구성원으로 share1, share2, share3라는 사용자들을 만들어 편입시킨다.

또한, **숨김 공유 폴더 *userdata***를 생성하여, *Domain Admins*, *Administrators*, *Enterprise Admins* 그룹만 접근할 수 있도록 권한을 설정할 것이다.

<br>

---

#### Organization Unit 설정

<br>

---

#### Domain Policy 설정

<br>

---

#### 공유폴더 설정

<br>

---

#### Window Server 방화벽 설정

<br>

---

#### 원격 데스크톱 설정

<br>

---

#### RAID 설정

<br>

---

#### 보안 웹 서버 설정

<br>

---

### SVR1 VPN 서버 설정

<br>

---

### SVR2 VPN 서버 설정

<br>

---

### 테스트

<br>

#### Site to Site VPN 테스트

<br>

---

#### Domain Policy 테스트

<br>

---

#### 공유폴더 테스트

<br>

---

#### 원격 데스크톱 연결 테스트

<br>

---

#### RAID 및 보안 웹 서버 테스트

