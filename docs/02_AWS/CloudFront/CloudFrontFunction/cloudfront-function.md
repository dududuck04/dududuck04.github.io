---
layout: default
title: CloudFront Function vs Lambda@Edge 비교
nav_order: 10
permalink: docs/02_AWS/CloudFront/CloudFrontFunction/cloudfront-function
parent: CloudFront
grand_parent: AWS
---

# CloudFront Function vs Lambda@Edge 비교
{: .no_toc }

## 목차
{: .no_toc .text-delta }

1. TOC
{:toc}

---

# CloudFront Function이란?

Amazon CloudFront는 글로벌 콘텐츠 배포 네트워크(CDN) 서비스로, 데이터, 비디오, 애플리케이션 및 API를 낮은 지연 시간과 높은 전송 속도로 유저에게 전달합니다. 
CloudFront Functions는 이러한 CloudFront 요청을 처리하기 위한 서버리스 함수 기능을 제공합니다. 
이를 통해 간단한 JavaScript 코드를 사용하여 HTTP 요청과 응답을 수정할 수 있습니다.

# Lambda@Edge란?

Lambda@Edge는 AWS Lambda 함수를 유저와 가까운 위치에서 응답하여 실행할 수 있는 서비스입니다. 
이를 통해 함수가 아래와 같은 CloudFront 이벤트 종류에 따라 트리거됩니다

Viewer Request: CloudFront가 뷰어의 요청을 받은 후
Viewer Response: CloudFront가 응답을 뷰어에게 전달하기 전
Origin Request: CloudFront가 오리진으로 요청을 전달하기 전
Origin Response: CloudFront가 오리진으로부터 응답을 받은 후

Lambda@Edge를 사용하면 함수를 오리진 서버가 아닌 최종 사용자에게 가까운 위치에서 실행할 수 있습니다. 이는 지연 시간을 크게 단축시킬 수 있습니다.

# CloudFront Functions와 Lambda@Edge 비교

[Choosing between CloudFront Functions and Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-choosing.html)

| 특징 | CloudFront Functions | Lambda@Edge |
| --- | --- | --- |
| **프로그래밍 언어** | JavaScript (ECMAScript 5.1 호환) | Node.js 및 Python |
| **이벤트 소스** | Viewer request, Viewer response | Viewer request, Viewer response, Origin request, Origin response |
| **Amazon CloudFront KeyValueStore 지원** | 예 (JavaScript 런타임 2.0에서만) | 아니요 |
| **확장성** | 초당 10,000,000 요청 이상 처리 가능 | 리전당 초당 최대 10,000 요청 |
| **함수 지속 시간** | 서브밀리초 | 최대 5초 (Viewer request 및 Viewer response), 최대 30초 (Origin request 및 Origin response) |
| **최대 메모리** | 2MB | 128MB – 10,240MB (10GB) |
| **함수 코드 및 라이브러리 최대 크기** | 10KB | 1MB (Viewer request 및 Viewer response), 50MB (Origin request 및 Origin response) |
| **네트워크 액세스** | 아니요 | 예 |
| **파일 시스템 액세스** | 아니요 | 예 |
| **요청 본문 액세스** | 아니요 | 예 |
| **지리적 위치 및 장치 데이터 액세스** | 예 | 아니요 (Viewer request 및 Viewer response), 예 (Origin request 및 Origin response) |
| **CloudFront 내에서 완전한 빌드 및 테스트 가능 여부** | 예 | 아니요 |
| **함수 로깅 및 지표** | 예 | 예 |
| **가격** | 무료 티어 제공; 요청당 요금 부과 | 무료 티어 없음; 요청 및 함수 지속 시간에 따라 요금 부과 |

# 사용 사례

## CloudFront Functions의 적합한 사용 사례

* **헤더 조작**: CloudFront 응답에 사용자가 필요한 헤더를 추가하여 브라우저 캐싱 제어 및 특정 헤더 추가등 기능을 사용할 수 있습니다.
* URL 리디렉션 또는 다시 쓰기: 요청 정보를 기반으로 사용자를 다른 페이지로 리디렉션하거나 경로를 변경합니다.
* 요청 권한 부여: JWT와 같은 해시된 권한 부여 토큰을 검증하여 요청을 인증합니다.

샘플 코드 주소 : [aws-samples/amazon-cloudfront-functions](https://github.com/aws-samples/amazon-cloudfront-functions)

## CloudFront Functions의 적합한 사용 사례
- **캐시 키 정규화**: HTTP 요청 속성(헤더, 쿼리 문자열, 쿠키, URL 경로)을 변환하여 최적의 캐시 키를 생성하고 캐시 적중률을 개선합니다.
- **헤더 조작**: 요청 또는 응답에서 HTTP 헤더를 삽입, 수정 또는 삭제합니다.
- **URL 리디렉션 또는 다시 쓰기**: 요청 정보를 기반으로 사용자를 다른 페이지로 리디렉션하거나 경로를 변경합니다.
- **요청 권한 부여**: JWT와 같은 해시된 권한 부여 토큰을 검증하여 요청을 인증합니다.

[CloudFront Functions 시작하기](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html)

## Lambda@Edge의 적합한 사용 사례
- **수 밀리초 이상의 실행 시간이 필요한 함수**
- **조정 가능한 CPU 또는 메모리가 필요한 함수**
- **타사 라이브러리(AWS SDK 포함)에 의존하는 함수**
- **외부 서비스 처리를 위해 네트워크 액세스가 필요한 함수**
- **파일 시스템 액세스 또는 HTTP 요청 본문에 액세스가 필요한 함수**

[Lambda@Edge 시작하기](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html)