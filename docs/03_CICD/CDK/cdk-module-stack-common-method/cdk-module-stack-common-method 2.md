---
layout: default
title: 2-1. TypeScript CDK로 재사용가능한 모듈 구조 작성하기 - Stack Advanced (5편)
nav_order: 21
permalink: docs/02_Tech/03_CICD/CDK/cdk-module-stack-advanced
parent: 03_CICD
grand_parent: Tech
---

# 재활용 할 수 있는 CDK 모듈 생성하기 - Stack Advanced 5편

{: .no_toc }

## 목차
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 글을 쓴 배경

이 글은 AWS CDK를 사용하여 재사용 가능한 모듈 구조를 작성하는 방법을 설명하기 위해 작성되었습니다. 
stack 배포, infra 및 lib 디렉토리 역할, 사용법을 다룹니다.

## 글 요약

이 글에서는 CDK 프로젝트의 진입점인 stack custom interface인 base-stack 인터페이스와 bin 디렉토리의 역할을 설명하고,
CDK 컨텍스트 값이 무엇인지, 어떻게 사용되는지에 대해 다룹니다.

## 시작하기 전

이 글을 읽기 전 [재활용 할 수 있는 CDK 모듈 생성하기](../cdk-module-context)을 우선 읽어주시길 바랍니다.


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

## 2. BaseStack 활용하기

기존 사용되고 있는 `BaseStack`의 주요 역할은 스택별 공통 설정 기능을 제공합니다.

## 3. CfnIncludeStack 클래스

CfnIncludeStack 클래스는 BaseStack을 확장하여 CloudFormation 템플릿을 포함하고, 
이를 기반으로 스택을 구성하는 예제입니다.

```typescript
import * as cfn_inc from 'aws-cdk-lib/cloudformation-include';
import * as base from './base-stack';
import { AppContext } from '../../../app-context';

export interface CfnTemplateProps {
    templatePath: string;
    parameters?: any;
}

export abstract class CfnIncludeStack extends base.BaseStack {
    private cfnTemplate?: cfn_inc.CfnInclude;

    abstract onLoadTemplateProps(): CfnTemplateProps | undefined;
    abstract onPostConstructor(cfnTemplate?: cfn_inc.CfnInclude): void;

    constructor(appContext: AppContext, stackConfig: any) {
        super(appContext, stackConfig);

        const props = this.onLoadTemplateProps();

        if (props != undefined) {
            this.cfnTemplate = this.loadTemplate(props);
        } else {
            this.cfnTemplate = undefined;
        }

        this.onPostConstructor(this.cfnTemplate);
    }

    private loadTemplate(props: CfnTemplateProps): cfn_inc.CfnInclude {
        const cfnTemplate = new cfn_inc.CfnInclude(this, 'cfn-template', {
            templateFile: props.templatePath,
        });

        if (props.parameters != undefined) {
            for (let param of props.parameters) {
                const paramEnv = cfnTemplate.getParameter(param.Key);
                paramEnv.default = param.Value;
            }
        }

        return cfnTemplate;
    }
}

```

### 3.1 onLoadTemplateProps 메소드

이 추상 메서드는 사용자가 CloudFormation 템플릿 경로와 필요한 매개변수를 제공하도록 합니다. 사용자는 이 메서드를 구현하여 필요한 설정을 반환해야 합니다.

### 3.2 onPostConstructor 메서드

이 추상 메서드는 스택 생성 후 추가적인 설정을 수행하기 위해 사용됩니다. 사용자는 이 메서드를 구현하여 스택 생성 후 필요한 작업을 정의할 수 있습니다.

### 3.3 loadTemplate 메서드

이 메서드는 CloudFormation 템플릿을 로드하고, 필요한 경우 매개변수를 설정합니다.

### 3.4 사용 예제

`SampleCfnVpcStack` 클래스

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as cfn_inc from 'aws-cdk-lib/cloudformation-include';

import * as base from '../../../lib/template/stack/base/cfn-include-stack';
import { AppContext } from '../../../lib/app-context';
import { Override } from '../../../lib/template/stack/base/base-stack';
import { CfnTemplateProps } from '../../../lib/template/common/interfaces'


export class SampleCfnVpcStack extends base.CfnIncludeStack {

    constructor(appContext: AppContext, stackConfig: any) {
        super(appContext, stackConfig);
    }

    @Override
    onLoadTemplateProps(): CfnTemplateProps | undefined {
        return {
            templatePath: this.stackConfig.TemplatePath,
            parameters: this.stackConfig.Parameters
        };
    }

    @Override
    onPostConstructor(cfnTemplate?: cfn_inc.CfnInclude) {
        const cfnVpc = cfnTemplate?.getResource('VPC') as ec2.CfnVPC;

    }
}
```


