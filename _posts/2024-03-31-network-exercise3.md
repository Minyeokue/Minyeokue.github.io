---
title: 네트워크 실습 3 - OSPF 라우팅 프로토콜(-> ABR, ASBR)
excerpt: "네트워크 OSPF 실습 및 영역을 지정해 ABR, ASBR을 설정"
author: minyeokue
date: 2024-03-31 15:25:19 +0900
last_modified_at: 2024-03-31 15:25:23 +0900
categories: [Exercise]
tags: [Network]

toc: true
toc_sticky: true
---

<br>

네트워크 OSPF 실습 및 영역을 지정해 ABR(Area Border Router), ASBR(AS Boundary Router)을 설정한다.

<br>

---

<br>

## 시나리오

<br>

- Area 1

    - Serial line : 10.10.10.1-2/24

        - Router0 -> 10.10.10.1/24

            - 내부 네트워크 GW : 192.168.10.1/24
            - 내부 네트워크 PC 1대 : 192.168.10.10/24
            - Loopback : 120.120.120.1/24

- ABR (Area 1 <---> Area 0)

    - Router 1 -> 10.10.10.2/24, 20.20.20.1/24

- Area 0

    - ASBR Router 3 -> 20.20.20.2/24, 30.30.30.2/24

        - Loopback : 100.100.100.100/24

- ABR (Area 0 <---> Area 2)

    - Router 4 -> 30.30.30.3/24, 40.40.40.1/24

- Area 2

    - Router 5 -> 40.40.40.2/24

        - 내부 네트워크 GW : 172.16.10.1/24
        - 내부 네트워크 PC 1대 : 172.16.10.10/24
        - Loopback : 120.120.120.1/24

<br>

![토폴로지 초기 상태](/assets/img/2024-03-31/1.png)
_토폴로지 초기 상태_

<br>

위 상태에서 설정을 마친 후 PC0에서 웹 서버에 접근해보겠다.

PC0 -> Router0 -> Router1 -> Router3 -> Router4 -> Router5 -> Server0 순으로 설정하겠다.

<br>

![PC0 IP 설정](/assets/img/2024-03-31/2.png)
_PC0 IP 설정_

<br>

### Router 설정

<br>

Router0 설정

```ios
Router>en
Router#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#hostname R0
R0(config)#int fa0/0
R0(config-if)#ip addr 192.168.10.1 255.255.255.0
R0(config-if)#no shutdown

R0(config-if)#
%LINK-5-CHANGED: Interface FastEthernet0/0, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/0, changed state to up

R0(config-if)#int se2/0
R0(config-if)#ip addr 10.10.10.1 255.255.255.0
R0(config-if)#no shutdown

R0(config-if)#
%LINK-5-CHANGED: Interface Loopback0, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface Loopback0, changed state to up

R0(config-if)#ip addr 120.120.120.1 255.255.255.0

%LINK-5-CHANGED: Interface Serial2/0, changed state to down
R0(config-if)#exit
R0(config)#router ospf 101
R0(config-router)#network 192.168.10.0 0.0.0.255 area 1
R0(config-router)#network 10.10.10.0 0.0.0.255 area 1
R0(config-router)#network 120.120.120.0 0.0.0.255 area 1
R0(config-router)#exit
R0(config)#int lo0
```

<br>

Router1 설정

```ios
Router>enable
Router#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#hostname R1
R1(config)#interface se2/0
R1(config-if)#ip addr 10.10.10.2 255.255.255.0
R1(config-if)#no shutdown

R1(config-if)#
%LINK-5-CHANGED: Interface Serial2/0, changed state to up

R1(config-if)#
%LINEPROTO-5-UPDOWN: Line protocol on Interface Serial2/0, changed state to up

R1(config-if)#interface se3/0
R1(config-if)#ip addr 20.20.20.1 255.255.255.0
R1(config-if)#no shutdown

%LINK-5-CHANGED: Interface Serial3/0, changed state to down
R1(config-if)#exit
R1(config)#router ospf 1010
R1(config-router)#network 10.10.10.0 0.0.0.255 area 1
R1(config-router)#network 20.20.20.0 0.0.0.255 area 0
R1(config-router)#end
```

<br>

Router3 설정

```
Router>enable
Router#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#hostname R3
R3(config)#interface se2/0
R3(config-if)#ip addr 20.20.20.2 255.255.255.0
R3(config-if)#no shutdown

R3(config-if)#
%LINK-5-CHANGED: Interface Serial2/0, changed state to up

R3(config-if)#int lo0

R3(config-if)#
%LINK-5-CHANGED: Interface Loopback0, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface Loopback0, changed state to up

R3(config-if)#ip addr 100.100.100.100 255.255.255.0
R3(config-if)#int se3/0
R3(config-if)#ip addr 30.30.30.2 255.255.255.0
R3(config-if)#no shutdown

%LINK-5-CHANGED: Interface Serial3/0, changed state to down
R3(config-if)#exit
R3(config)#router ospf 100
R3(config-router)#network 20.20.20.0 0.0.0.255 area 0
R3(config-router)#network 30.30.30.0 0.0.0.255 area 0
R3(config-router)#network 100.100.100.0 0.0.0.255 area 0
R3(config-router)#end
```

<br>

Router4 설정

```
Router>enable
Router#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#hostname R4
R4(config)#interface se2/0
R4(config-if)#ip addr 30.30.30.3 255.255.255.0
R4(config-if)#no shutdown

R4(config-if)#
%LINK-5-CHANGED: Interface Serial2/0, changed state to up

R4(config-if)#exit
R4(config)#
%LINEPROTO-5-UPDOWN: Line protocol on Interface Serial2/0, changed state to up

R4(config)#int se3/0
R4(config-if)#ip addr 40.40.40.1 255.255.255.0
R4(config-if)#no shutdown

%LINK-5-CHANGED: Interface Serial3/0, changed state to down
R4(config-if)#exit
R4(config)#router ospf 1002
R4(config-router)#network 30.30.30.0 0.0.0.255 area 0
R4(config-router)#network 40.40.40.0 0.0.0.255 area 2
R4(config-router)#end
```

<br>

Router5 설정

```ios
Router>enable
Router#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#hostname 
% Incomplete command.
Router(config)#hostname R5
R5(config)#int se2/0
R5(config-if)#ip addr 40.40.40.2 255.255.255.0
R5(config-if)#no shutdown

R5(config-if)#
%LINK-5-CHANGED: Interface Serial2/0, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface Serial2/0, changed state to up

R5(config-if)#int lo0
%LINK-5-CHANGED: Interface Loopback0, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface Loopback0, changed state to up

R5(config-if)#ip addr 130.130.130.1 255.255.255.0
R5(config-if)#exit
R5(config)#int fa0/0
R5(config-if)#ip addr 172.16.10.1 255.255.255.0
R5(config-if)#no shutdown

R5(config-if)#
%LINK-5-CHANGED: Interface FastEthernet0/0, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/0, changed state to up

R5(config-if)#exit
R5(config)#router ospf 102
R5(config-router)#network 130.130.130.0 0.0.0.255 area 2
R5(config-router)#network 172.16.10.0 0.0.0.255 area 2
R5(config-router)#network 40.40.40.0 0.0.0.255 area 2
R5(config-router)#end

```

<br>

Server0을 DNS 서버이자 Web 서버로 구성하고 IP를 설정한다.

![Server0 IP 설정](/assets/img/2024-03-31/3.png)
_Server0 설정_

<br>

![Server0 DNS 설정](/assets/img/2024-03-31/4.png)
_Server0 DNS 설정_

<br>

이제 PC0에서 웹 서버에 접근한다.

![PC0에서 웹 서버 접근](/assets/img/2024-03-31/5.png)
_PC0에서 웹 서버 접근_

<br>

PC0에서 Server0에 접근하는 과정을 `tracert` 명령어를 통해 출력해보겠다

![PC0 tracert 출력](/assets/img/2024-03-31/6.png)
_PC0 tracert 출력_

<br>

이제 토폴로지 전체를 완성했다.

![토폴로지 구성 완료](/assets/img/2024-03-31/7.png)
_토폴로지 구성 완료_