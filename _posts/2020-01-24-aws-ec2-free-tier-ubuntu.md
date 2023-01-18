---
layout: post
title: AWS EC2 프리티어 ubuntu 인스턴스 생성
toc: true
---

# Introduction
## Motivation
- 회사에서 서비스를 배포하기 위해 AWS를 사용하고 있다
- 현재 필자가 맡은 프로젝트는 아직 배포 전이지만, 머지않아 AWS를 사용하여 배포될 예정이다
- AWS 사용법을 미리 익혀보고 개인 프로젝트를 배포해보기 위해 프리티어 계정을 만들어 사용해보려 한다
- ECS, ECR을 사용해보고 싶었지만 프리티어에 해당하지 않았기 때문에 EC2, RDS 정도를 먼저 사용해보려 한다
- AWS를 사용해본 적은 있지만 다시 사용할 때마다 헤매는 경향이 있어 이 포스트에 간단히 기록해둔다

## Overview
- EC2 ubuntu 인스턴스를 생성하고, ssh로 연결해볼 것이다
- 개인 프로젝트를 위한 ELK를 설치할 것이다
- 추후에 PostgreSQL RDS와 Spring Boot 서버를 배포할 것이다

# EC2 Instance 생성
- 별도 설정이 필요한 부분만 메모했다
- Ubuntu 18.04 LTS를 선택했다
- Region : Seoul
  - 짧은 Latency를 위하여
- VPC
  - Virtual Private Network, 인스턴스들 간의 논리적 네트워크 분리가 필요할 때 사용한다.
  - 그러므로 기본 VPC를 사용하면 된다
- AZ, Subnet
  - AZ : Available Zone, VPC와 반대로 리소스(인스턴스)가 생성되는 물리적인 공간
  - Subnet : VPC의 하위 개념으로 특정 AZ와 연결되어 VPC 이하의 네트워크 영역을 지정한다.
    - 이것 역시 기본 Subnet을 사용하면 된다
- Tag
  - 결제 요금 청구서를 볼 때 Tag별로 따로 볼 수 있다고 들은 것 같다. 사용해본 적은 없어서 모른다.
  - 현재는 지정할 필요 없다.
- 보안 그룹
  - AWS Level의 방화벽으로, EC2 전용 보안 그룹을 하나 만들자
  - 필요한 인바운드 포트들을 열어주면 된다
- 나머지 설정은 변경하지 않았다

# EC2 Instance 연결
- 인스턴스가 생성되면 인스턴스의 이름을 정해주자
- 생성 완료 후 public DNS 또는 IP로 SSH를 연결할 수 있다
- 좌측 상단의 연결 버튼을 누르면 연결 방법을 볼 수 있다
- 필자는 독립 실행형 SSH 클라이언트를 사용할 것이다

## 독립 실행형 SSH 클라이언트
- Mac에 설치된 ssh를 사용한다
- 인스턴스 생성 시 다운로드 받은 인스턴스의 private key의 permission을 수정하고, 연결하면 된다
```shell script
chmod 400 $KEY_PATH
ssh -i $KEY_PATH ${USERNAME}@${PUBLIC_DNS}
```
- ubuntu instance의 경우 USERNAME은 ubuntu이다

# Conclusion
- 어려운 설정 없이도 EC2 인스턴스를 생성하고 SSH 연결할 수 있을 것이다
- 본 포스팅은 여기서 마무리하고 ELK 설치 포스팅으로 돌아오도록 하겠다
- AWS의 VPC, Subnet, 보안 그룹 개념을 잘 기억해두도록 하자

