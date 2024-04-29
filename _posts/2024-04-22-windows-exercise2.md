---
title: 윈도우 서버 실습 2 - Hyper-V 가상 머신들로 수업 복습과 네트워크 공유 그리고 Hyper-V Live Migration 실습
excerpt: 
author: minyeokue
date: 2024-04-22 17:48:54 +0900
last_modified_at: 2024-04-29 22:36:10 +0900
categories: [Exercise]
tags: [Windows, Hyper-V, Live-Migration, Firewall, Network, Active Directory]

toc: true
toc_sticky: true
---

<br>

Window Server에서 Hyper-V 가상화로 지난 수업들을 복습하고, 가상 머신들을 실시간 복제(Live Migration)하며 핵심 기능을 학습한다.

<br>

---

## 시나리오

<br>

- [기본 환경 구성](#기본-환경-구성)

- DC1

    - Domain Controller

    - [저장소 공유](#저장소-공유)

- SVR1 & SVR2

    - Hyper-V 호스트

    - [가상 컴퓨터 생성](#가상-컴퓨터-생성)

        - [Hyper-V 설정](#hyper-v-설정)    

    - [NAT 구성](#nat-구성)

        - [Windows Server 2003 Telnet 서비스 설정](#windows-server-2003-telnet-서비스-설정)

        - [Windows Server 2003 Telnet 원격 데스크톱 연결](#windows-server-2003-telnet-원격-데스크톱-연결)

        - [NAT 포트포워딩](#nat-포트포워딩)

            - [테스트 - Telnet 접속](#테스트---telnet-접속)

            - [테스트 - 원격 데스크톱 연결](#테스트---원격-데스크톱-연결)

    - [Hyper-V 핵심기능](#hyper-v-핵심-기능)

        - [Hyper-V Shared Nothing Migration](#hyper-v-shared-nothing-migration)

            - [Hyper-V Shared Nothing Migration 테스트](#shared-nothing-live-migration-테스트)

        - [Hyper-V SMB Live Migration](#hyper-v-smb-live-migration)

            - [Hyper-V SMB Live Migration 테스트](#hyper-v-smb-live-migration-테스트)

        - [Hyper-V Storage Live Migration](#hyper-v-storage-live-migration)

        - [Hyper-V Replication](#hyper-v-replication)

            - [Hyper-V Replcation 테스트](#hyper-v-replication-테스트)

<br>

---

### 기본 환경 구성

<br>

기본적으로 Hyper-V를 실행하려면 관련 설정이 필요하다.

메모리와 CPU 코어 개수를 조정하고, 가상 CPU의 가상화 기능을 켜줘야 Hyper-V를 설치할 수 있다.

![가상머신 메모리 추가](/assets/img/2024-04-22/1.png)
_가상머신 메모리 추가_

![가상머신 CPU 추가 및 가상화 기능 활성화](/assets/img/2024-04-22/2.png)
_가상머신 CPU 추가 및 가상화 기능 활성화_

<br>

이후 가상머신의 환경설정을 저장하는 파일의 수정이 필요하다.

해당 VMware가 저장된 폴더로 이동한 뒤 `가상머신명.vmx`{:.filepath} 파일을 수정한다.

![가상머신 설정 파일 수정](/assets/img/2024-04-22/3.png)
_가상머신 설정 파일 수정_

맨 아래에 `hypervisor.cpuid.v0 = "FALSE"`를 추가한다.

<br>

본격적인 설치하기 전, 가상머신들과 가상 하드디스크들이 저장될 폴더를 생성하겠다.

![가상머신 저장 폴더 생성](/assets/img/2024-04-22/4.png)
_가상머신 저장 폴더 생성_

`C:\Hyper-V`{:.filepath} 폴더를 생성하고 그 안에 `C:\Hyper-V\VHDs`{:.filepath}를 생성한다.

<br>

Hyper-V 호스트도 역시 Active Directory 환경이어야 한다. 핵심 기능을 사용하는 데 있어 인증 절차를 Domain Controller에 위임하는 경우가 생기기 때문이다.

그것은 추후 실제 실습에서 설명하도록 하고, DC1은 Active Directory 서비스가 설치된 스냅샷으로 롤백하고, SVR1과 SVR2에 Hyper-V 관리자를 설치한 이후 Domain에 가입시킨다.

먼저 Hyper-V 기능을 추가한다.

![역할 및 기능 추가 위치](/assets/img/2024-04-22/5.png)
_역할 및 기능 추가 위치_

![Hyper-V 기능 추가 1](/assets/img/2024-04-22/6.png)
_Hyper-V 기능 추가 1_

![Hyper-V 기능 추가 2](/assets/img/2024-04-22/7.png)
_Hyper-V 기능 추가 2_

<br>

![Hyper-V 기능 추가 3](/assets/img/2024-04-22/8.png)
_Hyper-V 기능 추가 3_

위 단계까지 **다음** 버튼을 누르고, 이후 Ethernet0를 선택하고 **다음** 버튼을 누른다.

<br>

![Hyper-V 기능 추가 4](/assets/img/2024-04-22/9.png)
_Hyper-V 기능 추가 4_

*실시간 마이그레이션* 단계는 Hyper-V의 핵심 기능이지만, 추후에 설명하도록 하고 지금은 **다음** 버튼을 눌러서 설치를 진행하겠다.

<br>

![Hyper-V 기능 추가 5](/assets/img/2024-04-22/10.png)
_Hyper-V 기능 추가 5_

아까 생성했던 폴더들을 지정한 뒤 **다음** 버튼을 누른다.

<br>

![Hyper-V 기능 추가 설치](/assets/img/2024-04-22/11.png)
_Hyper-V 기능 추가 설치_

위 단계까지 진행한 뒤, **설치** 버튼을 누른다. 잠시 기다린다.

<br>

![Hyper-V 기능 추가 후 다시 시작](/assets/img/2024-04-22/12.png)
_Hyper-V 기능 추가 후 다시 시작_

설치가 이루어진 뒤, *역할 및 기능 추가 마법사* **닫기** 버튼을 누르면 위와 같이 *알림* 메뉴에서 **서버를 다시 시작**해야 한다고 알린다.

**계획됨**을 선택하여 시스템일 재부팅한다.

<br>

재부팅이 완료되면 다음처럼 진행한다.

![Hyper-V 관리자 위치](/assets/img/2024-04-22/13.png)
_Hyper-V 관리자 위치_

*도구* 탭의 **Hyper-V 관리자** 메뉴를 선택하거나, Window + R키로 실행창에서 `virtmgmt.msc`를 입력한다.

<br>

![Hyper-V 가상 스위치 관리자 위치](/assets/img/2024-04-22/14.png)
_Hyper-V 가상 스위치 관리자 위치_

이후 현재 켜져있는 서버를 우측 마우스 클릭 -> **가상 스위치 관리자** 메뉴를 선택한다.

<br>

**새 가상 네트워크 스위치**에서 *외부*, *내부*, *개인*을 선택해 총 3개의 스위치를 생성한다.

![Hyper-V 가상 스위치 관리자 - 가상 네트워크 스위치 추가](/assets/img/2024-04-22/15.png)
_Hyper-V 가상 스위치 관리자 - 가상 네트워크 스위치 추가_

<br>

![Hyper-V 가상 스위치 추가 완료](/assets/img/2024-04-22/16.gif)
_Hyper-V 가상 스위치 추가 완료_

위 상태와 같이 가상 스위치들을 추가 완료하였다.

<br>

![Hyper-V 가상 스위치 - External IP 설정](/assets/img/2024-04-22/17.gif)
_Hyper-V 가상 스위치 - External IP 설정_

외부와 연결된 가상 스위치를 직접 IP를 설정한다.

<br>

SVR2도 같은 과정으로 진행한다.

그 이후, Domain Controller가 관리할 수 있도록 **Domain에 가입**한다. 반드시 필요한 과정이니 잊지 않고 진행한다.

또한, 로컬 계정이 아닌 **도메인 관리자 계정**으로 접속하도록 하자. ~~물론 일반적인 상황은 아니나, 실습이 원활하게 이루어지도록 관리자 계정으로 접속하는 것이다.~~

![Hyper-V 호스트 - 도메인 로그인](/assets/img/2024-04-22/18.gif)
_Hyper-V 호스트 - 도메인 로그인_

<br>

---

### 가상 컴퓨터 생성

<br>

이제 Linux와 Window Server 2003의 가상 하드디스크를 생성한다.

**sysprep**이라는 개념이 등장하는데, 이것은 윈도우 가상머신의 신속한 배포를 위해 미리 준비하는 것으로 *System Preparation*의 약자이다.

Windows의 Domain Controller는 각 컴퓨터들을 SID라는 고유한 보안 식별자로 구분하는데, 이것이 같은 경우 도메인에 가입이 안되기 때문이다.

<br>

![Hyper-V 관리자 - 가상 하드디스크 추가 위치](/assets/img/2024-04-22/19.png)
_Hyper-V 관리자 - 가상 하드디스크 추가 위치_

새로운 가상 하드디스크를 추가한다.

![Hyper-V 관리자 - 가상 하드디스크 추가 1](/assets/img/2024-04-22/20.png)
_Hyper-V 관리자 - 가상 하드디스크 추가 1_

<br>

![Hyper-V 관리자 - 가상 하드디스크 추가 2](/assets/img/2024-04-22/21.png)
_Hyper-V 관리자 - 가상 하드디스크 추가 2_

VHD보다 발전된 형태의 **VHDX 형식**으로 가상 하드디스크를 생성하겠다.

<br>

![Hyper-V 관리자 - 가상 하드디스크 추가 3](/assets/img/2024-04-22/22.png)
_Hyper-V 관리자 - 가상 하드디스크 추가 3_

*고정 크기*가 실질적인 성능은 뛰어나지만 공간을 잡는 만큼 해당 용량을 차지하기 때문에 선택하지 않고, **동적 확장**을 선택하고 **다음** 버튼을 누른다.

<br>

![Hyper-V 관리자 - 가상 하드디스크 추가 4](/assets/img/2024-04-22/23.png)
_Hyper-V 관리자 - 가상 하드디스크 추가 4_

이전에 미리 설정해뒀던 가상 하드디스크가 저장되는 장소로 **win2003-1.vhdx** 가상 하드디스크를 생성한다.

<br>

![Hyper-V 관리자 - 가상 하드디스크 추가 5](/assets/img/2024-04-22/24.png)
_Hyper-V 관리자 - 가상 하드디스크 추가 5_

*비어 있는 새 가상 하드 디스크 만들기*를 선택하고 **20GB**를 입력하고 **다음** 버튼을 누른다. *동적 디스크*이기 때문에, 20GB 전부를 차지하지는 않을 것이다. -> 이후 **마침** 버튼을 누른다.

<br>

이제 새로운 가상 컴퓨터를 생성한다.

![Hyper-V 관리자 - 가상 컴퓨터 생성 위치](/assets/img/2024-04-22/25.png)
_Hyper-V 관리자 - 가상 컴퓨터 생성 위치_

<br>

![Hyper-V 관리자 - 가상 컴퓨터 생성 1](/assets/img/2024-04-22/26.png)
_Hyper-V 관리자 - 가상 컴퓨터 생성 1_

**다음** 버튼을 누른다.

<br>

![Hyper-V 관리자 - 가상 컴퓨터 생성 2](/assets/img/2024-04-22/27.png)
_Hyper-V 관리자 - 가상 컴퓨터 생성 2_

**win2003-1**이라는 이름으로 가상 컴퓨터를 생성하겠다고 지정하고, **다음** 버튼을 누른다.

<br>

![Hyper-V 관리자 - 가상 컴퓨터 생성 3](/assets/img/2024-04-22/28.png)
_Hyper-V 관리자 - 가상 컴퓨터 생성 3_

**1세대**를 선택하고, **다음** 버튼을 누른다.

<br>

![Hyper-V 관리자 - 가상 컴퓨터 생성 4](/assets/img/2024-04-22/29.png)
_Hyper-V 관리자 - 가상 컴퓨터 생성 4_

메모리를 **512MB**로 설정하고, **다음** 버튼을 누른다.

메모리는 VMware를 실행한 호스트 OS의 실제 메모리에 기반하여 적절한 양을 선택해야 한다.

<br>

![Hyper-V 관리자 - 가상 컴퓨터 생성 5](/assets/img/2024-04-22/30.png)
_Hyper-V 관리자 - 가상 컴퓨터 생성 5_

외부 인터넷과 통신을 하기 위해 VMnet8과 동일한 대역인 192.168.1.X 대역의 IP인 **External**을 선택하고 **다음** 버튼을 누른다.

<br>

![Hyper-V 관리자 - 가상 컴퓨터 생성 6](/assets/img/2024-04-22/31.png)
_Hyper-V 관리자 - 가상 컴퓨터 생성 6_

아까 생성했던 가상 하드디스크를 지정하고, **다음** 버튼을 누른다.

![Hyper-V 관리자 - 가상 컴퓨터 생성 완료](/assets/img/2024-04-22/32.png)
_Hyper-V 관리자 - 가상 컴퓨터 생성 완료_

**마침** 버튼을 누른다.

<br>

![Hyper-V 관리자 - 가상 컴퓨터 생성 확인](/assets/img/2024-04-22/33.png)
_Hyper-V 관리자 - 가상 컴퓨터 생성 확인_

가상 컴퓨터가 생성된 것을 확인할 수 있다.

이제 OS 이미지 파일을 마운트시키고 부팅하면 설치가 시작된다.

<br>

![Hyper-V 관리자 - 가상 컴퓨터 설정 위치](/assets/img/2024-04-22/34.png)
_Hyper-V 관리자 - 가상 컴퓨터 설정 위치_

마운트 시키기 위해 가상 컴퓨터의 설정을 조작한다.

![Hyper-V 관리자 - 가상 컴퓨터 이미지 마운트](/assets/img/2024-04-22/35.png)
_Hyper-V 관리자 - 가상 컴퓨터 이미지 마운트_

위 사진처럼 필요한 이미지를 마운트하고, 부팅하면 설치가 진행된다.

설치가 다 진행되었다고 가정한다. 이제 신속한 배포를 위해 **sysprep**을 생성할 것이다.

<br>

![Hyper-V - 가상 컴퓨터 sysprep 1](/assets/img/2024-04-22/36.png)
_Hyper-V - 가상 컴퓨터 sysprep 1_

이미지 파일이 마운트된 디스크를 **열기**를 눌러 들어온 상태이다. **SUPPORT** 폴더 안으로 들어간다.

![Hyper-V - 가상 컴퓨터 sysprep 2](/assets/img/2024-04-22/37.png)
_Hyper-V - 가상 컴퓨터 sysprep 2_

**TOOLS** 폴더 안으로 들어간다.

<br>

![Hyper-V - 가상 컴퓨터 sysprep 3](/assets/img/2024-04-22/38.png)
_Hyper-V - 가상 컴퓨터 sysprep 3_

**DEPLOY.CAB**을 더블 클릭한다.

<br>

![Hyper-V - 가상 컴퓨터 sysprep 4](/assets/img/2024-04-22/39.png)
_Hyper-V - 가상 컴퓨터 sysprep 4_

모든 파일을 *드래그로 선택*하고 *오른쪽 마우스 클릭* 후, **압축 풀기** 메뉴를 선택한다.

<br>

![Hyper-V - 가상 컴퓨터 sysprep 5](/assets/img/2024-04-22/40.png)
_Hyper-V - 가상 컴퓨터 sysprep 5_

C 드라이브 아래에 **sysprep**이라는 폴더를 생성한 뒤, 선택하고 **압축 풀기** 버튼을 누른다.

<br>

이제 선택했던 파일들이 *sysprep* 폴더 안에 압축이 풀렸을 것이다.

해당 폴더로 이동하면 다음과 같은 상태이다.

![Hyper-V - 가상 컴퓨터 sysprep 6](/assets/img/2024-04-22/41.png)
_Hyper-V - 가상 컴퓨터 sysprep 6_

**sysprep.exe** 파일을 실행한다.

![Hyper-V - 가상 컴퓨터 sysprep 7](/assets/img/2024-04-22/42.png)
_Hyper-V - 가상 컴퓨터 sysprep 7_

**확인** 버튼을 누른다.

<br>

![Hyper-V - 가상 컴퓨터 sysprep 8](/assets/img/2024-04-22/43.png)
_Hyper-V - 가상 컴퓨터 sysprep 8_

**다시 봉인** 버튼을 누른다.

![Hyper-V - 가상 컴퓨터 sysprep 완료](/assets/img/2024-04-22/44.png)
_Hyper-V - 가상 컴퓨터 sysprep 완료_

*보안 ID(SID)*를 다시 생성하여 동일한 값이 생길 수 없도록 한다.

잠시 기다리면 컴퓨터가 종료된다.

지금 상태에서 해당 디스크를 복사하면 부팅할 때마다 새로운 SID를 생성하며 신속한 배포가 가능한 상태가 된다.

> 최신 OS의 경우 Window + R키 -> 실행창에서 `sysprep`을 입력하면 바로 시스템 준비도구를 실행시키는 폴더가 열린다. 즉, 최신 OS가 좀 더 수월하고 빠르게 가상머신들을 배포할 수 있다.
{: .prompt-info }

> **SID**를 확인하려면 *명령 프롬프트*에서 `whoami /user` 명령어를 입력해서 확인할 수 있다.
{: .prompt-tip }

<br>

![SVR1 가상 하드디스크 상태](/assets/img/2024-04-22/45.png)
_SVR1 가상 하드디스크 상태_

현재 만들어진 **sysprep**을 복사하여 Windows Server 2003이 설치된 하드디스크 2개가 준비되어 있다.

DC1에도 1개를 복사하여 공유받은 하드디스크를 통해 가상머신을 부팅시킬 것이다.

![DC1 가상 하드디스크 상태](/assets/img/2024-04-22/46.png)
_DC1 가상 하드디스크 상태_

아직 공유를 진행하지는 않았지만, `C:\smb\win2003-3.vhdx`{:.filepath}라는 폴더와 sysprep을 준비했다.

<br>

---

#### 저장소 공유

<br>

![DC1 공유 폴더 설정 1](/assets/img/2024-04-22/47.png)
_DC1 공유 폴더 설정 1_

**sysprep**이 들어있는 폴더를 네트워크 공유를 통해서 각 Hyper-V 호스트들이 접근할 수 있도록 한다.

*공유* 탭에서 **고급 공유** 버튼을 누른다.

<br>

![DC1 공유 폴더 설정 2](/assets/img/2024-04-22/48.png)
_DC1 공유 폴더 설정 2_

**선택한 폴더 공유**를 체크하고, **권한** 버튼을 누른다.

![DC1 공유 폴더 설정 3](/assets/img/2024-04-22/49.png)
_DC1 공유 폴더 설정 3_

*Everyone* 그룹에게 모든 권한을 부여한다. *Everyone* 그룹은 익명 사용자도 포함하는 그룹으로 보안 상 허용하면 안되는 그룹이지만, 현재 사설망에 존재하는 가상머신이며 실습임을 감안해 진행한다.

<br>

---

SVR1에는 `win2003-1`{:.filepath}, `win2003-2`{:.filepath}, `win2003-3`{:.filepath}을 생성하고, 각각의 CD-ROM에는 Windows Server 2003 이미지 파일을 마운트하여 부팅하도록 한다.

![가상 컴퓨터 생성 1](/assets/img/2024-04-22/50.png)
_가상 컴퓨터 생성 1_

부팅시키고 기다리다보면 위 사진과 같은 상태가 된다. **다음** 버튼을 눌러 진행한다.

이후 약관 동의를 진행하고, 시리얼 번호를 입력하는 등의 과정이 진행된다.

<br>

![가상 컴퓨터 생성 2](/assets/img/2024-04-22/51.png)
_가상 컴퓨터 생성 2_

사용자의 이름을 지정하고, 이어서 진행하다보면 컴퓨터의 이름도 지정한다. 암호의 경우 *sysprep*을 봉인하기 전 입력했던 암호가 설정되어있어서 암호를 지정하지 않아도 기본 비밀번호로 설정된다.

![가상 컴퓨터 생성 3](/assets/img/2024-04-22/52.png)
_가상 컴퓨터 생성 3_

<br>

![가상 컴퓨터 생성 완료](/assets/img/2024-04-22/53.png)
_가상 컴퓨터 생성 완료_

Windows Server 2003의 상세한 설치 과정은 생략하고, SVR1에 존재하는 `win2003-1`{:.filepath}, `win2003-2`{:.filepath}과, DC1에서 공유받은 폴더에 있는 가상 하드디스크로 부팅한 `win2003-3`{:.filepath}을 모두 설치한 상태이다.

<br>

---

#### Hyper-V 설정

<br>

`win2003-1`{:.filepath}은 NAT 서버로 활용하기 위해 네트워크 어댑터를 총 3개 장착시키고, `win2003-2`{:.filepath}, `win2003-3`{:.filepath}은 사설망에 구축해 처음엔 통신이 불가능하지만 NAT 서버에서 *라우팅 및 원격 액세스*를 구성하여 서로 통신이 가능하도록 한다.

`win2003-2`{:.filepath}에 Telnet 서버를 구축하고, `win2003-3`{:.filepath}에는 원격 데스크톱이 가능하도록 `win2003-1`{:.filepath}에서 NAT 포트포워딩을 통해 ~~가상의~~ 외부 인터넷에서 접속할 수 있도록 한다.

`win2003-2`{:.filepath}는 10.10.10.0/24 대역에 존재하고, `win2003-3`{:.filepath}는 10.10.20.0/24 대역에 존재해 서로 통신이 불가능한 상태로 설정할 것이며, `win2003-1`{:.filepath}에 추가되는 2개의 네트워크 어댑터는 10.10.10.0/24 대역과 10.10.20.0/24 대역의 게이트웨이 역할을 할 것이다.

그러기 위해서는 *가상 스위치 관리자*에서 새로운 **내부** 네트워크를 추가해야 한다. 해당 메뉴의 위치는 다음과 같다.

![Hyper-V 가상 스위치 관리자 위치](/assets/img/2024-04-22/54.png)
_Hyper-V 가상 스위치 관리자 위치_

둘 중 하나를 눌러서 진행한다.

<br>

![Hyper-V 가상 스위치 관리자 - 새 가상 네트워크 스위치 추가](/assets/img/2024-04-22/55.png)
_Hyper-V 가상 스위치 관리자 - 새 가상 네트워크 스위치 추가_

처음 설정할 때 만들어뒀던 외부, 내부, 개인 네트워크 스위치들 외에 새로운 **내부** 네트워크 스위치를 추가한다.

<br>

![Hyper-V 가상 스위치 관리자 - Internal2 내부 스위치 추가](/assets/img/2024-04-22/56.png)
_Hyper-V 가상 스위치 관리자 - Internal2 내부 스위치 추가_

추가되는 가상 스위치의 이름을 "Internal2"로 설정하고 **확인** 혹은 **적용**을 누른다.

<br>

이제 NAT 서버(`win2003-1`{:.filepath})에 네트워크 어댑터를 2개 추가한다. 그러기 위한 메뉴의 위치는 다음과 같다.

![Hyper-V 설정 - 위치](/assets/img/2024-04-22/57.png)
_Hyper-V 설정 - 위치_

둘 중 하나를 눌러서 진행할 수 있다. 

<br>

![Hyper-V 설정 - 가상 네트워크 스위치 추가 진행](/assets/img/2024-04-22/58.gif)
_Hyper-V 설정 - 가상 네트워크 스위치 추가 진행_

위와 같이 진행하면 3개의 NIC(Network Interface Controller)을 장착한 것과 같다.

<br>

*가상 스위치 관리자*에서 진행했던 과정을 통해 SVR1 네트워크 어댑터에 변화가 생겼다. 확인해본다.

![가상 스위치 추가로 인한 SVR1 네트워크 연결 변화](/assets/img/2024-04-22/59.gif)
_가상 스위치 추가로 인한 SVR1 네트워크 연결 변화_

vEthernet이 추가된 것을 확인할 수 있다.

<br>

위에서 진행한 것처럼 `win2003-2`{:.filepath}의 네트워크 어댑터는 "Internal"과 연결하고, `win2003-3`{:.filepath}의 네트워크 어댑터는 "Internal2"와 연결한다.

`win2003-2`{:.filepath}의 네트워크 어댑터가 "Internal"인 것을 확인하고, 연결 후 부팅을 시작한다.

![win2003-2 가상 컴퓨터 시작](/assets/img/2024-04-22/60.gif)
_win2003-2 가상 컴퓨터 시작_

<br>

`win2003-2`{:.filepath}의 IP를 설정한다.

![win2003-2 IP 설정](/assets/img/2024-04-22/61.gif)
_win2003-2 IP 설정_

위와 같은 과정을 반복하며, `win2003-3`{:.filepath}의 네트워크 어댑터는 "Internal2"인지 확인하고 부팅 후 IP를 설정한다.

<br>

`win2003-2`{:.filepath}와 `win2003-3`{:.filepath}는 서로 다른 대역의 네트워크여서 통신이 불가능한지 테스트한다. 분명히 되지 않을 것이다.

![win2003-2, win2003-3 통신 확인](/assets/img/2024-04-22/62.gif)
_win2003-2, win2003-3 통신 확인_

통신이 불가능한 것을 확인했다. 아직 `win2003-1`{:.filepath}에서 게이트웨이 IP를 설정하지 않았을 뿐더러, NAT 기능(라우터 기능 포함)을 구성하지 않았기 때문이다.

<br>

---

### NAT 구성

<br>

이제 `win2003-1`{:.filepath}에서 네트워크 연결을 설정한다.

- Public(External)

    - IP : 192.168.1.111/24

    - GW : 192.168.1.2

    - DNS : 8.8.8.8

- GW1(Internal)

    - IP : 10.10.10.1/24

- GW2(Internal2)

    - IP : 10.10.20.1/24

주의할 점은 가상 네트워크 스위치의 MAC 주소를 잘 확인하고 설정해야 한다. MAC 주소를 확인하는 방범은 다음을 참고한다.

![win2003-1 MAC 주소 확인](/assets/img/2024-04-22/63.gif)
_win2003-1 MAC 주소 확인_

`ipconfig /all` 명령어를 명령 프롬프트에서 입력해 MAC 주소를 포함한 NIC의 많은 정보를 확인할 수 있다.

<br>

![win2003-1 네트워크 연결 상태](/assets/img/2024-04-22/64.png)
_win2003-1 네트워크 연결 상태_

IP 및 네트워크 연결의 이름을 변경한 상태이다. 

<br>

위와 같이 설정한 뒤, *라우팅 및 원격 액세스*를 통해 NAT 서버의 역할을 구성한다.

![라우팅 및 원격 액세스 메뉴 위치](/assets/img/2024-04-22/65.png)
_라우팅 및 원격 액세스 메뉴 위치_

![라우팅 및 원격 액세스 구성 및 사용](/assets/img/2024-04-22/66.png)
_라우팅 및 원격 액세스 구성 및 사용_

*이 컴퓨터의 이름*을 마우스 우클릭해 메뉴를 켜고, **라우팅 및 원격 액세스 구성 및 사용** 메뉴를 선택한다.

<br>

![라우팅 및 원격 액세스 구성 및 사용 설정 진행](/assets/img/2024-04-22/67.gif)
_라우팅 및 원격 액세스 구성 및 사용 설정 진행_

*다음* -> **네트워크 주소 변환(NAT)** -> 외부 인터넷과 연결된 **Public** 선택 후, **기본 방화벽 사용 해제** -> *다음* -> *다음* -> *다음* -> *다음* -> *마침* 버튼 순으로 진행한다.

이제 내부 네트워크에서 통신이 가능해진다.

간단한 테스트를 진행한다.

![win2003-2, win2003-3 통신 가능](/assets/img/2024-04-22/68.gif)
_win2003-2, win2003-3 통신 가능_

이제 기본적인 설정이 끝났다.

<br>

---

#### Windows Server 2003 Telnet 서비스 설정

<br>

![win2003-2 Telnet 서비스 활성화](/assets/img/2024-04-22/69.gif)
_win2003-2 Telnet 서비스 활성화_

실행창에서 `services.msc` 명령어로 *서비스* 창을 띄운 뒤, "t" 입력해 't'로 시작하는 서비스로 바로 이동할 수 있다.

Telnet을 우클릭 한 뒤, **속성** -> *시작 유형*을 *사용 안 함*에서 **자동**으로 변경 -> **적용** -> **시작** -> *확인* 순으로 진행한다.

그 이후, cmd에서 23번 포트가 "LISTENING" 상태로 변한 것을 확인할 수 있다.

<br>

이제 Telnet 서비스를 사용할 유저를 생성하고, 해당 유저를 *TelnetClients* 그룹의 구성원으로 포함시켜야 한다.

![로컬 사용자 및 그룹 - 사용자 추가 및 그룹 구성원 추가](/assets/img/2024-04-22/70.gif)
_로컬 사용자 및 그룹 - 사용자 추가 및 그룹 구성원 추가_

실행창 `lusrmgr.msc` -> *사용자* -> *teluser* 생성 -> *그룹* -> **TelnetClients** 그룹 -> 구성원으로 해당 사용자 추가 순으로 진행한다.

<br>

---

#### Windows Server 2003 Telnet 원격 데스크톱 연결

<br>

![win2003-3 원격 데스크톱 연결 포트 변경 및 활성화](/assets/img/2024-04-22/71.gif)
_win2003-3 원격 데스크톱 연결 포트 변경 및 활성화_

서비스가 활성화되지 않은 상태에서 3389번 포트를 사용하는 서비스가 없는지 확인했다.

그 이후 레지스트리 편집기에서 *원격 데스크톱 연결*에서 사용하는 포트 번호를 검색한다. 다음 찾기는 `F3` 키를 입력해서 진행할 수 있다.

16진수로 적혀있는 정수를 10진수로 변환하니 3389가 확인되며, 이를 35000번으로 변경한 뒤 **원격 데스크톱 연결**을 활성화할 수 있는 메뉴가 있는 곳으로 이동한다.

*내 컴퓨터*를 오른쪽 마우스 클릭 -> **속성** -> *원격* 탭 -> *원격 데스크톱* 메뉴에서 **이 컴퓨터에서 원격 데스크톱 사용** 체크 -> *적용* 순으로 진행한다.

그 이후, 3389번 포트가 대기중인지 확인 -> 35000번 대기중인지 확인한다.

<br>

---

#### NAT 포트포워딩 

<br>

![NAT 포트포워딩 설정 위치](/assets/img/2024-04-22/72.png)
_NAT 포트포워딩 설정 위치_

해당 컴퓨터 확장 -> *IP 라우팅* 확장 -> **NAT/기본 방화벽** -> **Public** 마우스 오른쪽 클릭 -> **속성** 순으로 진행한다.

<br>

Telnet 서비스는 그대로 진행할 수 있지만, 기존에 존재하는 *원격 데스크톱*은 3389 포트를 변경할 수 없기 때문에, **추가** 버튼을 눌러야 한다.

![NAT 포트포워딩 설정 진행](/assets/img/2024-04-22/73.gif)
_NAT 포트포워딩 설정 진행_

위와 같이 설정하면 NAT 포트포워딩이 모두 완료된다. 이제 ~~가상의 외부 인터넷인~~ 호스트 OS에서 접속해보자.

<br>

---

##### 테스트 - Telnet 접속

<br>

![Telnet 접속 테스트](/assets/img/2024-04-22/74.gif)
_Telnet 접속 테스트_

Telnet 접속을 테스트 완료했다.

---

##### 테스트 - 원격 데스크톱 연결

<br>

![원격 데스크톱 연결 테스트](/assets/img/2024-04-22/75.gif)
_원격 데스크톱 연결 테스트_

원격 데스크톱 연결 테스트를 완료했다.

<br>

---

### Hyper-V 핵심 기능

<br>

이제 Hyper-V의 핵심 기능인 Live Migration에 대해서 실습한다. **Live Migration**은 가상 머신이 켜져있는 상태에서 가상머신이 실행되는 프로세스(메모리에 탑재된) 혹은 하드디스크를 옮길 수 있는 기술이다.

가동 중지 시간 없이 실행 중인 Virtual Machines를 한 Hyper-V 호스트에서 다른 호스트로 투명하게 이동할 수 있다. **Windows 장애 조치(Failover) 클러스터링**와 함께 할 경우 Live Migration을 통해 **고가용성** 및 **내결함성** 시스템을 만들 수 있다.

<br>

Hyper-V의 실시간 Migration은 총 3가지가 존재한다.

- SMB Live Migration

- Live Storage Migration

- Shared Nothing Live Migration

먼저 **Shared Nothing Live Migration** 실습을 진행해보겠다.

<br>

---

#### Shared Nothing Live Migration

<br>

`virtmgmt.msc`를 실행창에서 입력해 *Hyper-V 관리자* 창으로 온다.

*SVR1*을 마우스 오른쪽 클릭으로 메뉴를 호출하거나 오른쪽 *작업* 탭에서 **Hyper-V 관리자**를 누른다.

![Hyper-V 관리자 위치](/assets/img/2024-04-22/76.png)
_Hyper-V 관리자 위치_

<br>

![실시간 마이그레이션 설정 1](/assets/img/2024-04-22/77.png)
_실시간 마이그레이션 설정 1_

**들어오고 나가는 실시간 마이그레이션 사용** 체크 -> **다음 IP 주소를 실시간 마이그레이션에 사용** 선택 -> **추가** 버튼

<br>

![실시간 마이그레이션 설정 2](/assets/img/2024-04-22/78.png)
_실시간 마이그레이션 설정 2_

현재 SVR1의 IP(192.168.1.110)를 입력한 뒤 **확인** 버튼을 누른다.

![실시간 마이그레이션 설정 3](/assets/img/2024-04-22/79.png)
_실시간 마이그레이션 설정 3_

추가된 IP 주소를 확인하고 **적용** 혹은 **확인** 버튼을 누른다.

<br>

![실시간 마이그레이션 설정 완료](/assets/img/2024-04-22/80.png)
_실시간 마이그레이션 설정 완료_

*실시간 마이그레이션* 메뉴의 강조된 "+"를 눌러 확장하고 **고급 기능**을 누른다.

**Kerberos 사용**을 선택하고 **확인** 혹은 **적용** 버튼을 누른다.

<br>

Shared Nothing Live Migration을 진행하기 위한 기본 설정이 모두 완료되었다.

SVR2에도 같은 설정을 진행한다. 이제 실습을 진행한다.

<br>

---

##### Shared Nothing Live Migration 테스트

<br>

> **Shared Nothing Live Migration**은 Virtual Machine의 *계획된 Live Migration*을 위해 고가의 하드웨어 요구사항이 필요 없이, 단순한 Hyper-V 호스트 2대만으로 Virtual Machine의 *계획된 Live Migration*을 구현할 수 있다는 장점이 있다.
{: .prompt-info }

![실시간 마이그레이션 위치](/assets/img/2024-04-22/81.png)
_실시간 마이그레이션 위치_

이동하고자 하는 가상 컴퓨터(가상 머신)을 선택하고 *마우스 오른쪽 클릭* 혹은 오른쪽 *작업* 탭에서 **이동** 메뉴를 선택한다.

<br>

![실시간 마이그레이션 마법사 진행](/assets/img/2024-04-22/82.gif)
_실시간 마이그레이션 마법사 진행_

위와 같은 단계로 진행한다. 프로그레스 바가 나오면 잠시 대기한다.

<br>

![실시간 마이그레이션 마법사 ping 확인](/assets/img/2024-04-22/83.gif)
_실시간 마이그레이션 마법사 ping 확인_

위 사진처럼 호스트 OS 명령 프롬프트에서 `ping -t 192.168.1.111` 명령어로 현재 가상 머신의 상태를 확인할 수 있다.

진행 중에 통신이 닿지 않거나 시간이 튀는 부분이 생길 것이다.

<br>

![실시간 마이그레이션 마법사 완료](/assets/img/2024-04-22/84.png)
_실시간 마이그레이션 마법사 완료_

Live Migration이 완료되면 SVR1에서 연결했었던 "win2003-1"과의 세션이 아직 종료되지 않았다.

해당 창에 SVR2.kosa.vm에서 실행 중인 것을 확인할 수 있다.

<br>

![실시간 마이그레이션 확인](/assets/img/2024-04-22/85.gif)
_실시간 마이그레이션 확인_

SVR1의 *Hyper-V 관리자*에서 win2003-1이 사라진 것을 확인할 수 있으며, SVR2의 *Hyper-V 관리자*에서 실행중이다.

<br>

---

#### Hyper-V SMB Live Migration

<br>

**SMB Live Migration**은 Windows Server 2012 Hyper-V에서부터 소개된 기능으로, VM 데이터 파일이 위치하는 **공유 스토리지(SAN or iSCSI)**가 필요하며 지금은 네트워크 공유 기능으로 진행했다.

SMB 공유 서버를 사용해 훨씬 효율적인 SMB Live Migration을 구현할 수도 있다. SMB Live Migration을 진행하기 앞서 먼저 설정해줘야 하는 부분들이 있다.

<br>

DC1에서 공유받은 스토리지인 "smb" 폴더에서 *공유* 탭과 *보안* 탭에서 설정을 진행하고, 실행창에서 `dsa.msc`를 입력해 *Active Directory 사용자 및 컴퓨터* 창에서 설정을 진행해야 한다.

먼저 공유 폴더에서 설정을 진행한다.

![SMB Live Migration 사전 설정 - 위치](/assets/img/2024-04-22/86.png)
_SMB Live Migration 사전 설정 - 위치_

smb 폴더를 *마우스 오른쪽 클릭*으로 메뉴를 호출한 뒤 **속성**을 선택한다.

<br>

![SMB Live Migration 사전 설정 - 네트워크 공유 위치](/assets/img/2024-04-22/87.png)
_SMB Live Migration 사전 설정 - 네트워크 공유 위치_

*공유* 탭에서 **고급 공유** 버튼을 누른다.

<br>

![SMB Live Migration 사전 설정 - 네트워크 공유 권한 위치](/assets/img/2024-04-22/88.png)
_SMB Live Migration 사전 설정 - 네트워크 공유 권한 위치_

**권한** 버튼을 누른다.

<br>

![SMB Live Migration 사전 설정 - 네트워크 공유 권한 추가](/assets/img/2024-04-22/89.gif)
_SMB Live Migration 사전 설정 - 네트워크 공유 권한 추가_

Active Directory의 **Global Catalog** DB에 **LDAP(Lightweight Directory Access Protocol)** 프로토콜을 통해 빠른 질의를 하는데, 대상에 컴퓨터도 추가한다.

그 후 *SVR1*과 *SVR2*를 입력해 공유 폴더의 모든 권한을 부여하고, *Administrators* 그룹을 추가해 역시 모든 권한을 부여한다.

<br>

![SMB Live Migration 사전 설정 - NTFS 보안 권한 추가](/assets/img/2024-04-22/90.gif)
_SMB Live Migration 사전 설정 - NTFS 보안 권한 추가_

SVR1과 SVR2 컴퓨터에 NTFS 사용권한을 부여한다.

> *NTFS 사용권한*은 파티션 혹은 볼륨이 **NTFS 파일 시스템으로 포맷**되어 있어야만 부여할 수 있으며, NTFS 사용 권한은 공유 폴더 사용권한과 달리 파일과 폴더 각각에 대해서 부여할 수 있다. 또한, 부여된 권한은 개별적으로 적용된다. NTFS 사용권한은 *사용자가 로컬로 자원에 액세스하거나, 네트워크를 통해 액세스하는 모든 경우에 적용된다.*
{: .prompt-info }

> NTFS 파티션 또는 볼륨 상에 있는 모든 폴더와 파일들은 자신의 **ACL(Access Control List)**를 가지고 있으며, 이 ACL이 NTFS 사용권한이다. ACL은 액세스 가능한 사용자들의 목록이며 *설정한 사용자들을 제외한 모든 것을 거부하는 화이트리스트(Whitelist; passlist, allowlist)의 특성*을 갖는다. -> **명시적 허가 목록**
{: .prompt-tip }

<br>

실행창에서 `dsa.msc`를 입력해 *Active Directory 사용자 및 컴퓨터* 창을 실행한다.

![SMB Live Migration 사전 설정 - AD 사용자 및 컴퓨터](/assets/img/2024-04-22/91.png)
_SMB Live Migration 사전 설정 - AD 사용자 및 컴퓨터_

<br>

![SMB Live Migration 사전 설정 - AD 사용자 및 컴퓨터 권한 위임 진행](/assets/img/2024-04-22/92.gif)
_SMB Live Migration 사전 설정 - AD 사용자 및 컴퓨터 권한 위임 진행_

SVR1에서 권한을 DC1에게 위임한다. Kerberos 인증을 통해 진행되며 **CIFS(Common Internet File System)**라는 프로토콜을 선택한다.

> CIFS(Common Internet File System)은 네트워크상의 효율적인 파일 공유를 위한 파일 액세스 스토로지 프로토콜로, CIFS는 SMB(Server Message Block) 프로토콜을 기반으로 확장된 버전이다. 또한, 윈도우와 유닉스 환경을 동시에 지원하는 표준 파일 규약입니다.
{: .prompt-info }

추가한 뒤 **확인** 버튼을 누른다. 또한, SVR2를 선택해서 동일하게 진행한다.

<br>

![SMB Live Migration 사전 설정 - AD 사용자 및 컴퓨터 서비스 권한 허용](/assets/img/2024-04-22/93.gif)
_SMB Live Migration 사전 설정 - AD 사용자 및 컴퓨터 서비스 권한 허용_

Domain Controller(DC1)에서 SVR1과 SVR2에 대해 **Microsoft Virtual System Migration** 서비스 종류에 대해 Kerberos 인증 과정을 거쳐 가능하도록 허용하는 과정이다.

여기까지 진행했다면, 사전 설정은 모두 완료되었다.

---

##### Hyper-V SMB Live Migration 테스트

<br>

DC1에서 다시 SVR1으로 돌아온다.

![SMB Live Migration 테스트 진행](/assets/img/2024-04-22/94.gif)
_SMB Live Migration 테스트 진행_

DC1으로부터 공유받은 폴더에 존재하는 가상 하드디스크로 SVR1에서 부팅해 SVR1의 컴퓨터 리소스를 사용하던 "win2003-3"을 SMB Live Migration을 통해 SVR2가 CPU 및 메모리를 이어받아 켜진 상태로 Live Migration을 진행된다.

<br>

![SMB Live Migration 테스트 결과 확인](/assets/img/2024-04-22/95.gif)
_SMB Live Migration 테스트 결과 확인_

"win2003-3"의 실시간 마이그레이션이 정상적으로 완료되었다.

*Shared Nothing Live Migration*에 비하면 더 빠른 속도 진행된다. 그 이유는 *Shared Nothing Live Migration*은 가상 컴퓨터와 가상 하드디스크 모두를 이동하기 때문이며, **SMB Live Migration**은 가상 컴퓨터만 이동해 훨씬 빠르다.

<br>

---

#### Hyper-V Storage Live Migration

<br>

![Storage Live Migration - 하드디스크 상태](/assets/img/2024-04-22/96.png)
_Storage Live Migration - 하드디스크 상태_

![Storage Live Migration - 저장소 마이그레이션 상태](/assets/img/2024-04-22/97.png)
_Storage Live Migration - 저장소 마이그레이션 상태_

현재 상태는 SVR1의 `C:\Hyper-V\VHDs\win2003-2.vhdx`{:.filepath}에 존재한다. DC1의 공유 스토리지로 이동시켜보겠다.

<br>

![Storage Live Migration - 테스트 진행](/assets/img/2024-04-22/98.gif)
_Storage Live Migration - 테스트 진행_

![Storage Live Migration - 테스트 완료](/assets/img/2024-04-22/99.gif)
_Storage Live Migration - 저장소 마이그레이션 상태_

저장소 마이그레이션이 정상적으로 완료되었다.

<br>

---

#### Hyper-V Replication

<br>

**Hyper-V Replica**는 가상 머신의 *"계획된 다운타임 -> Maintenance"* 또는 *"계획되지 않은 다운타임 -> Emergency"*시에 Hyper-V Replica는 **가상 머신의 중단 없는 사용을 보장**할 수 있다.

*Hyper-V Replica*는 가상 머신의 **비동기(Asynchronous)** 복제 방식을 지원하며, 아주 단순하게 구성할 수 있고 공유 스토리지 및 특정 스토리지 하드웨어를 요구하지 않는다.

복제는 통상적인 IP 기반 네트워크를 통해 전송되며, 복제되는 데이터는 전송 중에 **암호화**된다.

또한, Hyper-V Replica는 단독 서버, 장애 조치(Failover) 클러스터 및 혼합 환경 모두 지원한다.

Hyper-V Replica에 참여하는 서버들은 물리적으로 동일한 지역에 위치할 수도 있고, 지역적으로 분산되어 존재할 수도 있다. 이러한 물리적인 서버들은 반드시 동일 도메인 내의 서버일 필요도 없다. 물론 물리적 서버들이 동일 도메인이 아닌 다른 도메인의 멤버 서버일 수도 있다.

<br>

![Replica 복제 서버 설정 진행](/assets/img/2024-04-22/100.gif)
_Replica 복제 서버 설정 진행_

Hyper-V 설정 -> *복제 구성* 메뉴 -> **이 컴퓨터을(를) 복제본 서버로 사용합니다.** 체크 -> **Kerberos 인증 사용(HTTP)** 체크 및 포트 변경 -> **인증된 서버로부터 복제 허용** 체크 -> 실행창 `wf.msc` -> 고급 보안이 포함된 Windows Defender 방화벽 "인바운드 설정" -> *Hyper-V 복제본 HTTP 수신기(TCP-In)* **사용** 순서로 진행한다.

같은 내용을 SVR2에서도 진행한다.

<br>

---

##### Hyper-V Replication 테스트

<br>

SVR2에 있는 Linux01을 복제하며 SVR1을 복제본 서버로 활용할 것이다.

![Replica 주서버 복제 사용](/assets/img/2024-04-22/101.png)
_Replica 주서버 복제 사용_

<br>

![Replica 복제 사용 설정 진행](/assets/img/2024-04-22/102.gif)
_Replica 복제 사용 설정 진행_

이제 복제를 진행한다. 초기 복제를 진행하며 5분 간격으로 변경 사항을 SVR2에서 SVR1으로 전달한다.

![Replica 복제 사용 설정 확인](/assets/img/2024-04-22/103.gif)
_Replica 복제 사용 설정 확인_

조금 기다리면 복제본 상태가 *정상*으로 전환된다.

<br>

이제 **계획된 장애 조치(Failover)**를 진행할 수 있다.

계획된 장애 조치는 해당 가상 머신을 종료한 상태에서 진행할 수 있으며, 복제본 서버를 주 서버로 전환하고 해당 가상 머신을 가동한다.

![Replica - 계획된 장애 조치(Failover) 진행](/assets/img/2024-04-22/104.gif)
_Replica - 계획된 장애 조치(Failover) 진행_

계획된 장애 조치가 진행되어 복제본 서버였던 SVR1이 주서버로 전환되고, 역방향 복제를 진행해 SVR2를 복제본 서버로 설정하는 과정이 필요하다.

모든 테스트 및 실습을 완료하였다.