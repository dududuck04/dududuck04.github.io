---
layout: default
title: EKS Encryption
nav_order: 10
permalink: docs/02_AWS/EKS/eks-encryption/eks-encryption
parent: EKS
grand_parent: AWS
---

# AWS EKS 암호화 정책 적용
{: .no_toc }

## 목차
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 글을 쓴 배경

AWS EKS 환경에서 데이터 보호는 필수적입니다. 이 포스트에서는 AWS EKS에서 제공하는 암호화 옵션을 사용하여 Kubernetes 시크릿을 보호하는 방법을 안내합니다. 관련 자료는 AWS EKS Best Practices, AWS 공식 블로그, 및 AWS KMS 개발자 가이드에서 찾을 수 있습니다.


관련 문서 : https://aws.github.io/aws-eks-best-practices/ko/security/docs/data/

https://aws.amazon.com/ko/blogs/containers/using-eks-encryption-provider-support-for-defense-in-depth/

https://docs.aws.amazon.com/ko_kr/kms/latest/developerguide/data-keys.html

## 글 요약

## 시작하기 전

- 
- 
- 

---

# AWS Encryption 이란?

AWS EKS에서 Kubernetes 시크릿은 민감한 정보를 보호하는 역할을 하며 기본적으로 Base64 인코딩된 형태로 etcd에 저장됩니다. AWS에서는 이를 더 안전하게 관리할 수 있도록 KMS(Key Management Service)를 사용한 암호화 방법을 제공합니다.

EKS 에서 쿠버네티스 시크릿을 안전하게 관리하는 방법 가이드

방법 : KMS를 사용하여 secret을 암호화 하기

기존 : Kubernetes 시크릿 - 비밀번호나 API 키 같은 민감한 정보를 관리하는 방식. Base64로 인코딩된 형태로 etcd에 저장된다.

옵션 적용 : 마스터키를 KMS에 저장하여 데이터 암호화 키를 만든다. Data Encryption Key를 이용해 쿠버네티스의 시크릿을 암호화 복호화 한다.

# 시크릿 관리 방법

시크릿 생성 및 저장: 사용자가 Kubernetes 시크릿을 생성하면, 이는 Data Encryption Key(DEK)를 사용하여 암호화되고, 이후 etcd에 저장됩니다.
DEK의 관리: 각 시크릿에 대해 생성된 DEK는 Customer Master Key(CMK)를 이용해 KMS에서 관리되며 암호화됩니다.
암호화 프로세스: 사용자가 시크릿을 요청할 때, API 서버는 KMS를 통해 DEK를 복호화하고, 복호화된 DEK로 시크릿 데이터를 액세스 가능한 형태로 변환합니다.

# 실제 적용 사례

테스트 1: EBS Encryption

상황: Node Group에서 EBS 암호화 정책 미적용 상태
결과: 노드 그룹 배포 후 볼륨에 대한 암호화 적용이 자동으로 이루어지지 않았음.
테스트 2: Secrets Encryption 적용

변경: Secrets encryption 설정을 Off에서 On으로 변경
결과: 인스턴스 및 볼륨의 재시작 없이, 볼륨의 암호화 상태 변경 없음.

# 문제점 및 해결 방법

문제: Fargate 사용 중 CloudTrail에 encrypt 이벤트가 과도하게 발생함.
해결: 네트워크 트래픽과 API 호출 패턴을 분석하여 원인 규명 필요.



참고자료 : 

# AWS Encryption 적용


## 시나리오

글을 쓴 배경
AWS EKS 환경에서 데이터의 안전성은 매우 중요합니다. 이 포스트에서는 Kubernetes 시크릿을 보호하기 위해 AWS EKS에서 제공하는 암호화 옵션에 대해 자세히 설명합니다. 이와 관련된 자세한 정보는 AWS EKS Best Practices, AWS 공식 블로그, 및 AWS KMS 개발자 가이드에서 확인할 수 있습니다.

AWS EKS에서의 데이터 암호화
Kubernetes 시크릿 관리

Kubernetes 시크릿은 API 키나 비밀번호 같은 민감한 정보를 저장하는데 사용됩니다. 기본적으로 이 시크릿들은 Base64로 인코딩된 형태로 etcd에 저장됩니다. AWS는 이 데이터를 더 안전하게 보호하기 위해 KMS(Key Management Service)를 활용한 암호화 방법을 제공합니다.

암호화 방법

시크릿 생성 및 암호화: 사용자가 시크릿을 생성하면, 해당 시크릿은 Data Encryption Key(DEK)를 이용하여 암호화됩니다.
DEK 관리: 생성된 DEK는 Customer Master Key(CMK)에 의해 관리되고 KMS에서 추가 암호화됩니다.
암호화 프로세스: 시크릿을 요청할 때 API 서버는 KMS를 통해 DEK를 복호화하고, 복호화된 DEK로 시크릿 데이터를 사용자에게 제공합니다.
실제 적용 사례
EBS Encryption 테스트

상황: Node Group에서 EBS 암호화 정책 미적용 상태
결과: 노드 그룹 배포 후 자동으로 볼륨 암호화가 적용되지 않음
Secrets Encryption 적용 테스트

변경: Secrets encryption 설정을 Off에서 On으로 변경
결과: 인스턴스 및 볼륨 재시작 없이 볼륨의 암호화 상태가 유지됨
문제점 및 해결 방법
문제: Fargate 사용 중 CloudTrail에 encrypt 이벤트가 과도하게 발생 해결: 네트워크 트래픽과 API 호출 패턴 분석을 통한 원인 규명 필요

