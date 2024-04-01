---
title: 네트워크 실습 4 - RIPv2 OSPF 재분배 및 서브네팅, ACL을 활용한 보안
excerpt: "RIPv2와 OSPF 재분배 및 서브네팅을 진행한 뒤 EACL로 특정 PC 및 서버만 접근 가능하도록 실습한다."
author: minyeokue
date: 2024-04-01 17:51:56 +0900
last_modified_at: 2024-04-01 21:45:06 +0900
categories: [Exercise]
tags: [Network, Secure]

toc: true
toc_sticky: true
---

<br>

**RIPv2와 OSPF 재분배** 및 **서브네팅**을 진행한 뒤 **EACL**로 특정 PC 및 서버만 접근 가능하도록 실습한다.

<br>

---

<br>

## 시나리오 1

<br>

- Router 0

    - 직접 연결된 네트워크

        - 192.168.10.0/27 => GW : 192.168.10.30/27

            - 사용가능 IP : 1 ~ 30

            - 브로드캐스트 주소 : 192.168.10.31

            - Server0 => 192.168.10.1/24 

                - DNS : www.minyeokue.io -> Server1(192.168.10.33)

                - FTP 서버

                - DHCP 서버 => IP 분배 범위 : 2 ~ 29

            - PC0 => DHCP 클라이언트

        - 192.168.10.32/28 => GW : 192.168.10.46/28

            - 사용가능 IP : 33 ~ 46

            - 브로드캐스트 주소 : 192.168.10.47

            - Server1 => 192.168.10.33/28

                - HTTP / HTTPS 서버

                - Syslog 서버

        - Serial Line : 1.1.1.1/24

            - Router0 -> Router1
        
        - Serial line : 2.2.2.1/24

            - Router0 -> Router2

- Router 1

    - 직접 연결된 네트워크

        - 192.168.10.48/29 => GW : 192.168.10.54/29

            - 사용가능 IP : 49 ~ 54

            - 브로드캐스트 주소 : 192.168.10.55

            - PC1 => 192.168.10.49/29

            - PC2 => 192.168.10.50/29
    
    - Serial Line : 1.1.1.2/24

        - Router1 -> Router0

- Router 2

    - 직접 연결된 네트워크

        - 192.168.10.56/30 => GW : 192.168.10.58/30

            - 사용가능 IP : 57 ~ 58

            - 브로드캐스트 주소 : 192.168.10.59

            - PC3 => 192.168.10.57/29
    
    - Serial Line : 2.2.2.2/24

        - Router2 -> Router0

- EACL 정책

    - Router0

        - Server0 => 192.168.10.1/27 (DNS 서버, DHCP 서버)

            - 192.168.10.48 대역은 DNS 서버 질의 및 ping 명령어 가능

            - 192.168.10.57/30 시스템은 FTP 서버 로그인 가능
        
        - Server1 => 192.168.10.33/28 (웹 서버, Syslog 서버)

            - 192.168.10.48 대역은 HTTP / HTTPS 서버 접근 가능, ping 명령어 가능

            - 192.168.10.57/30 시스템은 HTTPS 접근 가능

-> 192.168.10.48 대역의 시스템들은 웹 서버에 접속할 때 주소창에 `http://www.minyeokue.gitblog`{: .filepath }또는 `https://www.minyeokue.gitblog`{: .filepath } 입력해서 접속 가능

-> 192.168.10.56 대역은 시스템은 웹 서버에 접속할 때 `https://www.minyeokue.gitblog`{: .filepath }로만 접속 가능, FTP 서버 접속 가능

<br>

![토폴로지 구성 초기 상태](/assets/img/2024-04-01/1.png)
_토폴로지 구성 완료_

<br>

### 환경 구성

<br>

먼저 Server0 설정 -> PC1 IP 설정 (DHCP 클라이언트) -> Server1 설정 -> PC1 IP 설정 -> PC2 IP 설정 -> PC3 IP 설정 순서로 시스템 설정을 진행하고, 라우터 설정을 하도록 하겠다.

<br>

**Server0 설정**

![Server0 IP 설정](/assets/img/2024-04-01/2.png)
_Server0 IP 설정_

<br>

다음으로 Server0 서비스들을 설정한다. HTTP/HTTPS 서비스 OFF, DNS 서비스 ON, DHCP 서비스 ON, FTP 서비스 ON.

![Server0 IP 설정](/assets/img/2024-04-01/2.png)
_Server0 IP 설정_

![Server0 HTTP/HTTPS 서비스 설정](/assets/img/2024-04-01/3.png)
_Server0 웹 서비스 설정_

![Server0 DNS 서비스 설정](/assets/img/2024-04-01/4.png)
_Server0 DNS 서비스 설정_

![Server0 DHCP 서비스 설정](/assets/img/2024-04-01/5.png)
_Server0 DHCP 서비스 설정_

![Server0 FTP 서비스 설정](/assets/img/2024-04-01/6.png)
_Server0 FTP 서비스 설정_

<br>

PC0이 DHCP 서버로부터 IP 할당을 정상적으로 받는지 확인

![DHCP 클라이언트 PC0](/assets/img/2024-04-01/7.png)
_DHCP 클라이언트 PC0_

<br>

**Server1 설정**

![Server1 IP 설정](/assets/img/2024-04-01/8.png)
_Server1 IP 설정_

<br>

다음으로 Server1 서비스들을 설정한다. HTTP/HTTPS 서비스 ON, Syslog 서비스 ON, FTP 서비스 OFF.

![Server1 HTTP/HTTPS 서비스 설정](/assets/img/2024-04-01/9.png)
_Server1 HTTP/HTTPS 서비스 설정_

![Server1 Syslog 서비스 설정](/assets/img/2024-04-01/10.png)
_Server1 DHCP 서비스 설정_

![Server1 FTP 서비스 설정](/assets/img/2024-04-01/11.png)
_Server1 FTP 서비스 설정_

<br>

다음으로 PC1과 PC2, PC3 IP 설정 진행

![PC1 IP 설정](/assets/img/2024-04-01/12.png)
_PC1 IP 설정_

![PC2 IP 설정](/assets/img/2024-04-01/13.png)
_PC2 IP 설정_

![PC3 IP 설정](/assets/img/2024-04-01/14.png)
_PC3 IP 설정_

<br>

이제 제일 중요한 라우터들의 인터페이스를 설정하고, 라우팅 프로토콜 설정을 진행하겠다.

<br>

Router0 인터페이스 설정

```
Router>en
Router#configure terminal
Router(config)#hostname R0
# 192.168.10.0/27 GW 설정
R0(config)#int fa0/0
R0(config-if)#ip addr 192.168.10.30 255.255.255.224
R0(config-if)#no shutdown

# 192.168.10.32/28 GW 설정
R0(config-if)#int fa1/0
R0(config-if)#ip addr 192.168.10.46 255.255.255.240
R0(config-if)#no shutdown

# Router1로 연결되는 Se2/0 설정
R0(config-if)#int se2/0
R0(config-if)#ip addr 1.1.1.1 255.255.255.0
R0(config-if)#no shutdown

# Router2로 연결되는 Se3/0 설정
R0(config-if)#interface se3/0
R0(config-if)#ip addr 2.2.2.1 255.255.255.0
R0(config-if)#no shutdown
```

<br>

Router1 인터페이스 설정

```
Router>enable
Router#configure terminal
Router(config)#hostname R1

# 192.168.10.48/29 GW 설정
R1(config)#int fa0/0
R1(config-if)#ip addr 192.168.10.54 255.255.255.248
R1(config-if)#no shutdown

# Router0와 연결되는 Se2/0 설정
R1(config-if)#interface se2/0
R1(config-if)#ip addr 1.1.1.2 255.255.255.0
R1(config-if)#no shutdown
```

<br>

Router2 인터페이스 설정

```
Router>enable
Router#configure terminal
Router(config)#hostname R2

# 192.168.10.56/30 GW 설정
R2(config)#int fa0/0
R2(config-if)#ip addr 192.168.10.58 255.255.255.252
R2(config-if)#no shutdown

# Router0와 연결되는 Se2/0 설정
R2(config-if)#int se2/0
R2(config-if)#ip addr 2.2.2.2 255.255.255.0
R2(config-if)#no shutdown
```

<br>

Router1 라우팅 프로토콜 설정

```
R1(config)#router rip

# VLSM, CIDR 가능한 Classless한 RIP version 2
R1(config-router)#version 2

# 자동 축약기능(Classful) 해제
R1(config-router)#no auto-summary

R1(config-router)#network 192.168.10.48
R1(config-router)#network 1.1.1.0
R1(config-router)#exit
```

<br>

Router0 라우팅 프로토콜 설정 (ABR; Area Border Router)

```
R0(config)#router rip

# VLSM, CIDR 가능한 Classless한 RIP version 2
R0(config-router)#version 2

# 자동 축약기능(Classful) 해제
R0(config-router)#no auto-summary

R0(config-router)#network 192.168.10.0
R0(config-router)#network 1.1.1.0

# OSPF에 재분배
R0(config-router)#redistribute ospf 1
R0(config-router)#exit

# OSPF (Classless)
R0(config)#router ospf 1
R0(config-router)#network 192.168.10.32 0.0.0.31 area 1
R0(config-router)#network 2.2.2.0 0.0.0.255 area 1

# RIP에 서브넷을 포함한 채 재분배
R0(config-router)#redistribute rip subnets
```

> Router version 1의 경우는 재분배할 때 Classful한 프로토콜이기 때문에 `redistribute rip`만 입력하면 되지만, Router version 2의 경우 재분배할 때 Classless한 프로토콜이기 때문에(VLSM, CIDR 가능) `redistribute rip subnets`를 반드시 입력해줘야 한다. => 또한, `no auto-summary` 명령어도 반드시 입력해야 서브넷이 적용된다.
{: .prompt-info }

<br>

Routerk2 라우팅 프로토콜 설정

```
R2(config)#router ospf 1
R2(config-router)#network 192.168.10.56 0.0.0.3 area 1
R2(config-router)#network 2.2.2.0 0.0.0.255 area 1
R2(config-router)#exit
```

<br>

이제 각 라우터에서 라우팅 테이블을 확인하고, 각 PC에 통신이 가능한지 `ping` 명령어로 확인한다.

<br>

![각 라우터 라우팅 테이블](/assets/img/2024-04-01/15.png)
_각 라우터 라우팅 테이블_

<br>

![PC1에서 Server0으로 통신](/assets/img/2024-04-01/16.png)
_PC1에서 Server0으로 통신_

![PC2에서 Server1로 통신](/assets/img/2024-04-01/17.png)
_PC2에서 Server1로 통신_

![PC3에서 Server1까지의 라우터 경로](/assets/img/2024-04-01/18.png)
_PC3에서 Server1까지의 라우터 경로_

<br>

서브네팅과 **라우팅 프로토콜 재분배**가 이뤄졌고, 정상적으로 통신이 가능한 것을 확인했다. 이제 Server0의 DNS 서비스를 통해 Server1의 웹 서버로 접근한다.

<br>

![PC1에서 도메인네임으로 웹 서버 접근](/assets/img/2024-04-01/19.gif)
_PC1에서 도메인네임으로 웹 서버 접근_

사람에게 친숙한 도메인 네임으로 Server1의 웹 서버에 접근했다. DNS 서버가 없었다면 `http://192.168.10.33`{: .filepath }를 직접 입력해야 했을 것이다.  

<br>

이제 Syslog 서버의 작동을 확인해보자. 다음의 명령어를 입력한다.

```
R0>en
R0#conf t
R0(config)#logging on
R0(config)#logging console
R0(config)#logging host 192.168.10.33
```

<br>

Router와 연결된 192.168.10.30/27 게이트웨이를 차단하고 다시 활성화하는 과정에서 log가 저장되는지 확인하면 된다.

![Syslog 서버 정상 작동 확인](/assets/img/2024-04-01/20.gif)
_Syslog 서버 정상 작동 확인_

<br>

위의 시나리오를 참조해 **EACL(Extended Access Control List)**를 실습한다.

<br>

```
# 192.168.10.48 대역의 시스템 -> Server0 DNS 질의, ping 명령어 가능
R0(config)#access-list 100 permit tcp 192.168.10.48 0.0.0.7 192.168.10.0 0.0.0.31 eq 53 
R0(config)#access-list 100 permit udp 192.168.10.48 0.0.0.7 192.168.10.0 0.0.0.31 eq 53 
R0(config)#access-list 100 permit icmp 192.168.10.48 0.0.0.7 192.168.10.0 0.0.0.31

# 192.168.10.56 대역의 시스템 -> Server0 FTP 서버 접근 가능
R0(config)#access-list 100 permit tcp 192.168.10.56 0.0.0.3 192.168.10.0 0.0.0.31 eq 21

# 192.168.10.48 대역의 시스템 -> Server1 HTTP/HTTPS 접속
R0(config)#access-list 110 permit tcp 192.168.10.48 0.0.0.7 192.168.10.32 0.0.0.15 eq 80
R0(config)#access-list 110 permit tcp 192.168.10.48 0.0.0.7 192.168.10.32 0.0.0.15 eq 443

# 192.168.10.56 대역의 시스템 -> Server1 HTTPS 서버 접속 가능
R0(config)#access-list 110 permit tcp 192.168.10.56 0.0.0.3 192.168.10.32 0.0.0.15 eq 443

# 192.168.10.0/27 아웃바운드 설정
R0(config)#int fa0/0
R0(config-if)#ip access-group 100 out

# 192.168.10.32/28 아웃바운드 설정
R0(config-if)#int fa1/0
R0(config-if)#ip access-group 110 out
```

<br>

위의 명령어를 입력 후 테스트를 진행한다. 192.168.10.48 대역에서 DNS를 통해 접속하고, 192.168.10.56 대역에서는 HTTPS로 접속한다 (단, 192.168.10.56 대역은 DNS 질의를 할 수 없으므로 Server1의 IP 주소를 입력한다.)

<br>

![EACL 웹 서버 접속 확인](/assets/img/2024-04-01/21.gif)
_웹 서버 접속 확인_

<br>

다음은 Server0에 `ping` 명령어를 입력한다. 192.168.10.48 대역에서는 ICMP를 허가했기 때문에 ping 명령으로 통신이 가능하지만, 192.168.10.56 대역에서는 `ping` 명령어로 통신이 불가능하다.

![EACL ping 명령어 확인](/assets/img/2024-04-01/22.gif)
_EACL ping 명령어 확인_

<br>

다음은 Server0의 FTP 서버에 접속해본다. 192.168.10.48 대역에서는 FTP 서버 접속 권한이 없기 때문에 접속할 수 없고, 192.168.10.56 대역에서는 접근이 가능하다. (기본으로 설정된 username과 password인 cisco를 입력한다.)

![EACL FTP 서버 접속 확인](/assets/img/2024-04-01/23.gif)
_EACL FTP 서버 접속 확인_

<br>

모든 구성을 마친 토폴로지 사진을 마지막으로 실습을 마치도록 하겠다.

![토폴로지 구성 완료](/assets/img/2024-04-01/24.png)
_토폴로지 구성 완료_