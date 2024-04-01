---
title: 네트워크 실습 2 - RIP 라우팅 프로토콜 활용
excerpt: "네트워크 RIP 동적 라우팅 및 토폴로지 구성"
author: minyeokue
date: 2024-03-28 13:30:51 +0900
last_modified_at: 2024-03-31 15:28:55 +0900
categories: [Exercise]
tags: [Network]

toc: true
toc_sticky: true
---

<br>

라우터 RIP v2 프로토콜을 사용해 동적으로 구현 및 서브네팅, 다른 네트워크의 DNS 서버를 참조해 웹 서버 접근 등 네트워크 개념을 이해하기 위해 실습한다. 

<br>

---

<br>

## 시나리오

<br>

- L3 스위치 0 (10.10.10.1/24)

    - 192.168.10.0/27, GW : 192.168.10.30

        - 서버 0 (192.168.10.1/27)

            - DHCP 서버, DNS 서버, 웹서버
            
                - DHCP IP 할당 범위 2 ~ 29
        
        - DHCP 클라이언트 2대

            - PC 0 (192.168.10.2/27)

            - PC 1 (192.168.10.3/27)

- L3 스위치 1 (10.10.10.2/24, 20.20.20.1/24)

    - 192.168.10.32/28, GW : 192.168.10.46

        - IP 범위 : 33 ~ 46

            - PC 2 (192.168.10.33/28)

- L3 스위치 2 (20.20.20.2/24)

    - 192.168.10.48/29, GW : 192.168.10.54

        - IP 범위 : 49 ~ 54

            - PC 3 (192.168.10.49/29)

<br>

각 라우터는 RIP 2를 사용해 서브네팅을 사용한 동적 라우팅으로 진행하도록 한다.

<br>

---

<br>

![토폴로지 초기 상태](/assets/img/2024-03-28/1.png)
_토폴로지 초기 상태_

<br>

### 192.168.10.1 설정 -> DHCP, DNS, WEB

<br>

서버는 항상 고정 IP로 설정해야 하므로 IP 설정을 다음과 같이 진행한다.

![서버 IP 설정](/assets/img/2024-03-28/2.png)
_서버 IP 설정_

<br>

이후 DNS 서버 설정, DHCP 서버 설정을 다음과 같이 진행한다.

<br>

![DNS 서버 설정](/assets/img/2024-03-28/3.png)
_DNS 서버 설정_

![DHCP 서버 설정](/assets/img/2024-03-28/4.png)
_DHCP 서버 설정_

<br>

위와 같이 설정을 마친 후에는 DHCP 클라이언트의 IP를 할당받는 것을 확인한다.

<br>

![PC0 IP 확인](/assets/img/2024-03-28/5.png)
_PC0 IP 확인_

![PC1 IP 확인](/assets/img/2024-03-28/6.png)
_PC1 IP 확인_

<br>

![PC0 웹서버 확인](/assets/img/2024-03-28/7.png)
_PC0 웹서버 확인_

<br>


### L3 스위치 0 설정 -> SW0

<br>


```

Switch>en
Switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#hostname SW0
SW0(config)#int range fa0/1 - 2
SW0(config-if-range)#no switchport
%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/2, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/2, changed state to up

SW0(config-if-range)#exit
SW0(config)#ip routing

```

<br>

위 명령어들을 입력해 스위치를 L3 스위치로 전환한 이후 각 인터페이스의 IP를 설정한다. 서브넷마스크가 적용된 네트워크 동적 라우팅을 위해 RIPv2를 사용한다.

위 처럼 설정을 마치면 L2 스위치가 아닌 L3 스위치가 된다.

<br>

라우터에서 게이트웨이를 지정하는데 위의 토폴로지에서는 192.168.10.0/27에 접하고 있는 interface가 fa0/2이기 때문에 다음과 같이 설정한다.

```

SW0>en
SW0#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
SW0(config)#int fa0/2
SW0(config-if)#ip addr 192.168.10.30 255.255.255.224

```

<br>

스위치는 라우터랑 달리 `no shutdown` 명령어를 입력할 필요 없다.


```

SW0(config)#int fa0/1
SW0(config-if)#ip addr 10.10.10.1 255.255.255.0
SW0(config-if)#exit
SW0(config)#router rip
SW0(config-router)#version 2
SW0(config-router)#no auto-summary
SW0(config-router)#network 192.168.10.0
SW0(config-router)#network 10.10.10.0
SW0(config-router)#exit

```

<br>

자신과 연결된 네트워크에 대한 정보만 저장한다.

<br>

### L3 스위치 1 설정 -> SW1

<br>

```

Switch>
Switch>en
Switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#hostname SW1
SW1(config)#int range fa0/1 - 3
SW1(config-if-range)#no switchport
%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/2, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/2, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/3, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/3, changed state to up

SW1(config-if-range)#exit
SW1(config)#ip routing

```

<br>

위의 명령어를 입력해 L3 스위치로 전환한 후 다음과 같이 IP 설정을 마친다.

<br>

```

SW1(config)#int fa0/1
SW1(config-if)#ip addr 10.10.10.2 255.255.255.0
SW1(config-if)#int fa0/2
SW1(config-if)#ip addr 20.20.20.1 255.255.255.0
SW1(config-if)#int fa0/1
SW1(config-if)#ip addr 192.168.10.46 255.255.255.240
SW1(config-if)#exit
SW1(config)#router rip
SW1(config-router)#version 2
SW1(config-router)#no auto-summary
SW1(config-router)#network 10.10.10.0
SW1(config-router)#network 20.20.20.0
SW1(config-router)#network 192.168.10.32

```

<br>

자신과 관련된 네트워크에 대한 정보만 저장한다.

<br>

### L3 스위치 2 설정 -> SW2

<br>

```

Switch>en
Switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#hostname SW2
SW2(config)#int range fa0/1 - 2
SW2(config-if-range)#no switchport
%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/2, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/2, changed state to up

SW2(config-if-range)#exit
SW2(config)#ip routing
SW2(config)#int fa0/1
SW2(config-if)#ip addr 20.20.20.2 255.255.255.0
SW2(config-if)#int fa0/2
SW2(config-if)#ip addr 192.168.10.54 255.255.255.248
SW2(config-if)#exit
SW2(config)#router rip
SW2(config-router)#version 2
SW2(config-router)#no auto-summary
SW2(config-router)#network 20.20.20.0
SW2(config-router)#network 192.168.10.48
SW2(config-router)#end

SW2#
%SYS-5-CONFIG_I: Configured from console by console

SW2#show ip route
Codes: C - connected, S - static, I - IGRP, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     10.0.0.0/24 is subnetted, 1 subnets
R       10.10.10.0 [120/1] via 20.20.20.1, 00:00:15, FastEthernet0/1
     20.0.0.0/24 is subnetted, 1 subnets
C       20.20.20.0 is directly connected, FastEthernet0/1
     192.168.10.0/24 is variably subnetted, 3 subnets, 3 masks
R       192.168.10.0/27 [120/2] via 20.20.20.1, 00:00:15, FastEthernet0/1
R       192.168.10.32/28 [120/1] via 20.20.20.1, 00:00:15, FastEthernet0/1
C       192.168.10.48/29 is directly connected, FastEthernet0/2

```

<br>

마지막에 Routing table을 출력해보니 20.20.20.1(SW1)에서 전달받은 정보가 있는 것을 확인할 수 있다.

<br>

PC3에서 ping으로 통신이 가능한지 확인한다.


```

Packet Tracer PC Command Line 1.0
PC>ping 192.168.10.1

Pinging 192.168.10.1 with 32 bytes of data:

Reply from 192.168.10.1: bytes=32 time=0ms TTL=126
Reply from 192.168.10.1: bytes=32 time=0ms TTL=126
Reply from 192.168.10.1: bytes=32 time=4ms TTL=126

Ping statistics for 192.168.10.1:
    Packets: Sent = 3, Received = 3, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 4ms, Average = 1ms

PC>ping 192.168.10.2

Pinging 192.168.10.2 with 32 bytes of data:

Request timed out.
Reply from 192.168.10.2: bytes=32 time=0ms TTL=126
Reply from 192.168.10.2: bytes=32 time=0ms TTL=126
Reply from 192.168.10.2: bytes=32 time=0ms TTL=126

Ping statistics for 192.168.10.2:
    Packets: Sent = 4, Received = 3, Lost = 1 (25% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms

PC>ping 192.168.10.3

Pinging 192.168.10.3 with 32 bytes of data:

Request timed out.
Reply from 192.168.10.3: bytes=32 time=0ms TTL=126
Reply from 192.168.10.3: bytes=32 time=0ms TTL=126
Reply from 192.168.10.3: bytes=32 time=0ms TTL=126

Ping statistics for 192.168.10.3:
    Packets: Sent = 4, Received = 3, Lost = 1 (25% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms

PC>ping 192.168.10.33

Pinging 192.168.10.33 with 32 bytes of data:

Reply from 192.168.10.33: bytes=32 time=5ms TTL=128
Reply from 192.168.10.33: bytes=32 time=5ms TTL=128
Reply from 192.168.10.33: bytes=32 time=4ms TTL=128
Reply from 192.168.10.33: bytes=32 time=4ms TTL=128

Ping statistics for 192.168.10.33:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 4ms, Maximum = 5ms, Average = 4ms

```

<br>

PC3에서 192.168.10.1의 웹서버에 접근해보겠다.

![PC3 웹서버 접근](/assets/img/2024-03-28/8.png)
_PC3 웹서버 접근_