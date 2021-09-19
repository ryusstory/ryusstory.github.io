---
title: "AWS EKS Terraform 작성"
date: 2021-09-15T21:52:55+09:00
description: "Example article description"
categories:
  - "tech"
tags:
  - "terraform"
  - "akos"
  - "eks"
  - "aws"
menu: main # Optional, add page to a menu. Options: main, side, footer

# Theme-Defined params
# thumbnail:  "img/placeholder.png" # Thumbnail image
lead: "테라폼으로 AWS EKS 배포하기" # Lead text
comments: true # Enable Disqus comments for specific page
authorbox: true # Enable authorbox for specific page
pager: true # Enable pager navigation (prev/next) for specific page
toc: true # Enable Table of Contents for specific page
mathjax: false # Enable MathJax for specific page
sidebar: "right" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "search"
  - "recent"
  - "categories"
  - "taglist"
---

## AKOS 스터디

CloudNet@에서 진행하는 AKOS(AWS Kubernetes Online Study)를 진행하면서 자체적으로 Cloudformation 템플릿을 제공했는데, 이를 테라폼으로 사용해 보고 싶었다.

### 테라포밍

현재 사용하고 있는 메인 툴이 클라우드 포메이션이기도 하고, 과거에 클라우드 포메이션으로 약간 아쉬운 점도 있었기에, EKS 테라포밍으로 공부하면서 스터디를 할 생각으로 테라폼으로 이를 작성 해봤다.

> 테라폼과 클라우드포메이션 등 리소스를 배포하는데는 다양한 방법이 있는데 어떤 방식을 사용해도 장단점은 존재하며 그걸 어떻게 수용할지는 본인의 선택이다.  \
> 예를 들면 테라폼에서 프로그래밍 혹은 bash에서 curl로 다양한 값들을 가져와서 처리하고 싶지만 그러기엔 좀 더 HCL(Hashcorp Configuration Language)을 깊게 공부해야 하며 나름의 한계가 보이는 부분도 있다. \
> Python의 boto3를 사용해 리소스 배포와 같은 스크립트를 작성한다고 하면 클라우드포메이션이나 테라폼같은 툴들과는 다른 자유도에 행복할 수 있으나 기존의 툴들에서 제공하는 상태를 저장하는 기능에 대해서는 또 직접 구현하거나 멱등성을 고려해서 하나하나 코드를 작성해야 한다.

테라폼을 작성함에 있어 가능하면 eksctl을 사용하지 않으면서 진행하려 했다.
eksctl이 대부분 작업을 해주는 것은 cloudformation을 통해 필요한 리소스를 생성해주는 것인데, 이는 충분히 테라폼으로 커버가 가능하다고 생각했다.

그래서 테라폼 작업을 위해 
1. AWS 매뉴얼을 보며 AWS콘솔에서 한번 테스트
2. AWS CLI 명령어 확인 (AWS CLI는 AWS콘솔에서 보여주지 못한 값들을 요구하는 경우가 많다)
3. eksctl의 cloudformation 템플릿 확인
4. 테라폼 문서 확인
5. 구글 서칭 (특정 기능은 테라폼에서 만들어서 제공하기도 하고, 그 외에 사용자들이 작성한 모듈에서도 도움을 많이 받았다.)

이런 방식을 번갈아 가며 확인하며 진행했다. eksctl의 경우 실행 후 클라우드포메이션에서 템플릿과 리소스를 테라폼으로 옮겼다.

### 테라폼 템플릿

main 브랜치에는 eks_cluster까지만 구성하고, 각각 feature/nodegroup, feature/fargate 브랜치를 통해 각각의 테스트를 가능하게 해봤다.

아래는 브랜치별 구성된 내용을 아래처럼 정리해봤다.
- [main 브랜치](https://github.com/ryusstory/eks-study-terraform)
  - VPC + public subnet + eks cluster + bastion host
- [nodegroup 브랜치](https://github.com/ryusstory/eks-study-terraform/tree/eks-ec2node)
  - VPC + public subnet + eks cluster + bastion host + EC2 node group
- [fargate 브랜치](https://github.com/ryusstory/eks-study-terraform/tree/eks-fargate)
  - VPC + public subnet + eks cluster + bastion host + private subnet + nat gateway + fargate

## 사용법 설명

### git clone

클론을 받아야 하는데, nodegroup용 branch와 fargate용 브랜치를 나눠놨다.

그래서 필요한 대로 받으면 되고, main 브랜치의 경우 eks cluster까지만 구성되도록 해놨다.

```
git clone -b eks-ec2node https://github.com/ryusstory/eks-study-terraform.git
# or 
git clone -b eks-fargate https://github.com/ryusstory/eks-study-terraform.git

```

### AWS credential 설정

[`aws configure` 등으로 설정](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-files.html)해줘야 한다.

### 테라폼 설치

[테라폼은 기본적으로 설치](https://learn.hashicorp.com/tutorials/terraform/install-cli?in=terraform/aws-get-started)되어 있어야 한다.


### 테라폼 실행

`terraform init` 을 통해 aws에 필요한 툴을 설치한 뒤 `terraform apply`로 배포하면 된다.

처음에는 변수를 편집했어야 하는데, 이 경우 특히 중요한 IP 설정을 놓칠 것 같아  `terraform apply`를 입력할 경우 변수를 입력받도록 해놨다.

그래서 `terraform apply`를 입력할 경우 아래와 같이 ip를 물어보게 된다.

```bash
> terraform apply
var.home_cidr
  Enter a value:
```

하지만 매번 이 귀찮은 것을 넣을 수 없으니 아래 방법으로 실행하면 된다.

### 변수 설정

```bash
terraform apply -var home_cidr=1.2.3.4/32
```

혹은 [home_cidr variable을 찾아서](https://github.com/ryusstory/eks-study-terraform/blob/c48ba8cc47dd1acfad7a9ef327e6068c7fbac659/01.variables.tf#L34-L37) 직접 넣어서 사용할 수도 있다. 아래처럼 넣어주면 된다.
```hcl
variable "home_cidr" {
    # 접근할 집 주소 등 설정
    type = string
    default = "1.2.3.4/32"
}
```
그 외에도 [노드그룹등 설정](https://github.com/ryusstory/eks-study-terraform/blob/c48ba8cc47dd1acfad7a9ef327e6068c7fbac659/01.variables.tf#L48-L64)정도가 포함되어 있다.

위와 같이 설정하면 `terraform apply`만으로 바로 배포가 가능하다. `terraform deploy`를 사용해 지울 때도 변수입력이 필요하기 때문에 변수를 미리 설정해두는 하는 방법이 가장 좋다.

### 배포 완료 후 kubectl 설정

배포가 완료되면 바로 kubectl을 사용할 수 있으며 [`aws eks update-kubeconfig --name k8s-eks` 를 사용해서 ~/.kube/config 파일을 생성](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/create-kubeconfig.html)해 준다. 

해당 커맨드는 배포 완료 후 output에서 확인할 수 있고, `terraform output`을 입력해도 볼 수 있다.

현재 사용하고 있는 credential 기준으로 업데이트된다. 예를 들면 사용하는 PC 에서는 `~/.aws/credentials` 파일을 참고하고 EC2 IAM Profile이 있는 EC2에서는 Iam Role 을 기준으로 업데이트된다.

(EC2 IAM Profile을 생성하는 부분은 여기를 참고)[https://github.com/ryusstory/eks-study-terraform/blob/8711ae552473ec76f2e984372e424d2d0598148a/40.eks_bastion_host.tf#L32-L63]하면 된다.

### bastion host에서 kubectl 사용하기 (kubectl권한을 다른 IAM 사용자에게도 설정)

좀 더 이상적인 모델을 위해 bastion host에는 별도 access key를 사용하지 않고, EC2 IAM Profile 을 통해 권한을 부여하도록 했다. 그러다 보니 해당 IAM Role에 kubectl 권한을 줘야 하는데, kubectl 로 컨피그를 받아서 sed로 해당 내용을 추가해서 적용하는 방법도 있긴 한데, 이 경우 윈도우에서 사용할 수가 없어 수동으로 추가하기로 했고, 아래 두 자료를 참고해 직접 넣는 방법을 사용하기로 했다.

- [Amazon EKS에서 클러스터를 생성한 후 다른 IAM 사용자 및 역할이 액세스할 수 있도록 하려면 어떻게 해야 합니까?
](https://aws.amazon.com/ko/premiumsupport/knowledge-center/amazon-eks-cluster-access/)
- [클러스터의 사용자 또는 IAM 역할 관리
](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/add-user-role.html)

`kubectl edit -n kube-system configmap/aws-auth` 명령어를 치면 윈도우의 경우 메모장, 리눅스의 경우 vi 편집기가 나온다.

그러면 보통 아래와 같은 출력이 나오는데 중간의 추가해야 할 곳에 IAM Role 에 대한 컨피그를 추가해야 한다.
```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      - system:node-proxier
      rolearn: arn:aws:iam::123456789012:role/k8s-eks-fargate-profile
      username: system:node:{{SessionName}}
# ------------ 추가해서 넣어야 할 곳 ------------
kind: ConfigMap
metadata:
  ---생략---
```

[configmap.yml 파일은 테라폼에서 iam_role 부분을 가져와서 직접 생성](https://github.com/ryusstory/eks-study-terraform/blob/8711ae552473ec76f2e984372e424d2d0598148a/40.eks_bastion_host.tf#L185-L196)했다.
`data:`, `mapRoles:` 의 내용을 줄 맞춤을 위한 출력이라 붙여넣으면 안 된다.

`./output/configmap.yml` 을 열어보면 아래와 같은 내용이 나오는데 mapRoles 아래의 내용을 `kubectl edit -n kube-system configmap/aws-auth`에 나온 편집기에 mapRoles 리스트에 붙여넣어주면 된다.

```yaml
"data":
  "mapRoles": |
# ------------ 여기 아래부터 복사
    - rolearn: arn:aws:iam::123456789012:role/k8s-eks_bastion_host_role
      username: k8s-eks_bastion_host_role
      groups:
        - system:masters
```
적용하면 아래와 같이 된다. 붙여넣은 부분 외에 다른 부분은 각자 그리고 배포 방식에 따라 다를 수 있다.
```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      - system:node-proxier
      rolearn: arn:aws:iam::123456789012:role/k8s-eks-fargate-profile
      username: system:node:{{SessionName}}
    - rolearn: arn:aws:iam::123456789012:role/k8s-eks_bastion_host_role   # 붙여넣은 부분
      username: k8s-eks_bastion_host_role                                 # 붙여넣은 부분
      groups:                                                             # 붙여넣은 부분
        - system:masters                                                  # 붙여넣은 부분
kind: ConfigMap
metadata:
  creationTimestamp: "2021-09-18T13:14:33Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "44084"
  uid: 67b19ca3-b8e3-4570-bf24-a03c0ae49cc5
```

위처럼 만들고 저장하고 닫으면 바로 설정이 적용된다.

bastion host가 배포될 때 마지막 스크립트에 `aws eks update-kubeconfig --name k8s-eks`를 통해 kubeconfig를 업데이트하도록 해놨는데, 이게 먹힐 때가 있고 안 먹힐 때가 있다. 그 이유는 cluster 생성중에는 해당 커맨드를 사용할 수가 없어 EKS Cluster보다 Bastion host가 먼저 생성될 경우 `Cluster status is CREATING` 메세지와 함께 업데이트 되지 않는다.

아무튼 bastion host에서도 테라폼 생성 완료 후 aws-auth를 설정하고 `aws eks update-kubeconfig --name k8s-eks` 설정을 해주면 된다.

### (option) bastion_host 접속 시 윈도우에서 keypair 파일 권한

putty와 같은 ssh 접속 프로그램을 사용하면 아래와 같은 문제는 없는데, 윈도우에서 예전에는 지원을 안했지만 이제는 단순히 cmd창 혹은 powershell창에서 `ssh`를 사용할 수 있어서 해당 방식으로 접근할때는 아래와 같은 작업이 필요하다.

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions for './outputs/aws_ssh_keypair.pem' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "./outputs/aws_ssh_keypair.pem": bad permissions
```

리눅스나 맥의 경우 단순히 파일 생성 시 권한을 400으로 설정해서 바로 접근이 가능하지만 윈도우의 경우 테라폼에서 이러한 권한을 설정할 수가 없기 때문에 직접 권한 상속을 처리해줘야해서 좀 귀찮지만 직접 스크립트를 사용해야 한다.

[GUI를 통해 설정하는 방법](https://techsoda.net/windows10-pem-file-permission-settings/)도 있긴 하지만 스크립트를 사용하면 더 간단하다.

CMD를 사용할 경우 (.\scrits\windows-cmd.bat)를 실행하면 된다.

```
.\scripts\windows-cmd.bat
```

Powershell의 경우에는 기본적으로 파워셀에서 파워셀 스크립트 실행을 막아놔서 [파워쉘용 스크립트](./outputs/windows-powershell.ps1)를 열어 내용으로 복사해서 파워쉘에 붙여넣으면 된다.

생각해보자면 좀 더 편하게 하려면 내 PC의 Access key를 그냥 EC2에 넣어서 설정하면 되지만 공부를 위한 목적이라 해당 부분을 조금 불편하더라도 이렇게 구성했다.

### 파일설명

```bash
│  00.provider.tf           # 테라폼 기본 환경 설정 파일
│  01.variables.tf          # 변수 설정 파일
│  10.vpc+public_subnets.tf # VPC, 서브넷 배포 템플릿
│  20.eks_cluster_role.tf   # eks 클러스터용 role 배포 템플릿
│  21.eks_cluster.tf        # eks 클러스터 배포 템플릿
│  40.eks_bastion_host.tf   # eks 접속용 bastion host 배포 템플릿
│  README.md                # README
├─scripts
│  ├─windows-cmd.bat        # 윈도우용 ssh keypair 권한 설정 스크립트
│  └─windows-powersehll.ps1 # 파워쉘용 ssh keypair 권한 설정 스크립트
└─outputs                   # aws keypair 등 output이 생겼을때 파일을 담을 폴더
```

각 두 자리 숫자마다 사용하려는 용도 정리를 하면 아래와 같다.

00 대역은 기본적인 설정
10 대역은 기본적인 인프라 설정
20 대역은 EKS Cluster 관련 설정
30 대역은 Node group or fargate 관련 설정
40 대역은 bastion hosts 관련 설정

### 위 내용을 구성도로 그리면 아래와 같다.

![image](https://user-images.githubusercontent.com/20898758/133926419-dd739d6b-3199-460f-ba5b-d2c0de5b177f.png)

## AWS EKS on Nodegroup

배포되고 나면 bastion host에 배포 스크립트로 워커노드에 쉽게 접속하는 것을 넣어놨다.

[`/set_worker_node.sh` 를 실행](https://github.com/ryusstory/eks-study-terraform/blob/8711ae552473ec76f2e984372e424d2d0598148a/40.eks_bastion_host.tf#L166-L182)하면 아래처럼 `/usr/local/bin/` 폴더에 워커노드의 정보가 들어가고  `w1`, `w2` 와 같은 명령어를 통해 바로 워커노드로 바로 접속이 가능하다.

```bash
[root@eksctl-host ~]# /set_worker_node.sh 
WORKER_NODES exists
[root@eksctl-host ~]# ls -al /usr/local/bin/w*
-rwxr-xr-x 1 root root 87 Sep 19 20:06 /usr/local/bin/w1
-rwxr-xr-x 1 root root 87 Sep 19 20:06 /usr/local/bin/w2
[root@eksctl-host ~]# w1
Warning: Permanently added '192.168.1.139' (ECDSA) to the list of known hosts.
Last login: Sun Sep 19 11:05:10 2021 from ip-192-168-0-100.ap-northeast-2.compute.internal

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-192-168-1-139 ~]$ exit
logout
Connection to 192.168.1.139 closed.
[root@eksctl-host ~]# w2
Warning: Permanently added '192.168.2.222' (ECDSA) to the list of known hosts.
Last login: Sun Sep 19 11:05:11 2021 from ip-192-168-0-100.ap-northeast-2.compute.internal

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-192-168-2-222 ~]$ 
```

## AWS EKS on Fargate

처음 terraform 을 작성하고 그 뒤로 fargate 배포 형태를 적용해봤다.
- https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/fargate-getting-started.html

하지만 기본적으로 EC2 노드그룹에서 사용하던 eks cluster는 fargate로 사용하려면 문제가 있었다.

배포 직후 coredns 상태를 보면 아래와 같다
```
[root@eksctl-host ~]# kubectl get deployment -n kube-system
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   0/2     2            0           21m
```

[fargate로 사용하려면 aws 문서](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/fargate-getting-started.html)를 통해 fargate profile과 coredns profile을 설정해주어야만 제대로 동작한다.

fargate profile은 모두 terraform에 포함되어 있기에 coredns 배포에 대해 패치만 해주면 coredns가 정상적으로 올라온다.

```bash
kubectl patch deployment coredns -n kube-system --type json -p='[{"op": "remove", "path": "/spec/template/metadata/annotations/eks.amazonaws.com~1compute-type"}]'
```

fargate 배포에 대한 설정을 k8sfp 네임스페이스에 대해 셀렉터를 설정해 두었다.

그래서 먼저 `kubectl create namespace k8sfp`로 네임스페이스를 생성하고 `kubens k8sfp`나 `kubectl config set-context --current --namespace=k8sfp`로 네임스페이스를 전환해 사용하면 된다.

네임스페이스를 변경하고 배포하면 fargate에 배포되게 된다. 그리고 fargate로 배포하면 배포당 약 1분~정도 소요된다.
```
[root@eksctl-host ~]# kubectl get deployment
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
nginx-to-scaleout   3/3     3            3           15m
[root@eksctl-host ~]#
```

### 리소스 삭제
클라우드포메이션도 마찬가지겠지만 삭제 시에는 eks에서 로드밸런서와 같은 AWS 자원을 삭제 후 테라폼을 삭제해야 한다.

그래서 `kubectl get deployment --all-namespaces`등으로 확인해서 kube-system 외의 네임스페이스에서 배포된 자원들을 삭제 후 

`terraform destroy -var home_cidr=1.2.3.4/32` 명령어를 통해 자원을 삭제하면 된다.
