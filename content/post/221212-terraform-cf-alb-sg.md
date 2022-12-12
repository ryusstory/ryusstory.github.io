---
title: "Cloudfront 와 ALB 보안그룹 설정 테라폼으로만 구성하기"
date: 2022-12-12T21:28:55+09:00
description: "Example article description"
categories:
  - "tech"
tags:
  - "terraform"
  - "aws"
  - "t101"
menu: main # Optional, add page to a menu. Options: main, side, footer

# Theme-Defined params
# thumbnail:  "img/placeholder.png" # Thumbnail image
lead: "Cloudfront 와 ALB 보안그룹 설정 테라폼으로만 구성하기" # Lead text
comments: false # Enable Disqus comments for specific page
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

## 소개

이 글을 쓰게 된 계기는 다른 글을 쓰다가 Cloudfront와 ALB의 보안 구성을 어떻게 하면 테라폼으로 편하게 배포할까 고민하다가 생각보다 잘 동작하는걸 보고 쓰게 되었습니다.

기존에 lambda를 통해 보안그룹을 업데이트가 가능하지만 이를 테라폼으로 가능한 것을 확인하고 quota를 적용하지 않고 순수 상태로 cloudfront용 보안그룹을 만들어 보려합니다.

## Cloudfront와 ALB의 보안구성

먼저 보통 Cloudfront를 붙여서 구성하게 되면 아래와 같이 인프라가 구성되게 됩니다.

![image](https://user-images.githubusercontent.com/20898758/207043038-fadd5e46-faa8-4ae9-87d4-3e130cb6e9e9.png)

Cloudfront를 통해서 내부 인프라에 접근하는 구성인데요. 유저는 실제 Cloudfront랑 통신하고 Cloudfront가 내부 S3나 ALB와 연결을 중계 해주는 방식입니다.

그런데 제대로 구성하지 않으면 S3의 경우나 ALB를 외부에서 모두 접근가능하도록 구성하기도 합니다.

S3의 경우 OAI (Origin Access Identity)로 쉽게(?) 구성이 가능하지만 Cloudfront-ALB의 경우 아래 두가지 방법이 있습니다.

- Cloudfront에서 사용하는 IP만 허용하는 보안그룹을 사용하는 방법 [https://aws.amazon.com/ko/blogs/security/how-to-automatically-update-your-security-groups-for-amazon-cloudfront-and-aws-waf-by-using-aws-lambda/](https://aws.amazon.com/ko/blogs/security/how-to-automatically-update-your-security-groups-for-amazon-cloudfront-and-aws-waf-by-using-aws-lambda/)
- 로드밸런서에서 특정 헤더를 허용하고 Cloudfront에서 ALB로 요청시 특정 헤더를 사용하는 방법[https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html)

읽어봐서는 ip로 제한하는 것이 더 강력한 방법인 것 같기도 하고 그렇습니다.

제가 아는 정도로는 대부분 이 IP 리스트 보안그룹을 통해서 사용하는 것으로 알고 있고 저도 회사에서 저렇게 사용하고 있습니다. 위의 Lambda를 생성해서 사용했었는데요.

이 글에서는 저 Lambda에서 하는 일을 Terraform에서 적용하는 걸 해보려 합니다.

## 사전에 알아야 할 지식

### AWS Limits

아마존을 사용해 보신 분들은 많이들 겪어서 아시겠지만 생각보다 AWS에는 Limitation이 많습니다.

일단 여기서 알아야 될 할당량은…

- EC2의 최대 보안그룹 개수
    - EC2에 보안 그룹은 EC2에 바로 붙는게 아니라 EC2에 ENI(네트워크 인터페이스)가 추가되고 ENI에 보안그룹이 붙는 형태입니다. 여기에 일단 한계점이 하나 있습니다.
    - 기본 값이 최대 5개이고 quota를 통해서 증가시킬 수 있습니다.
- 보안그룹의 최대 정책(rule) 개수
    - 보안 그룹 안에 넣을 수 있는 inbound rule(들어오는 정책)은 기본 최대 60개이고 이 역시 quota를 통해서 증가시킬 수 있습니다.
- 관련 정보
    - [https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/amazon-vpc-limits.html#vpc-limits-security-groups](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/amazon-vpc-limits.html#vpc-limits-security-groups)

위 quota에서 할당량 증가를 요청하면 여러개의 보안그룹에 나누지 않아도 넣을 수 있겠지만 제가 하려고 한 부분은 quota를 손대지 않고 terraform으로 이 리스트를 ALB에 넣기는 쉽지 않습니다.

예전에는 aws에서 ip list를 제공하고 여기서 니네가 알아서 뽑아써 느낌이었는데, 이제는 별도로 리스트를 제공하네요 ㅎㅎ

- IP list 관련된 아마존 공식문서
    - [https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/LocationsOfEdgeServers.html](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/LocationsOfEdgeServers.html)
- 전체 IP list
    - [https://ip-ranges.amazonaws.com/ip-ranges.json](https://ip-ranges.amazonaws.com/ip-ranges.json)
- Cloudfront IP list
    - [https://d7uri8nf7uskq.cloudfront.net/tools/list-cloudfront-ips](https://d7uri8nf7uskq.cloudfront.net/tools/list-cloudfront-ips)

그냥 세기는 어려우니 cloudfront ip 주소를 개수를 한번 확인해보겠습니다.

```jsx
> cloudFrontIpList
{
  CLOUDFRONT_GLOBAL_IP_LIST: [
    '120.52.22.96/27',    '205.251.249.0/24',   '180.163.57.128/26',
    '204.246.168.0/22',   '18.160.0.0/15',      '205.251.252.0/23',
    '54.192.0.0/16',      '204.246.173.0/24',   '54.230.200.0/21',
    '120.253.240.192/26', '116.129.226.128/26', '130.176.0.0/17',
    '108.156.0.0/14',     '99.86.0.0/16',       '205.251.200.0/21',
    '223.71.71.128/25',   '13.32.0.0/15',       '120.253.245.128/26',
    '13.224.0.0/14',      '70.132.0.0/18',      '15.158.0.0/16',
    '13.249.0.0/16',      '18.238.0.0/15',      '18.244.0.0/15',
    '205.251.208.0/20',   '65.9.128.0/18',      '130.176.128.0/18',
    '58.254.138.0/25',    '54.230.208.0/20',    '3.160.0.0/14',
    '116.129.226.0/25',   '52.222.128.0/17',    '18.164.0.0/15',
    '64.252.128.0/18',    '205.251.254.0/24',   '54.230.224.0/19',
    '71.152.0.0/17',      '216.137.32.0/19',    '204.246.172.0/24',
    '18.172.0.0/15',      '120.52.39.128/27',   '118.193.97.64/26',
    '223.71.71.96/27',    '18.154.0.0/15',      '54.240.128.0/18',
    '205.251.250.0/23',   '180.163.57.0/25',    '52.46.0.0/18',
    '223.71.11.0/27',     '52.82.128.0/19',     '54.230.0.0/17',
    '54.230.128.0/18',    '54.239.128.0/18',    '130.176.224.0/20',
    '36.103.232.128/26',  '52.84.0.0/15',       '143.204.0.0/16',
    '144.220.0.0/16',     '120.52.153.192/26',  '119.147.182.0/25',
    '120.232.236.0/25',   '54.182.0.0/16',      '58.254.138.128/26',
    '120.253.245.192/27', '54.239.192.0/19',    '18.68.0.0/16',
    '18.64.0.0/14',       '120.52.12.64/26',    '99.84.0.0/16',
    '130.176.192.0/19',   '52.124.128.0/17',    '204.246.164.0/22',
    '13.35.0.0/16',       '204.246.174.0/23',   '36.103.232.0/25',
    '119.147.182.128/26', '118.193.97.128/25',  '120.232.236.128/26',
    '204.246.176.0/20',   '65.8.0.0/16',        '65.9.0.0/17',
    '108.138.0.0/15',     '120.253.241.160/27', '64.252.64.0/18'
  ],
  CLOUDFRONT_REGIONAL_EDGE_IP_LIST: [
    '13.113.196.64/26',  '13.113.203.0/24',  '52.199.127.192/26',
    '13.124.199.0/24',   '3.35.130.128/25',  '52.78.247.128/26',
    '13.233.177.192/26', '15.207.13.128/25', '15.207.213.128/25',
    '52.66.194.128/26',  '13.228.69.0/24',   '52.220.191.0/26',
    '13.210.67.128/26',  '13.54.63.128/26',  '99.79.169.0/24',
    '18.192.142.0/23',   '35.158.136.0/24',  '52.57.254.0/24',
    '13.48.32.0/24',     '18.200.212.0/23',  '52.212.248.0/26',
    '3.10.17.128/25',    '3.11.53.0/24',     '52.56.127.0/25',
    '15.188.184.0/24',   '52.47.139.0/24',   '18.229.220.192/26',
    '54.233.255.128/26', '3.231.2.0/25',     '3.234.232.224/27',
    '3.236.169.192/26',  '3.236.48.0/23',    '34.195.252.0/24',
    '34.226.14.0/24',    '13.59.250.0/26',   '18.216.170.128/25',
    '3.128.93.0/24',     '3.134.215.0/24',   '52.15.127.128/26',
    '3.101.158.0/23',    '52.52.191.128/26', '34.216.51.0/25',
    '34.223.12.224/27',  '34.223.80.192/26', '35.162.63.192/26',
    '35.167.191.128/26', '44.227.178.0/24',  '44.234.108.128/25',
    '44.234.90.252/30'
  ]
}
> cloudFrontIpList.CLOUDFRONT_GLOBAL_IP_LIST.length
84
> cloudFrontIpList.CLOUDFRONT_REGIONAL_EDGE_IP_LIST.length
49
> cloudFrontIpList.CLOUDFRONT_GLOBAL_IP_LIST.length + cloudFrontIpList.CLOUDFRONT_REGIONAL_EDGE_IP_LIST.length
133
>
```

총 133개 네요!

아까 말씀드린 quota를 증가시키지 않고 적용할 경우 60개를 꽉꽉 채운다고 했을 때, 133/60 하면 최소 3개의 보안그룹이 필요하겠네요. 

### 최소 지식 조건 loop

다른 언어를 통해서도 좋고 테라폼을 통해서도 좋고 적어도 loop 에 대해서는 조금 이해가 있어야 쉽게 보실 수 있습니다.

[https://developer.hashicorp.com/terraform/language/expressions/for](https://developer.hashicorp.com/terraform/language/expressions/for)

그리고 이 글에서는 count 를 통해서 리소스를 생성하는 방식으로 진행했습니다. 

리소스에 `count`를 통해서 생성해본 적 있거나  `count.index` 가 어떻게 적용되는지 정도는 알아야 이해가 어렵지 않습니다.

[https://developer.hashicorp.com/terraform/language/meta-arguments/count](https://developer.hashicorp.com/terraform/language/meta-arguments/count)

## 선구자가 이미 있다

이걸 어떻게 적용할까 고민하다가 찾아보니 이미 외국에 적용한 사람이 있더라구요. 그것도 무려 2019년에… 

[https://jonnyzzz.com/blog/2019/03/26/terraform-cloudfront-sg/](https://jonnyzzz.com/blog/2019/03/26/terraform-cloudfront-sg/)

여기서 찾아본적도 없지만… 엄청 난걸 알았습니다. 테라폼에서 data source로 aws clodufront ip list를 제공한다는걸요!

[https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ip_ranges](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ip_ranges)

그리고 위 블로그는 133개 IP를 30개씩 나눠서 5개 보안 그룹을 사용하는데요. 그렇게 되면 Cloudfront 용도 보안그룹으로만 가득 채워지게 되니까 사용하기 좀 안좋을 것 같아서 좀 개선을 하고 싶었고, 어떤 과정을 통해서 적용했는지 정리해봤습니다.

## data를 까보자 with terraform console

약간 제가 삽질할때 쓰는 도구인데요. console이 생각보다 별로지만(?) 생각보다 유용합니다

일단 [main.tf](http://main.tf) 파일로 아래 내용을 넣어서 하나 만들어줍니다.

```jsx
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region  = "ap-northeast-2"
}

data "aws_ip_ranges" "cloudfront" {
    services = ["cloudfront"]
}

locals {
  foo = "bar"
}

variable "foo" {
  default = ["b","a","r"]
}
```

보통 다른 사람 코드를 가져와서 쓰다보면 이런 데이터가 어떻게 들어있는지 어떻게 가지고 놀아야 하는지 어려울 때가 많은데요. 저도 계속 그래왔었고 대부분 terraform 설정 파일 안의 값을 output으로 뽑거나 해서 확인했었는데요. 여기서 console을 확인하면 이것저것 미리 테스트가 가능합니다.

그럼 위에서 만든 `data`, `local`, `variable` 이 어떻게 console에서 보이는지 한번 보겠습니다.

먼저 위 내용만 가지고 `terraform init` 를 해줍니다.

 `terraform console` 을 입력하면 아래처럼 프롬프트가 뜨게 되는데 현재 폴더의 tf파일을 읽어온 채로 console을 실행하게 됩니다!

```jsx
❯ tf init   

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 4.0"...
- Installing hashicorp/aws v4.46.0...
- Installed hashicorp/aws v4.46.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
❯ terraform console
>
```

먼저 우리가 위 data를 사용할 때 `data.aws_ip_ranges.cloudfront` 를 통해서 사용하게 되는데요. 그대로 콘솔에서 한번 보겠습니다.

```jsx
> data.aws_ip_ranges.cloudfront
(known after apply)
> local.foo
"bar"
> var.foo
[
  "b",
  "a",
  "r",
]
>
```

data는 적어도 한번 `apply`를 해서 aws에서 내용을 가져와야 그 내용을 볼 수 있지만 locals나 variables는 바로 여기서 확인이 가능합니다

그러면 `exit` 로 나가서 terraform apply를 한번 해보겠습니다.

적용할게 없으니까 그동안 봐왔던 yes도 물어보지 않고 알아서 적용됩니다.

```jsx
❯ terraform apply  
data.aws_ip_ranges.cloudfront: Reading...
data.aws_ip_ranges.cloudfront: Read complete after 1s [id=1670634785]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
❯
```

다시 콘솔로 가서 `data.aws_ip_ranges.cloudfront` 를 보겠습니다. 

```jsx
> data.aws_ip_ranges.cloudfront 
{
  "cidr_blocks" = tolist([
    "108.138.0.0/15",
    생략...
    "99.86.0.0/16",
  ])
  "create_date" = "2022-12-10-01-13-05"
  "id" = "1670634785"
  "ipv6_cidr_blocks" = tolist([
    "2400:7fc0:500::/40",
    생략...
    "2600:9000:fff::/48",
  ])
  "regions" = toset(null) /* of string */
  "services" = toset([
    "cloudfront",
  ])
  "sync_token" = 1670634785
  "url" = "https://ip-ranges.amazonaws.com/ip-ranges.json"
}
>
```

엄청나죠…?자세히 보면 앞의 문서([https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ip_ranges](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ip_ranges))에서 attribute reference 값들이 여기서 다 보이는 걸 알 수 있습니다. 

문서에서 쓸 수 있다는 `cidr_blocks`, `ipv6_cidr_blocks`, `create_date`, `sync_token`까지 data_source안에서 다 보이는 걸 확인할 수 있습니다. 

그런데.. 가만 보니… 거기에 없는 값도 쓸 수 있네요?! (쓰면 책임 못지겠지만요)

```jsx
> data.aws_ip_ranges.cloudfront.url
"https://ip-ranges.amazonaws.com/ip-ranges.json"
>
```

이제 여기서 앞의 블로그에서 얻은 내용과 앞의 할당량을 조합해서 어떻게 배포하면 될지 한번 보겠습니다.

저희는 `cidr_blocks` attribute만 사용할 것이기 때문에 이것만 가져와서 앞에서 본 개수와 비교해 보겠습니다.

테라폼에서는 length 라는 function이 있고, 그 외에도 많은 function이 있습니다. 꼭 한번 봐두면 좋습니다.

[https://developer.hashicorp.com/terraform/language/functions/length](https://developer.hashicorp.com/terraform/language/functions/length)

```jsx
> length(data.aws_ip_ranges.cloudfront.cidr_blocks)
133
>
```

앞의 133개와 동일하네요! 그렇다면 이걸 가지고 쓰면 될 것 같습니다!

그리고 description에 create_date를 한번 넣어볼게요.

## terraform에서 어떻게 적용할지 고민을 해보자

자 이제 133개를 아까 할당량으로 보면 최대 60개인데 꽉 채울수도 있을것 같지만 왠지 꽉 채우면 아무래도 좀 찝찝하니까 55개로 해보겠습니다. 왜냐면… 50개는 17개만 대역이 추가되면 보안그룹 개수가 하나 늘어야 하니까… 

현재기준 133개를 좀 안정적으로 나누려면 마지막 보안그룹에 최대 개수의 반정도만 찼으면 좋겠거든요. IP list가 변경되어도 보안그룹이 추가되거나 삭제되지 않도록요.

그러면 55개로 하면 55 + 55 + 23 = 133 이렇게 들어가면 마지막 보안그룹에 들어갈 ingress rule이  23/55(41%)로 뭔가 좀 안정감을 얻을 수 있을것 같습니다. 

## 블로그의 좋은 것들을 훔쳐오자

그렇다면 이제 앞의 블로그에서 본 것중에서 chunklist만 마저 훔쳐오면 될 것 같습니다.

[https://developer.hashicorp.com/terraform/language/functions/chunklist](https://developer.hashicorp.com/terraform/language/functions/chunklist)

문서는 이렇구요. 정리하면 제가 정한 숫자를 최대값으로 list를 나눠줍니다.

1~133까지의 숫자가 든 list가 있다고 하면

`[1,2,3,…,131,132,133]`

 chunklist 로 50 개씩 나누면 아래처럼 나눠집니다.

`[[1,2,...,49,50],[51,52,...,99,100],[101,102,...,132,133]]`

그럼 이제 콘솔을 통해서 또 테스트해보겠습니다.

```jsx
> chunklist(data.aws_ip_ranges.cloudfront.cidr_blocks,55)
tolist([
  tolist([
    "108.138.0.0/15",
    생략...
    "180.163.57.128/26",
  ]),
  tolist([
    "204.246.164.0/22",
    생략...
    "52.84.0.0/15",
  ]),
  tolist([
    "54.182.0.0/16",
    생략...
    "99.86.0.0/16",
  ]),
])
```

원하는 대로 나왔네요!

물론.. 현재 가진 리스트보다 큰 숫자를 입력하면 그냥 그대로 뽑아서 리스트에 넣어줍니다.

`[1,2,3,4,5]` 의 리스트가 있고, `chunklist([1,2,3,4,5],10)` 을 하면 `[[1,2,3,4,5]]` 로 나옵니다.

그러면 원래대로 돌아와서 각 리스트의 개수를 확인도 해보겠습니다.

```jsx
> length(chunklist(data.aws_ip_ranges.cloudfront.cidr_blocks,55))
3
> length(chunklist(data.aws_ip_ranges.cloudfront.cidr_blocks,55)[0])
55
> length(chunklist(data.aws_ip_ranges.cloudfront.cidr_blocks,55)[1])
55
> length(chunklist(data.aws_ip_ranges.cloudfront.cidr_blocks,55)[2])
23
```

총 3개의 리스트로 구성되었구요 앞서 생각한대로 55 55 23으로 나눠졌습니다.

그래서 리스트 3개가 들어있는 리스트가 만들어 졌다고 생각하시면 됩니다!

이제 이걸 테라폼으로 넣어봅시다!

## 이제 테라폼을 만들어봅시다

먼저 위에서 해봤던 chunklist로 55개로 나눠주고 local 변수에 넣어줍니다.

```jsx
locals {
  cloudfront_ipv4_cidr_chunklist = "${chunklist(data.aws_ip_ranges.cloudfront.cidr_blocks, 55)}"
	// chunklist를 통해서 list를 나눠줍니다. 총3개 리스트가 들어있을 겁니다.
}
```

이제 count를 통해서 보안그룹을 만들어 볼게요

```jsx
resource "aws_security_group" "cloudfront" {
  count = length(local.cloudfront_ipv4_cidr_chunklist)
  // 앞의 콘솔에서 확인한대로 리스트 3개가 들어있으니까 length()를 사용하면 3이 나옵니다.
  name = "cloudfront security group for ALB-${count.index}" // 보기 쉽게 보안그룹에 이름을 달아줍시다
  ingress {
    from_port = 80
    to_port   = 80
    protocol  = "tcp"
    cidr_blocks = local.cloudfront_ipv4_cidr_chunklist[count.index] // cidr_blocks에는 list를 넣어줘야하는데, 저희는 이미 리스트를 가지고 있으므로 그대로 넣어줍니다!
    description = "${data.aws_ip_ranges.cloudfront.create_date}-${count.index}" 
    // description도 앞서 얘기한대로 create_date와 count.index를 조합해서 넣어줍니다.
  }
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  lifecycle {
    create_before_destroy = true
  }
}
```

이제 apply를 해보겠습니다.

```jsx
생략...
Enter a value: yes

aws_security_group.cloudfront[2]: Creating...
aws_security_group.cloudfront[1]: Creating...
aws_security_group.cloudfront[0]: Creating...
aws_security_group.cloudfront[2]: Creation complete after 2s [id=sg-0a9e63770cc558c32]
aws_security_group.cloudfront[1]: Creation complete after 2s [id=sg-00c06a56dce4bf000]
aws_security_group.cloudfront[0]: Creation complete after 2s [id=sg-0a2bf94b78840ae04]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
❯
```

생각한대로 3개 그룹이 잘 생성됐네요 AWS에 들어가서 한번 보겠습니다.

![image](https://user-images.githubusercontent.com/20898758/207043119-351592d0-54bf-4ce9-975f-a94191bd32ac.png)

항목도 55 55 23으로 잘 나왔고 description도 잘 나왔네요! 그런데 index가 0부터 되니까 좀 보기 싫네요!

description 부분을 0,1,2 가 아니라 1,2,3으로 나오도록 아래처럼 해보겠습니다. 보안그룹이름도 CF를 위한 ALB 보안그룹이 더 맞겠네요

```jsx
resource "aws_security_group" "cloudfront" {
  ...
  name = "ALB security group for Cloudfront-${count.index+1}"
  ingress {
    ...
    description = "${data.aws_ip_ranges.cloudfront.create_date}-${count.index+1}" 
  }
}
```


![image](https://user-images.githubusercontent.com/20898758/207043135-92a20bc5-0d48-40e0-90df-f1c00dee8663.png)
만족
