---
layout: default
title: CloudFront
nav_order: 20
has_children: true
permalink: docs/02_AWS/CloudFront
parent: AWS
---

# CloudFront
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

# CloudFront란?

Amazon CloudFront는 .html, .css, .js 및 이미지 파일과 같은 정적 및 동적 웹 콘텐츠를 사용자에게 더 빠르게 배포하도록 지원하는 웹 서비스입니다.

이 서비스는 엣지 로케이션이라는 데이터 센터의 전 세계 네트워크를 통해 콘텐츠를 제공합니다.

# CloudFront의 주요 기능

1. 글로벌 콘텐츠 배포
   * 전 세계에 분포된 엣지 로케이션을 통해 사용자에게 콘텐츠를 제공합니다.
2. 오리진 서버와의 통합
   * 콘텐츠가 엣지 로케이션에 없을 경우, CloudFront는 Amazon S3 버킷, 또는 오리진 서버에서 콘텐츠를 가져옵니다.
3. 향상된 성능 및 신뢰성
   * 네트워크 간의 경유 횟수를 줄여 성능을 향상시키며, 파일의 사본을 전 세계 여러 엣지 로케이션에 캐시함으로써 안정성과 가용성을 높입니다.

# CloudFront의 작동 방식

1. 배포 생성
   * 배포를 통해 콘텐츠를 어디에서 전송할지, 어떻게 전송할지를 지정할 수 있습니다.
2. 오리진 서버 설정
   * 콘텐츠의 최종 원본 버전을 저장합니다.
3. 캐싱 및 만료 설정
   * 기본적으로 엣지 로케이션에 캐시된 파일은 만료되기 전까지 24시간 동안 유지되며, 사용자 설정에 따라 이 시간을 조정할 수 있습니다.
4. 요금 구조
   * 엣지 로케이션에서 전송되는 데이터와 HTTP , HTTPS 요청에 대해 요금을 부과합니다.

# Lambda@Edge 함수

CloudFront 콘텐츠를 보호하기 위한 Authorization@Edge 솔루션
사용자가 CloudFront 배포에 접근하기 전에 인증을 거치도록 할 수 있습니다.