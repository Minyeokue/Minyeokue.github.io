---
title: 네트워크 실습 5 - EACL, VLAN, 라우팅 프로토콜 재분배
excerpt: "라우터에서 다루는 방화벽이라 할 수 있는 ACL, VLAN을 멀티 레이어 스위치에서 구현하며, 3가지 라우팅 프로토콜의 재분배를 실습하며 복습한다."
author: minyeokue
date: 2024-04-02 21:31:47 +0900
last_modified_at: 2024-04-03 00:49:13 +0900
categories: [Exercise]
tags: [Network, Secure]

toc: true
toc_sticky: true
---

<br>

라우터에서 다루는 방화벽이라 할 수 있는 ACL, VLAN을 멀티 레이어 스위치에서 구현하며, 3가지 라우팅 프로토콜의 재분배를 실습하며 복습한다.

<br>

---

<br>

## 시나리오 1

<br>

- 라우터 2대

    - Router 0

        - 192.168.10.0/24 네트워크

            - PC0 IP : 192.168.10.10/24

            - Server0 IP : 192.168.10.20/24

                - 웹 서버, FTP 서버, DNS 서버
    
    - Router 1

        - 192.168.20.0/24 네트워크

            - PC1 IP : 192.168.20.10/24

            - PC2 IP : 192.168.20.20/24
        
        - 192.168.30.0/24 네트워크

            - PC3 IP : 192.168.30.10/24

            - PC4 IP : 192.168.30.20/24

<br>

각 네트워크의 게이트웨이는 1번으로 설정.

OSPF 라우팅 프로토콜 사용.

라우터 사이의 Serial line의 IP는 192.168.0.0/24 대역을 사용한다.

<br>

- ACL 정책 => Destination : 192.168.10.0/24 네트워크

    - 192.168.20.10 시스템에서 www.minyeokue.gitblog 접속 가능

    - 192.168.20.0 네트워크에서 목적지의 FTP 서비스 접근 가능 / 웹 서버 접속 가능

    - 192.168.20.0 네트워크에서 ping 명령어로 목적지의 시스템들과 통신 가능

    - 192.168.30.10 시스템에서 ping 명령어 통신 가능 / FTP 서비스 접근 가능

    - 192.168.30.0 네트워크에서 보안 웹 접근 가능

<br>

![토폴로지 구성 초기 상태](/assets/img/2024-04-02/1.png)
_토폴로지 구성 초기 상태_

<br>

---

<br>

### 환경 구성

<br>

시스템의 `Desktop` 탭에서 IP 설정하는 부분은 생략하겠다.

다만 주의할 점은 192.168.20.0/24 대역은 DNS로 접근하는 조건이 있기 때문에 다른 시스템들과 다르게 IP 설정할 때 DNS 서버 설정을 할 필요가 있다.

<br>

Server0의 DNS 서비스 설정, FTP 서비스 설정을 다음의 사진과 같이 진행하도록 한다.

<br>

![Server0 DNS 서비스 설정](/assets/img/2024-04-02/2.png)
_Server0 DNS 서비스 설정_

![Server0 FTP 서비스 설정](/assets/img/2024-04-02/3.png)
_Server0 FTP 서비스 설정_

<br>

토폴로지와 같게 IP 설정을 마친 뒤 라우터 설정을 진행한다.

<br>

### Router 설정

<br>

#### Interface 설정

<br>

Router 0

```
Router>enable
Router#configure terminal
Router(config)#hostname R0
R0(config)#interface fa0/0
R0(config-if)#ip addr 192.168.10.1 255.255.255.0
R0(config-if)#no shutdown

R0(config-if)#interface se2/0
R0(config-if)#ip addr 192.168.0.1 255.255.255.0
R0(config-if)#no shutdown
```

<br>

Router 1

```
Router>en
Router#conf t
Router(config)#int fa0/0
Router(config-if)#ip addr 192.168.20.1 255.255.255.0
Router(config-if)#no sh

Router(config-if)#int fa1/0
Router(config-if)#ip addr 192.168.30.1 255.255.255.0
Router(config-if)#no sh

Router(config-if)#int se2/0
Router(config-if)#ip addr 192.168.0.2 255.255.255.0
Router(config-if)#no sh
```

<br>

위와 같이 설정을 마친 뒤 이제 라우팅 프로토콜을 설정한다.

<br>

---

<br>

#### OSPF 라우팅 프로토콜 설정

<br>

Router 0

```
R0#conf t
R0(config)#router ospf 100
R0(config-router)#network 192.168.10.0 0.0.0.255 area 1
R0(config-router)#network 192.168.0.0 0.0.0.255 area 1
R0(config-router)#exit
```

<br>

Router 1

```
Router(config)#router ospf 100
Router(config-router)#network 192.168.20.0 0.0.0.255 area 1
Router(config-router)#network 192.168.30.0 0.0.0.255 area 1
Router(config-router)#network 192.168.0.0 0.0.0.255 area 1
Router(config-router)#exit

Router(config)#do show ip route

Codes: C - connected, S - static, I - IGRP, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

C    192.168.0.0/24 is directly connected, Serial2/0
O    192.168.10.0/24 [110/65] via 192.168.0.1, 00:00:12, Serial2/0
C    192.168.20.0/24 is directly connected, FastEthernet0/0
C    192.168.30.0/24 is directly connected, FastEthernet1/0
```

<br>

위의 상태와 같다면 성공적으로 이루어진 것이다. Area 번호를 일치시켜야 한다.

<br>

---

<br>

### EACL(Extended Access Control List) 정책 설정

<br>

![EACL 정책 목록](/assets/img/2024-04-02/4.png)
_EACL 정책 목록_

<br>

목적지는 192.168.10.0/24 네트워크이기 때문에 Router 0에서 Access-List를 작성해야 한다는 것에 주의하며 정책을 설정한다.

<br>

#### 세부 정책 작성

<br>

> Router 0의 fa0/0에서 진행하는 것에 유의
{: .prompt-warning }

```
R0(config)#access-list 100 permit tcp host 192.168.20.10 host 192.168.10.20 eq 53
R0(config)#access-list 100 permit udp host 192.168.20.10 host 192.168.10.20 eq 53
R0(config)#access-list 100 permit tcp 192.168.20.0 0.0.0.255 host 192.168.10.20 eq www
R0(config)#access-list 100 permit tcp 192.168.20.0 0.0.0.255 host 192.168.10.20 eq 21
R0(config)#access-list 100 permit icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
R0(config)#access-list 100 permit icmp host 192.168.30.10 192.168.10.0 0.0.0.255
R0(config)#access-list 100 permit tcp host 192.168.30.10 host 192.168.10.20 eq 21
R0(config)#access-list 100 permit tcp 192.168.30.0 0.0.0.255 host 192.168.10.20 eq 443
```

<br>

> 한 시스템을 특정하는 경우 `host` 키워드와 함께 **와일드카드 마스크**를 생략한다.
{: .prompt-info }

<br>

#### EACL 테스트

<br>

PC1 -> 192.168.20.10/24

PC2 -> 192.168.20.20/24

위의 두 시스템에서 웹 서버 및 FTP 서버 접속, ICMP 프로토콜을 통한 통신을 진행한다.

<br>

![HTTP 웹 서버 접근 테스트](/assets/img/2024-04-02/5.gif)
_HTTP 웹 서버 접근 테스트_

![FTP 서버 접근 & ping 명령어](/assets/img/2024-04-02/6.gif)
_FTP 서버 접근 & ping 명령어_

<br>

PC3 -> 192.168.30.10/24

위의 시스템에서 ICMP 프로토콜을 통한 통신을 진행한 뒤 FTP 서버에 접속한다.

![PC3 tracert & FTP 서버 접속](/assets/img/2024-04-02/7.gif)
_PC3 tracert & FTP 서버 접속_

<br>

PC3 -> 192.168.30.10/24

PC4 -> 192.168.30.20/24

위의 두 시스템에서 보안 웹 서비스로 접속한다.

![HTTPS 접속](/assets/img/2024-04-02/8.gif)
_HTTPS 접속_

<br>

EACL 테스트를 완료했고, 토폴로지 상태는 다음과 같다.

![토폴로지 구성 완료](/assets/img/2024-04-02/9.png)
_토폴로지 구성 완료_

<br>

---

<br>

## 시나리오 2

<br>

- SVI(Switched Virtual Interface) 활용

    - VLAN(Virtual Local Area Network)에 IP를 할당 후 물리적 인터페이스가 아닌 VLAN에 라우팅 프로토콜을 구동시켜 VLAN끼리 통신을 하게하는 방식으로, VLAN과 라우팅 둘 다 가능한 L3 스위치에서 사용하는 방식이다.

        - VLAN 10

            - fa0/1 => GW : 192.168.10.1/24

            - PC0 IP : 192.168.10.10/24
        
        - VLAN 20

            - fa0/2 => GW : 192.168.20.1/24

            - PC1 IP : 192.168.20.10/24

        - VLAN 30

            - fa0/3 => GW : 192.168.30.1/24

            - PC2 IP : 192.168.30.10/24

<br>

![토폴로지 초기 상태](/assets/img/2024-04-02/10.png)
_토폴로지 초기 상태_

<br>

### 환경 구성

<br>

처음 토폴로지 상태처럼 IP 설정을 한 상태이다.

현재 게이트웨이 구성도 하지 않았기 때문에 서로의 PC는 통신할 수 없다.

<br>

어떤 설정도 하지 않아서 L2 스위치인 멀티레이어 스위치의 fa0/1 ~ fa0/3의 스위치 포트를 access 모드로 변경하고, 해당 포트에 vlan을 할당한다.

```
Switch>enable
Switch#configure terminal
Switch(config)#hostname MLSW

// 인터페이스 모드 변경
MLSW(config)#int range fa0/1 - 3
MLSW(config-if-range)#switchport mode access
MLSW(config-if-range)#exit

// vlan 이름 설정
MLSW(config)#vlan 10
MLSW(config-vlan)#name abc
MLSW(config-vlan)#vlan 20
MLSW(config-vlan)#name def
MLSW(config-vlan)#vlan 30
MLSW(config-vlan)#name ghi
MLSW(config-vlan)#exit

// 인터페이스 vlan 설정
MLSW(config)#int fa0/1
MLSW(config-if)#switchport access vlan 10
MLSW(config-if)#int fa0/2
MLSW(config-if)#switchport access vlan 20
MLSW(config-if)#int fa0/3
MLSW(config-if)#switchport access vlan 30
MLSW(config-if)#exit

// L3 스위치 전환
MLSW(config)#ip routing

// VLAN에 IP 할당
MLSW(config)#int vlan 10
MLSW(config-if)#ip addr 192.168.10.1 255.255.255.0

MLSW(config-if)#int vlan 20
MLSW(config-if)#ip addr 192.168.20.1 255.255.255.0

MLSW(config-if)#int vlan 30
MLSW(config-if)#ip addr 192.168.30.1 255.255.255.0
```

<br>

위와 같이 설정한 뒤에는 서로 통신이 가능한 상태가 된다.

![SVI VLAN 통신 확인](/assets/img/2024-04-02/11.gif)
_SVI VLAN 통신 확인_

<br>

---

<br>

## 시나리오 3

<br>

3개의 라우팅 프로토콜(RIPv2, OSPF, EIGRP)을 사용하며 5개의 라우터 중 2개는 ABR(Area Border Router)로 설정해 3개의 라우팅 프로토콜을 재분배하는 실습을 진행한다.

세부적인 설정보다는 다음의 토폴로지 구성을 확인한다.

<br>

![토폴로지 초기 상태](/assets/img/2024-04-02/12.png)
_토폴로지 초기 상태_

<br>

RIP(Router0) <--> ABR(Router1) <--> OSPF(Router2) <--> ABR(Router3) <--> EIGRP(Router4)

여기서 ABR인 Router1과 Router3에 주의하며 진행한다.

<br>

### Router Interface 설정

<br>

각 라우터들에 연결된 인터페이스들의 IP를 설정한다.

Router 0

```
Router>enable
Router#configure terminal
Router(config)#hostname R0

R0(config)#interface se2/0
R0(config-if)#ip addr 2.2.2.2 255.255.255.0
R0(config-if)#no shutdown

R0(config-if)#interface lo0
R0(config-if)#ip addr 100.100.100.1 255.255.255.0
```

<br>

Router 1

```
Router>en
Router#conf t
Router(config)#hostname R1-ABR

R1-ABR(config)#int se2/0
R1-ABR(config-if)#ip addr 2.2.2.3 255.255.255.0
R1-ABR(config-if)#no shutdown

R1-ABR(config-if)#int se3/0
R1-ABR(config-if)#ip addr 1.1.1.1 255.255.255.0
R1-ABR(config-if)#no sh
```

<br>

Router 2

```
Router>en
Router#conf t
Router(config)#hostname R2

R2(config)#int se2/0
R2(config-if)#ip addr 1.1.1.2 255.255.255.0
R2(config-if)#no sh

R2(config-if)#int se3/0
R2(config-if)#ip addr 2.2.2.1 255.255.255.0
R2(config-if)#no sh
```

<br>

Router 3

```
Router>en
Router#conf t
Router(config)#hostname R3-ABR

R3-ABR(config)#int se2/0
R3-ABR(config-if)#ip addr 2.2.2.2 255.255.255.0
R3-ABR(config-if)#no sh

R3-ABR(config-if)#int se3/0
R3-ABR(config-if)#ip addr 4.4.4.2 255.255.255.0
R3-ABR(config-if)#no sh
```

<br>

Router 4

```
Router>enable
Router#configure terminal
Router(config)#hostname R4

R4(config)#int se2/0
R4(config-if)#ip addr 4.4.4.3 255.255.255.0
R4(config-if)#no shutdown

R4(config-if)#int lo0
R4(config-if)#ip addr 200.200.200.1 255.255.255.0
```

<br>

> Loopback는 `no shutdown` 명령어로 포트를 항상 열어둔 상태로 전환할 필요가 없다.
{: .prompt-tip }

<br>

### 라우팅 프로토콜 설정 및 재분배

<br>

- Router0 => RIPv2

- Router1

    - Se2/0 => RIPv2

        - from RIPv2 to OSPF
    
    - Se3/0 => OSPF Area 0

        - from OSPF to RIPv2

- Router2 => OSPF

- Router3

    - Se2/0 => OSPF

        - from OSPF to EIGRP
    
    - Se3/0 => EIGRP

        - from EIGRP to OSPF

- Router4 => EIGRP

위의 리스트를 유의하며 진행하도록 한다.

<br>

Router 0

```
R0>en
R0#conf t
R0(config)#router rip
R0(config-router)#version 2
R0(config-router)#no auto-summary
R0(config-router)#network 100.100.100.0
R0(config-router)#network 2.2.2.0
```

<br>

Router 1

```
R1-ABR>en
R1-ABR#conf t
R1-ABR(config)#router rip
R1-ABR(config-router)#version 2
R1-ABR(config-router)#no auto-summary
R1-ABR(config-router)#network 2.2.2.0
R1-ABR(config-router)#redistribute ospf 10 metric 2
R1-ABR(config-router)#exit

R1-ABR(config)#router ospf 10
R1-ABR(config-router)#network 1.1.1.0 0.0.0.255 area 0
R1-ABR(config-router)#redistribute rip subnets
```

> from RIPv2 to OSPF일 때는 **metric**값을 주는 것을 잊지 말고, from OSPF to RIPv2일 때는 `subnets`를 반드시 포함한다.
{: .prompt-info }

<br>

Router 2

```
R2>enable
R2#configure terminal
R2(config)#router ospf 10
R2(config-router)#network 1.1.1.0 0.0.0.255 area 0
R2(config-router)#network 2.2.2.0 0.0.0.255 area 0
```

<br>

Router 3

```
R3-ABR>en
R3-ABR#conf t
R3-ABR(config)#router ospf 10
R3-ABR(config-router)#network 2.2.2.0 0.0.0.255 area 0
R3-ABR(config-router)#redistribute eigrp 100 subnets
R3-ABR(config-router)#exit

R3-ABR(config)#router eigrp 100
R3-ABR(config-router)#no auto-summary
R3-ABR(config-router)#network 4.4.4.0 0.0.0.255
R3-ABR(config-router)#redistribute ospf 10 metric 1 1 1 1 1
```

> from OSPF to EIGRP일 때는 EIGRP AS값을 일치시키며 `subnets`를 반드시 포함하고, from EIGRP to OSPF일 때는 **EIGRP의 metric값들**을 반드시 줘야한다.
{: .prompt-info }

<br>

Router 4

```
R4>en
R4#conf t
R4(config)#router eigrp 100
R4(config-router)#no auto-summary
R4(config-router)#network 4.4.4.0 0.0.0.255
R4(config-router)#network 200.200.200.0 0.0.0.255
```

<br>

바로 위까지의 과정을 무리없이 진행했다면, 각 라우터에서 `show ip route` 명령어를 입력하면 다음과 같은 상태가 된다.

![라우팅 테이블 상태](/assets/img/2024-04-02/13.png)
_라우팅 테이블 상태_

<br>

위처럼 설정한 뒤 테스트를 진행한다. Router0에서 Router4까지 tracert를 진행하고, Router4에서 Router0으로 ping을 보낸다.

<br>

![라우터 간 통신 테스트](/assets/img/2024-04-02/14.png)
_라우터 간 통신 테스트_

<br>

![토폴로지 구성 완료](/assets/img/2024-04-02/15.png)
_토폴로지 구성 완료_