---
title: AWS Site-to-Site VPN을 통한 하이브리드 클라우드 구축 프로젝트
excerpt: AWS S2S VPN을 통한 하이브리드 클라우드 구축 시 Terraform과 Ansible로 자동화한 과정입니다.
author: minyeokue
date: 2024-09-27 11:54:28 +0900
last_modified_at: 2024-09-30 15:07:22 +0900
categories: [Project]
tags: [AWS, VPN, IPSec, FRR, Terraform, Ansible]

toc: true
toc_sticky: true
---

<br>

2024년 7월 8일 ~ 2024년 8월 5일까지 진행한 하이브리드 클라우드 구축 프로젝트 진행과정 중 일부를 포트폴리오로 제출하기 위해 작성되었습니다.

팀 구성원으로는 팀장 고민혁, 김성훈, 김세벽, 오주화 총 4명이서 진행하였습니다

<br>

---

### 목차

<br>

1. [개요](#1-개요)
<br>

2. Terraform AWS 리소스 생성 <br>
  2-1. [VPC 및 VPN 생성](#2-1-vpc-및-vpn-생성) <br>
  2-2. [AWS EKS Addons](#2-2-aws-eks-addons)
<br>

3. VPN 연결 <br>
  3-1. [IPSec 구성 및 자동화](#4-1-ipsec-구성-및-자동화) <br>
  3-2. [FRR(Free Range Routing) 구성 및 자동화](#4-2-frr-구성-및-자동화)
<br>

4. [테스트](#테스트)
<br>

5. Trouble Shooting <br>
    5-1. [서버와 시스템 시간 불일치 문제](#5-1-서버와-시스템-시간-불일치-문제) <br>
    5-2. [EKS 클러스터 생성 권한 미지정 문제](#5-2-eks-클러스터-생성-권한-미지정-문제) <br>
    5-3. [AWS 라우팅 테이블 미등록 문제](#5-3-aws-라우팅-테이블-미등록-문제)
<br>

---

### 1. 개요

<br>

프로젝트명은 *티켓 예매 서비스 확장을 위한 **하이브리드 클라우드 환경 설계 및 구축***이며, 시나리오 고객의 첫 요구사항이 온프레미스 쿠버네티스 환경으로의 전환이었고, 그에 따른 저희의 온프레미스 쿠버네티스 솔루션은 다음과 같습니다.

![온프레미스 쿠버네티스 아키텍처](/assets/img/2024-09-27/1.png)
_온프레미스 쿠버네티스 아키텍처_

<br>

이후 프로젝트 시나리오에서 급격한 트래픽으로 인해 다운타임이 발생하였고, 다운타임 재발 방지를 위해 온프레미스 쿠버네티스에서 **AWS EKS로 마이그레이션을 결정**하였습니다.

만약 온프레미스 쿠버네티스 시스템을 확장한다면, 추가적인 비용과 시간이 소요되는 점과 그동안 안정적인 서비스 제공이 어려울 수 있기 때문입니다.

단, 온프레미스의 DB 클러스터는 유지하기로 결정하였는데, 그 이유는 이미 투자된 비용이 있다는 점과 **ProxySQL**을 활용해 *트랜잭션을 읽기와 쓰기로 분리하여 DB 클러스터에 가해지는 부하를 분산*하고 있으며, **Orchestrator**로 *DB 토폴로지 시각화와 함께 장애 복구 조치가 구성*되어 있기 때문입니다.

<br>

위의 사항들을 고려하여 구성한 최종 아키텍처는 다음과 같습니다.

![하이브리드 클라우드 아키텍처](/assets/img/2024-09-27/2.svg)
_하이브리드 클라우드 아키텍처_

<br>

AWS의 대부분의 리소스는 Terraform을 통해 프로비저닝하였으며 상세 정보는 `.tfstate`{:filepath}에 저장됩니다. 온프레미스의 DB 클러스터를 유지하겠다는 결정에 따라 AWS Site-to-Site VPN을 활용해 온프레미스와 AWS VPC를 연결합니다.

Customer Gateway는 VPN 서버를 의미하며, VPN 서버는 IPSec 구성과 라우팅 설정 등을 Ansible을 통해 자동화를 진행하였습니다.

IPSec 구성 정보를 정의하는 데 사용된 도구는 **Libreswan**이며, 프로젝트 진행 당시 물리적인 네트워크 장비에 접근할 수 없었기 때문에 **FRR(Free Range Routing)** 도구를 사용해 소프트웨어 라우팅을 구현하였습니다.

소프트웨어 라우팅을 진행할 때, 동적 라우팅 프로토콜인 OSPF 프로토콜과 BGP 프로토콜을 통해 서로 다른 네트워크인 온프레미스 네트워크와 AWS VPC 간 통신이 원활하게 이루어질 수 있도록 하였습니다.

마지막으로, 팀원 중 CI/CD 개발자가 EKS에서 빌드와 배포를 진행할 때, **Load Balancer Controller**라는 특정 파드가 호스팅 영역에 속하는 ALB(Application Load Balancer)를 생성하면 **External-DNS** 파드가 AWS Route 53에 DNS 레코드를 자동으로 등록할 수 있도록 구성하였습니다.

---

### 2. Terraform AWS 리소스 생성

<br>

#### 2-1. VPC 및 VPN 생성

<br>

먼저, 가장 기본적인 VPC 구성정보에 대한 Terraform 코드입니다.

```tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = local.cluster_vpc_name

  cidr            = var.network_cidr # 172.20.0.0/16
  azs             = slice(data.aws_availability_zones.available.names, 0, 3)
  private_subnets = [cidrsubnet(var.network_cidr, 8, 12), cidrsubnet(var.network_cidr, 8, 24), cidrsubnet(var.network_cidr, 8, 36)]
  public_subnets  = [cidrsubnet(var.network_cidr, 8, 112), cidrsubnet(var.network_cidr, 8, 124), cidrsubnet(var.network_cidr, 8, 136)]
  # private_subnets = ["172.20.12.0/24",  "172.20.24.0/24",  "172.20.36.0/24"]
  # public_subnets  = ["172.20.112.0/24", "172.20.124.0/24", "172.20.136.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  propagate_public_route_tables_vgw  = true
  propagate_private_route_tables_vgw = true

  public_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                      = 1
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"             = 1
  }
}
```

온프레미스의 네트워크 대역은 **192.168.2.0/24** 대역이며, 위의 Terraform 코드에서 AWS VPC 네트워크 대역은 **172.20.0.0/16**입니다.

위 코드 중 *propagate_public_route_tables_vgw* 필드와 *propagate_private_route_tables_vgw* 필드의 값을 true로 설정함으로써 AWS VPC의 퍼블릭 및 프라이빗 라우팅 테이블에 온프레미스 네트워크 정보가 자동으로 등록될 수 있습니다.

<br>

위 코드로 생성된 VPC 아래에서 EKS 클러스터가 생성됩니다. 그리고 EKS의 Addon을 생성하는 부분은 [Trouble Shooting](#3-2-aws-eks-addons)에서 진행하겠습니다.

다음으로는 VPN과 관련된 Terraform 코드입니다.

<br>

```tf
resource "aws_customer_gateway" "tc-vpc-cgw" {
  depends_on = [module.vpc]
  bgp_asn    = 65000
  ip_address = var.onpremise_ip # 온프레미스 외부 공인 
  type       = "ipsec.1"

  tags = {
    Name = "tc-vpc-cgw"
  }
}

resource "aws_vpn_gateway" "tc-vpc-vgw" {
  depends_on      = [module.vpc]
  amazon_side_asn = 64512
  vpc_id          = module.vpc.vpc_id

  tags = {
    Name = "tc-vpc-vgw"
  }
}

resource "aws_vpn_gateway_attachment" "tc-vpn-attach" {
  depends_on     = [module.vpc, aws_vpn_gateway.tc-vpc-vgw]
  vpc_id         = module.vpc.vpc_id
  vpn_gateway_id = aws_vpn_gateway.tc-vpc-vgw.id
}

resource "aws_vpn_gateway_route_propagation" "tc-vpn-propagate-to-pub" {
  vpn_gateway_id = aws_vpn_gateway.tc-vpc-vgw.id
  route_table_id = module.vpc.public_route_table_ids[0]
}

resource "aws_vpn_gateway_route_propagation" "tc-vpn-propagate-to-priv" {
  vpn_gateway_id = aws_vpn_gateway.tc-vpc-vgw.id
  route_table_id = module.vpc.private_route_table_ids[0]
}
```

위 코드 중 **aws_vpn_gateway_attachment** 리소스는 VPC와 VPN Gateway를 연결하는 설정으로 연결되지 않으면 문제가 발생합니다. 관련된 부분은 [AWS 라우팅 테이블 미등록 문제](#5-3-aws-라우팅-테이블-미등록-문제)에서 더 자세히 설명하겠습니다.

AWS의 VGW와 온프레미스 측의 CGW를 생성합니다. 그리고 퍼블릭 및 프라이빗 라우팅 테이블에 자동으로 전파하는 옵션을 설정합니다. 이 옵션이 설정되어 있지 않다면, AWS VPC의 라우팅 테이블에 온프레미스 네트워크 정보가 자동으로 갱신되지 않습니다.

<br>

#### 2-2. AWS EKS Addons

<br>

Terraform으로 LoadBalancer Controller와 External-DNS를 정의하는 코드입니다.

```terraform
# -----------------------------------------------------------------
# Addon - AWS Load Balancer Controller
# -----------------------------------------------------------------
# https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html
module "alb_controller_irsa" {
  depends_on = [module.eks]
  source     = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name                              = "alb_controller_role"
  attach_load_balancer_controller_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}

resource "kubernetes_service_account" "alb_controller" {
  metadata {
    name      = "aws-load-balancer-controller"
    namespace = "kube-system"

    annotations = {
      "eks.amazonaws.com/role-arn" = module.alb_controller_irsa.iam_role_arn
    }
  }
}

resource "helm_release" "aws-load-balancer-controller" {
  depends_on = [kubernetes_service_account.alb_controller]
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = ">= 1.8.0"

  values = [
    templatefile("${path.module}/alb_controller.yaml", {
      clusterName = module.eks.cluster_name,
      region      = var.region,
      vpcId       = module.vpc.vpc_id
      }
    )
  ]
}
```
_loadbalancer_controller_

로드밸런서 컨트롤러를 RBAC(Role-Based Access Controll)으로 최소 권한 원칙을 지키며 생성합니다.

이 때, Helm 차트를 통해 EKS 노드 위에 로드밸런서 컨트롤러 파드가 배포되고 쿠버네티스 서비스 계정으로 로드밸런서를 생성할 수 있게 됩니다.

<br>

```terraform
module "external-dns-irsa-role" {
  depends_on = [module.eks]
  source     = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name                     = "external-dns-irsa-role"
  attach_external_dns_policy    = true
  force_detach_policies         = false
  external_dns_hosted_zone_arns = ["arn:aws:route53:::hostedzone/*"]

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:external-dns"]
    }
  }
}

resource "helm_release" "external-dns-helm" {
  depends_on = [module.external-dns-irsa-role]
  name       = "external-dns"
  repository = "https://kubernetes-sigs.github.io/external-dns/"
  chart      = "external-dns"
  namespace  = "kube-system"
  version    = ">= 1.14.5"

  values = [
    templatefile("${path.module}/external_dns.yaml", {
      serviceAccount = module.external-dns-irsa-role.iam_role_arn
      }
    )
  ]
}
```
_external_dns_

비슷한 방식으로 AWS Route 53의 호스팅 영역에 레코드를 작성할 수 있는 External-DNS 파드를 Helm 차트를 통해 배포하고 필요한 권한을 위해 RBAC을 이용하였습니다.

<br>

쿠버네티스 클러스터의 정보를 수집하여 kube-api로 전달하는 Metric Server와 EBS CSI Driver는 좀 더 간단한 형식으로 선언됩니다.

```tf
# -----------------------------------------------------------------
# Addon - EBS CSI Driver
# -----------------------------------------------------------------
data "aws_iam_policy" "ebs_csi" {
  name = "AmazonEBSCSIDriverPolicy"
}

resource "kubernetes_storage_class" "ebs-gp3-sc" {
  depends_on = [module.eks]
  metadata {
    name = "ebs-gp3-sc"
  }
  storage_provisioner = "ebs.csi.aws.com"
  reclaim_policy      = "Delete"
  volume_binding_mode = "WaitForFirstConsumer"
  parameters = {
    "csi.storage.k8s.io/fstype" = "ext4"
    type                        = "gp3"
  }
}

# -----------------------------------------------------------------
# Addon - Metric Server
# -----------------------------------------------------------------
# https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/metrics-server.html
resource "helm_release" "metrics-server" {
  depends_on = [module.eks]
  name       = "metrics-server"
  repository = "https://kubernetes-sigs.github.io/metrics-server/"
  chart      = "metrics-server"
  namespace  = "kube-system"
  version    = ">= 3.12.0"
}
```

다음은 `terraform apply` 명령어로 실제 리소스를 생성하는 영상이며, 생성 시간이 15분 이상 걸리는 관계로 배속으로 진행하였습니다.

<video width="100%" preload="auto" muted controls>
  <source src="/assets/mp4/terraform-apply.mp4" type="video/mp4">
</video>

outputs.tf 파일을 정의해 `terraform apply`를 통해 생성한 AWS 리소스에 대한 정보 중 IPSec 및 라우팅에 필요한 정보를 tfstate 파일에 저장합니다.

이후 진행할 [IPSec 구성 및 자동화](#4-1-ipsec-구성-및-자동화), [FRR(Free Range Routing) 구성 및 자동화](#4-2-frr-구성-및-자동화)에서 tfstate 파일은 해당 Ansible 구성 정보 자동화의 입력으로 전환됩니다.

---

### 3. VPN 연결

<br>

#### 3-1. IPSec 구성 및 자동화

<br>

```yaml
---
- name: Configure settings for IPSec
  hosts:
    - VPN_Server
  vars_files:
    - "{{ home }}host_vars/vpn.yaml"
  vars:
    home: /home/user/automation/ansible/
  tasks:
    - name: Install libreswan for IPSec and jmespath for json_query
      dnf:
        name: libreswan, python3-jmespath
        state: present

    # Setting the "rp_filter" variable in "sysctl",
    # which can be changed to not block packets that are not registered in the Linux route table.

    # Copy the corresponding configuration file template to the "sysctl" configuration file directory location.
    
    - name: Copy forward.conf file to the sysctl.d
      template:
        src: "{{ ipsec_sysctl_src }}"
        dest: "{{ ipsec_sysctl_dest }}"
        owner: root
        group: root
        mode: '0600'

    - name: Apply the "sysctl" configuration file
      command: "sysctl -p {{ ipsec_sysctl_dest }}"

    # from .tfstate to register
    - name: Read .tfstate to register
      slurp:
        src: "{{ outputs }}"
      register: tfstate
      no_log: yes

    - name: Setting variables for AWS Site-to-Site VPN
      set_fact:
        tunnel1_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_address.value') }}"
        tunnel1_cgw_inside_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_cgw_inside_address.value') }}"
        tunnel1_preshared_key: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_preshared_key.value') }}"
        # tunnel1_inside_cidr: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_inside_cidr.value') }}"
        # tunnel1_vgw_inside_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_vgw_inside_address.value') }}"
        # tunnel1_preshared_key: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_preshared_key.value') }}"
        tunnel2_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_address.value') }}"
        tunnel2_cgw_inside_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_cgw_inside_address.value') }}"
        tunnel2_preshared_key: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_preshared_key.value') }}"
        # tunnel2_inside_cidr: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_inside_cidr.value') }}"
        # tunnel2_vgw_inside_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_vgw_inside_address.value') }}"
        # tunnel2_preshared_key: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_preshared_key.value') }}"
      no_log: yes
    
    # AWS Site-to-Site VPN Tunnel1 & Tunnel2 Preshared Key file
    - name: Copy aws.secrets file to the ipsec.d
      template:
        src: "{{ ipsec_secrets_src }}"
        dest: "{{ ipsec_secrets_dest }}"

    # AWS Site-to-Site VPN Tunnel1 & Tunnel2 Configuration
    - name: Copy aws.conf file to the ipsec.d
      template:
        src: "{{ ipsec_conf_src }}"
        dest: "{{ ipsec_conf_dest }}"

    - name: Gather ansible_facts services
      service_facts:
      no_log: yes

    # Check Firewall Service is available
    # If the status of firewalld is on, configure the firewall's settings.
    # ipsec service include 500/udp, 4500/udp, 4500/tcp ports and esp, ah protocols (50, 51)
    - name: IPSec Firewall port open
      firewalld:
        service: ipsec 
        state: enabled
        permanent: yes
        immediate: yes
      when: ansible_facts.services['firewalld.service'].state == 'running'

    - name: Enable ipsec.service
      service:
        name: ipsec
        state: restarted
        enabled: yes
      ignore_errors: yes

```
_ipsec.yaml_

<br>

리눅스 커널에서 IPSec에 필요한 설정을 적용하기 위해 `sysctl -p [설정 파일 위치]` 명령어를 수행합니다.

그리고 tfstate 파일을 읽어 JSON 쿼리를 통해 해당 정보를 변수에 담아 템플릿으로 설정 파일이 존재해야 하는 위치로 복사하게 됩니다.

```text
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.*.rp_filter = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
```
_ipsec-sysctl.conf_ 

*net.ipv4.ip_forward* 설정은 IP 포워딩을 사용했을 때 패킷이 한 네트워크 인터페이스에서 다른 인터페이스(VTI로 전달)로 전달될 때 사용하는 설정입니다.

위의 설정들은 VPN 연결에 있어 정상적인 트래픽이어도 차단할 가능성이 존재하기 때문에 VPN 트래픽 흐름을 안정적으로 흐르게 하기 위해 활성화하거나 비활성화합니다.

<br>

```text
conn aws-tunnel1
    authby=secret
    auto=start
    left=%defaultroute
    leftid={{ on_premise_public_address }}
    right={{ tunnel1_address }}
    type=tunnel
    ikelifetime=8h
    keylife=1h
    ike=aes256-sha2_256;dh14
    phase2alg=aes_gcm256;modp1536
    ikev2=insist
    keyingtries=%forever
    keyexchange=ike
    leftsubnet=0.0.0.0/0
    rightsubnet=0.0.0.0/0
    dpddelay=10
    dpdtimeout=30
    dpdaction=restart_by_peer
    aggrmode=no
    rekey=yes
    encapsulation=yes
    mark=5/0xffffffff
    vti-interface=vti1
    vti-routing=no
    leftvti={{ tunnel1_cgw_inside_address }}/30

...
```
_aws.conf_

같은 방식으로 tunnel2에 대한 정의도 진행되지만 유사하기 때문에 생략하였습니다.

위 설정 파일은 PSK(Pre-Shared Key)의 유효 시간, 암호화 방식 등을 정의하는 설정 파일입니다.

AWS에서 제공하는 터널을 위한 외부 공인 IP와 온프레미스 공인 IP에서 위의 정보들로 안전한 터널을 생성하게 됩니다.

<br>

다음은 Ansible로 IPSec 구성 자동화를 진행하는 영상입니다.

<video width="100%" preload="auto" muted controls>
  <source src="/assets/mp4/ansible-ipsec.mp4" type="video/mp4">
</video>

모든 구성이 완료된 후 `ipsec verify` 명령어를 입력했을 때, 모든 결과가 반드시 OK로 출력되어야 합니다.

<br>

#### 3-2. FRR 구성 및 자동화

<br>

```yaml
---
- name: Configure settings for Free Range Routing server
  hosts:
    - VPN_Server
  vars_files:
    - "{{ home }}host_vars/vpn.yaml"
  vars:
    home: /home/user/automation/ansible/
    FRRVER: frr-9
  tasks:

    - name: Install Free Range Routing(FRR) repo rpm
      dnf:
        name: "{{ frr_repo_pkg_url }}"
        state: present
        disable_gpg_check: true
    
    - name: Copy ospf_sysctl.conf file to the sysctl.d
      template:
        src: "{{ ospf_sysctl_conf_src }}"
        dest: "{{ ospf_sysctl_conf_dest }}"
        owner: root
        group: root
        mode: '0600'

    - name: Apply the "sysctl" configuration file for ospf dynamic routing
      command: "sysctl -p {{ ospf_sysctl_conf_dest }}"

    - name: Install frr
      dnf:
        name: frr
        state: present

    - name: Enable bgpd and ospfd daemons
      shell: "{{ daemons }}"
      loop:
        - "sudo sed -i 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons"
        - "sudo sed -i 's/ospfd=no/ospfd=yes/g' /etc/frr/daemons"
      loop_control:
        loop_var: daemons

    - name: Copy vtysh.conf file to the /etc/frr
      template:
        src: "{{ vtysh_conf_src }}"
        dest: "{{ vtysh_conf_dest }}"
        owner: frr
        group: frr
        mode: '0600'
    
    - name: Read .tfstate to register
      slurp:
        src: "{{ outputs }}"
      register: tfstate
      no_log: yes
    
    - name: Setting variables for AWS Site-to-Site VPN
      set_fact:
        tunnel1_inside_cidr: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_inside_cidr.value') }}"
        tunnel1_vgw_inside_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_vgw_inside_address.value') }}"
        tunnel1_cgw_inside_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_cgw_inside_address.value') }}"
        # tunnel1_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_address.value') }}"
        # tunnel1_preshared_key: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel1_preshared_key.value') }}"
        tunnel2_inside_cidr: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_inside_cidr.value') }}"
        tunnel2_vgw_inside_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_vgw_inside_address.value') }}"
        tunnel2_cgw_inside_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_cgw_inside_address.value') }}"
        # tunnel2_address: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_address.value') }}"
        # tunnel2_preshared_key: "{{ (tfstate.content | b64decode) | from_json | json_query('outputs.tunnel2_preshared_key.value') }}"
      no_log: yes

    - name: Copy vtysh command sample file to home
      template:
        src: "{{ vtysh_command_src }}"
        dest: "{{ vtysh_command_sample }}"
    
    # - name: Configure global bgp as 65000 for aws
    #   frr.frr.frr_bgp:
    #     config:
    #       bgp_as: 65000
    #       router_id: "{{ tunnel1_cgw_inside_address }}"
    #       log_neighbor_changes: true
    #       neighbors:
    #       - neighbor: "{{ tunnel1_vgw_inside_address }}"
    #         remote_as: 64512
    #         ebgp_multihop: 255
    #         enabled: true
    #       - neighbor: "{{ tunnel2_vgw_inside_address }}"
    #         remote_as: 64512
    #         ebgp_multihop: 255
    #         enabled: true
    #       address_family:
    #       - afi: ipv4
    #         safi: unicast
    #         neighbors:
    #         - neighbor: "{{ tunnel1_inside_cidr }}"
    #           activate: true
    #         - neighbor: "{{ tunnel2_inside_cidr }}"
    #           activate: true
    #         - neighbor: "{{ on_premise_cidr }}"
    #           activate: true
    #           route_reflector_client: true
    #         redistribute:
    #         - protocol: ospf
    #           id: 0

    - name: Gather ansible_facts services
      service_facts:
      no_log: yes

    # Check Firewall Service is available
    # If the status of firewalld is on, configure the firewall's settings.
    - name: FRR VPN BGP service open (179/tcp)
      firewalld:
        service: bgp
        state: enabled
        permanent: yes
        immediate: yes
      when: ansible_facts.services['firewalld.service'].state == 'running'

    - name: Firewall protocol icmp open for test network 
      firewalld:
        protocol: icmp
        state: enabled
        permanent: yes
        immediate: yes
      when: ansible_facts.services['firewalld.service'].state == 'running'

    - name: Firewall add interface vti1, vti2 at public zone
      firewalld:
        zone: public
        permanent: yes
        immediate: yes
        state: enabled
        interface: "{{ tunnel_interface }}"
      loop:
        - vti1
        - vti2
      loop_control:
        loop_var: tunnel_interface

    - name: Enable frr.service and firewalld.service
      service:
        name: "{{ frr_services }}"
        state: restarted
        enabled: yes
      loop:
        - frr
        - firewalld
      loop_control:
        loop_var: frr_services
      ignore_errors: yes
```

VPN 서버의 방화벽의 Public 영역에 기존에 존재하던 인터페이스 이외에 vti(Virtual Tunnel Interface)를 추가해줌으로써 방화벽의 설정을 터널 인터페이스에도 적용할 수 있습니다.

IPSec 구성에서 커널 설정이 필요했던 것처럼 라우팅에도 필요한 리눅스 커널 설정이 존재하기 때문에 유사한 방식으로 진행해 적용합니다.

<br>

다음은 Ansible로 FRR 구성을 자동화하며 tfstate 파일을 읽은 정보를 토대로 명령어 입력 순서대로 정리한 파일을 출력하며, 관련 설정을 진행하는 영상입니다.

<video width="100%" preload="auto" muted controls>
  <source src="/assets/mp4/ansible-frr.mp4" type="video/mp4">
</video>

<br>

이제 Ansible 플레이북의 출력으로 생성된 파일로 라우터 명령어를 입력하는 단계입니다. 라우터 명령어 입력까지도 Ansible로 자동화하는 것은 계속해서 발전 중에 있습니다.

다음은 VPN 연결을 위해 입력해야할 라우터 명령어를 정리한 것입니다.

```bash
sudo vtysh

configure terminal

router bgp 65000

  bgp router-id {{ tunnel1_cgw_inside_address }}
  
  bgp log-neighbor-changes
  
  no bgp ebgp-requires-policy
  
  bgp graceful-restart
  
  neighbor {{ tunnel1_vgw_inside_address }} remote-as 64512
  neighbor {{ tunnel1_vgw_inside_address }} ebgp-multihop 255
  neighbor {{ tunnel2_vgw_inside_address }} remote-as 64512
  neighbor {{ tunnel2_vgw_inside_address }} ebgp-multihop 255
  
  address-family ipv4 unicast
  
    redistribute ospf
    
    network {{ tunnel1_inside_cidr }}
    network {{ tunnel2_inside_cidr }}
    network {{ on_premise_cidr }}
    
    exit

  exit

router ospf

  network {{ aws_vpc_cidr }} area 0
  network {{ on_premise_cidr }} area 0

  exit

exit

write file
```

위 라우터 명령어를 입력한 결과는 다음과 같습니다.

<video width="100%" preload="auto" muted controls>
  <source src="/assets/mp4/vtysh.mp4" type="video/mp4">
</video>

---

### 4. 테스트

IPSec 구성이 완료되고, `vtysh`를 통해 라우터 명령어까지 입력했다면 AWS 콘솔을 통해 연결이 잘 이루어졌는지 확인할 수 있습니다.

![AWS VPN 연결 성공](/assets/img/2024-09-27/4.gif)
_AWS VPN 연결 성공_

<br>

이후 VPN 연결을 통해 온프레미스 네트워크(192.168.2.0/24) 대역에서 AWS CIDR(172.20.0.0/16)로 통신이 가능한지 테스트합니다.

<video width="100%" preload="auto" muted controls>
  <source src="/assets/mp4/ping-test.mp4" type="video/mp4">
</video>

<br>

---

### 5. Trouble Shooting

<br>

#### 5-1. 서버와 시스템 시간 불일치 문제

![시간 불일치 확인](/assets/img/2024-09-27/5.png)

위 사진은 VS Code에서 SSH를 통해 VPN 서버에 접속한 모습인데, 리눅스의 시스템 시간과 호스트 OS의 시스템 시간이 일치하지 않아 SignatureDoesNotMatch 에러가 발생하고 있습니다.

`aws sts get-caller-identity` 명령어는 AWS에서 작업을 호출하는 데 사용되는 자격 증명을 가진 IAM 사용자 또는 역할에 대한 세부 정보를 검색하는 데 사용됩니다.

해당 정보에 접근하기 위해서 `aws configure` 명령으로 특정 계정에 대한 액세스 키와 비밀 액세스 키를 등록합니다.

위 에러는 해당 키를 이용해 접근한 **사용자의 정보를 가져오는 도중, AWS 서버의 시간과 해당 요청을 보낸 시스템의 시간이 달라서 발생**하는 에러입니다.

<br>

따라서 시간을 일치시키기 위해 `chronyd` 또는 `ntpd` 등의 도구를 설치해야 합니다.

저는 이 문제를 해결하기 위해 `chronyd`를 설치하기로 결정했고 그 이유는 `chronyd`가 `ntpd`에 비해 정확성과 속도가 빠르며, `chronyd`는 `ntpd`와 달리 여러 시간 서버에 동시 요청을 지원하기 때문에 일관성 유지에 도움이 됩니다.

마지막으로 `chronyd`는 `ntpd`에 비해 시스템 자원을 더 적게 사용하기 때문입니다.

다음은 입력한 명령어의 목록입니다.

```bash
# chrony(ntp) 설치 및 시작 등록
sudo dnf install chrony -y

# 현재 적용 중인 서버 풀 및 서버 등록을 해제하기 위함
sed -i 's/^pool/#pool/g' /etc/chrony.conf
sed -i 's/^server/#server/g' /etc/chrony.conf

# /etc/chrony.conf 맨 아래에 time.bora.net, time.google.com 추가
cat <<EOF | sudo tee /etc/chrony.conf
server time.bora.net
server time.google.com
EOF

# 서비스 시작
sudo systemctl enable --now chronyd

# 시간 서버 정보 확인
chronyc sources -v

# 리눅스 시간을 시간 서버의 시간으로 동기화
chronyc tracking
```

시간 서버 데몬을 실행시켜도 시간이 동기화되지 않을 수 있기 때문에, `chronyc tracking` 명령어를 입력해야 합니다.

<br>

#### 5-2. EKS 클러스터 생성 권한 미지정 문제

![권한 획득 실패](/assets/img/2024-09-27/6.png)
_권한 획득 실패_

권한을 얻는 과정에서 실패하는 것을 확인할 수 있다.

이는, Terraform 모듈 중 EKS를 설정할 때 여러 테스트를 진행하다가 필수 설정을 제외하면서 발생한 오류이다.

<br>

![클러스터 생성 권한 필수 설정](/assets/img/2024-09-27/7.png)
_클러스터 생성 권한 필수 설정_

`enable_cluster_creator_admin_permissions = true` boolean 값을 true로 설정하면, **eks 모듈을 통해 eks 클러스터를 생성하는 사용자에게도 AmazoneEKSClusterAdminPolicy 를 부여**한다.

아래의 `access_entries` EKS 클러스터를 생성하며 함께 작업할 동료에게 권한을 부여할 때 지정하는 필드를 의미한다.

<br>

#### 5-3. AWS 라우팅 테이블 미등록 문제

![VPN Gateway 미연결](/assets/img/2024-09-27/8.png)
_VPN Gateway 미연결_

위 사진에서 VPN 게이트웨이가 NotAttached 되어 있다고 에러 메시지가 나오는데, 이는 Terraform에서 VPC 리소스와 VPN 리소스를 정의하는 부분에서 해결할 수 있다.

<br>

![VPN 필수 설정](/assets/img/2024-09-27/9.png)
_VPN 필수 설정_

위 사진은 Terraform에서 VPC를 정의할 때 VPN 연결을 위해 필수적인 설정입니다.

<br>

![Attachment](/assets/img/2024-09-27/10.png)
_Attachment_

이 문제가 발생한 근본적인 원인으로, VPN 게이트웨이가 연결되지 않았었는데, 위 사진처럼 리소스를 정의하면 해결할 수 있습니다.

`module.vpc.private_route_table_ids` 항목은 리스트 형태로 전달되기 때문에, 인덱싱을 통해 첫 번째 요소를 지정함으로써 퍼블릭 서브넷에 할당되는 라우팅 테이블과, 프라이빗 서브넷에 할당되는 라우팅 테이블에 온프레미스 네트워크 정보를 자동으로 전파할 수 있게 됩니다.