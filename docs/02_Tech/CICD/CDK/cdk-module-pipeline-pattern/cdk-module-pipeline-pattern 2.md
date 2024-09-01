---
layout: default
title: 2-1. TypeScript CDK로 재사용가능한 모듈 구조 작성하기 - pipeline (5편)
nav_order: 21
permalink: docs/02_Tech/CICD/CDK/cdk-module-pipeline-pattern
parent: CICD
grand_parent: Tech
---

# 재활용 할 수 있는 CDK 모듈 생성하기 - pipeline 5편

{: .no_toc }

## 목차
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 글을 쓴 배경


## 글 요약


## 시작하기 전

이 글을 읽기 전 [재활용 할 수 있는 CDK 모듈 생성하기](../cdk-module-app)을 우선 읽어주시길 바랍니다.

---

## 1. 프로젝트 구조

```perl
├── README.md
├── bin
│   └── app.ts
├── cdk.context.json
├── cdk.json
├── config
│   └── sample-pattern-config.json
├── infra
│   ├── pattern
│   │   ├── cloudwatch
│   │   │   └── cloudwatch-simple-pattern.ts
│   │   ├── codebuild
│   │   │   ├── codebuild-pol.json
│   │   │   └── codebuild-simple-pattern.ts
│   │   ├── codedeploy
│   │   │   ├── codedeploy-pol.json
│   │   │   └── codedeploy-simple-pattern.ts
│   │   └── s3
│   │       └── s3-check-pattern.ts
│   └── stack
│       ├── cfn-template
│       │   └── sample-cfn-vpc.yaml
│       ├── codebuild
│       │   ├── codebuild-simple-pattern-stack.ts
│       │   ├── codebuild-stack.ts
│       ├── codecommit
│       │   ├── codecommit-stack.d.ts
│       │   └── codecommit-stack.ts
│       ├── codedeploy
│       │   ├── codedeploy-simple-pattern-stack.ts
│       │   └── codedeploy-stack.ts
│       ├── iam
│       │   ├── codebuild-pol.json
│       │   ├── codedeploy-pol.json
│       │   └── iam-stack.ts
│       └── vpc
│           └── sample-cfn-vpc-stack.ts
├── jest.config.js
├── lib
│   ├── app-config.ts
│   ├── app-context.ts
│   └── template
│       ├── common
│       │   ├── common-guardian.ts
│       │   └── common-helper.ts
│       ├── construct
│       │   └── base
│       │       └── base-construct.ts
│       └── stack
│           ├── base
│           │   ├── base-stack.d.ts
│           │   ├── base-stack.js
│           │   ├── base-stack.ts
│           │   ├── cfn-include-stack.d.ts
│           │   ├── cfn-include-stack.js
│           │   ├── cfn-include-stack.ts
│           │   ├── vpc-base-stack.d.ts
│           │   ├── vpc-base-stack.js
│           │   └── vpc-base-stack.ts
├── package-lock.json
├── package.json
├── script
│   ├── db
│   │   └── database_helloworld.sql
│   ├── deploy_stacks.sh
│   ├── destroy_stacks.sh
│   └── setup_initial.sh
├── setup_initial.sh
└── tsconfig.json
```

## 2. CDK APP 이란?

registerAction 함수는 Codepipeline 액션을 등록하는 역할

ActionKind에 따라 적절한 액션을 생성하고 이를 파이프라인에 추가합니다.

이 함수는 다양한 종류의 액션을 처리하며, 각 액션에 대해 필요한 세부 정보를 포함하는 ActionProps 객체를 사용합니다.

actionKind.startsWith(ActionKindPrefix.Source)
액션의 종류가 Source로 시작하는지 확인합니다.

SourceCodeCommit , SourceS3Bucket 액션을 처리합니다.

