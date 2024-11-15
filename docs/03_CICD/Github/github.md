---
layout: default
title: Github
nav_order: 60
has_children: true
permalink: docs/03_CICD/Github
parent: CICD
---

# Github
{: .no_toc }

1. TOC
{:toc}

특정 태그가 Github 저장소에 푸시될 때 해당 태그 이름을 추출하여 사용할 수 있게 해주는 목적

특정 태그가 저장소에 푸시될 때만 해당 github action을 실행하도록 설계한다.

on:push: tags: - '*'



우선 해야할 일 yaml 파일을 사용하여 필요한 조건과 단계를 정의해야한다.

태그가 푸시될 때 마다 태그 이름을 추출하고 콘솔에 출력

run

core.setFailed 메서드

메세지 자체를 객체로 받아 

타입 정의와 선언부 // 라이브러리 

extractTag 함수에서는

GITHUB_REF 는 특정 경로에서 가져오게 됨

Github Actions에서 Github_ref

jobs와 runs 차이

태그가 푸시되었을 때 GITHUB_REF 환경 변수에서 태그를 추출하고, 이를 출력값으로 설정







2. Gradle ㅂㅣㄹ드 도구를 사용하는 프로젝트에 대해 Github Actions 환경을 쉽게 관리하고 통합할 수 있도록 하는 플러그인

kotlin-dsl 

group / 프로젝트나 라이브러리의 유니크한 네임스페이스를 정의한다.

group의 역할 // 라이브러리 충돌 방지 

플러그인은 gradle 빌드 프로세스에 특정 기능을 추가합니다.

java 플러그인은 자바 컴파일 태스크, 

자바 컴파일 태스크 - 자바 소스 파일을 자바 바이트 코드로 변환한다. java -> class javac 가 담당

단순 바이트 코드 변환 뿐만 아니라 코드의 문법을 검사하고 타입 체크를 수행하며 필요하다면 최적화 작업을 수행한다.

Java Archiving - 여러 개의 자바 클래스 파일과 그 외 리소스를 하나의 파일로 묶는 과정 

JAR 파일은 자바 플랫폼의 애플리케이션 배포 표준 형식 중 하나 클래스 로더에 의해 실행된다.

gradleplugin / pluginBundle


gradle plugin - gradle plugin portal에 플러그인을 등록하고, 사용자가 사용할 수 있도록 메타데이터 정의

gradle bundle - 

gradle version 

toolchain // 어떤 jdk 버전을 사용하지를 명시하는 과정이라서 필수 

source jar // jar 생성하도록 지시하는 것

dependencies 프로젝트에서 필요한 외부 라이브러리나 모듈의 의존성

tast - 특정 gradle 태스크의 설정의 정의한다.

gradle 프로젝트에서 settings.gradle.kts와 build.gradle.kts는  코틀린 스크립트 파일을 의미한다.

멀티 프로젝트 빌드에서 각 하위 프로젝트를 어떻게 구성할지를 정의한다. settings.gradle.kts //

settings를 통해 모든 하위 프로젝트를 포함하는 방법을 정의할 수 있으며 각 하위프로젝트별로 다른 구체적인 요구사항은 build에서 정의가 가능하다.

