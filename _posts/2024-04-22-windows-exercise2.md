---
title: 윈도우 서버 실습 2 - Hyper-V 가상 머신들로 수업 복습 및 VLAN 그리고 Hyper-V Live Migration 실습
excerpt: 
author: minyeokue
date: 2024-04-22 17:48:54 +0900
last_modified_at: 2024-04-23 17:49:03 +0900
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

    - [Hyper-V 설정](#hyper-v-설정)

        - [가상 컴퓨터 생성](#가상-컴퓨터-생성)

    - [NAT 구성](#nat-구성)

        - [Windows Server 2003 Telnet 서비스 설정](#windows-server-2003-telnet-서비스-설정)

        - [Windows Server 2003 Telnet 원격 데스크톱 연결](#windows-server-2003-telnet-원격-데스크톱-연결)

    - [VLAN 구성](#vlan-구성)

    - [Hyper-V 핵심기능](#hyper-v-핵심기능)

        - [Hyper-V Shared Nothing Migration](#hyper-v-shared-nothing-migration)

        - [Hyper-V SMB Live Migration](#hyper-v-smb-live-migration)

        - [Hyper-V Storage Live Migration](#hyper-v-storage-live-migration)

        - [Hyper-V Replication](#hyper-v-replication)
    
    - [NFS 공유 및 웹 서버](#nfs-공유-및-웹-서버)

        - [NFS 클라이언트](#nfs-클라이언트)

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

