---
layout: default
title: 2-1. TypeScript CDK로 재사용가능한 모듈 구조 작성하기 - Stack (3편)
nav_order: 21
permalink: docs/02_Tech/CICD/CDK/cdk-module-stack
parent: CICD
grand_parent: Tech
---

# 재활용 할 수 있는 CDK 모듈 생성하기 - Stack 3편

{: .no_toc }

## 목차
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 글을 쓴 배경

이 글은 AWS CDK를 사용하여 재사용 가능한 모듈 구조를 작성하는 방법을 설명하기 위해 작성되었습니다. 
stack 배포, 및 사용법을 다룹니다.

## 글 요약


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

## 2. Stack 이란

![img-1.png](../cdk-module-construct/img-1.png)

AWS Cloud Development Kit (AWS CDK) 스택은 리소스를 정의하는 하나 이상의 Construct의 모음입니다.
각 CDK 스택은 AWS CloudFormation 스택을 나타내며, 배포 시 스택 내의 리소스는 하나의 단위로 프로비저닝됩니다.

스택은 앱 내에서 정의되며, AWS CDK Stack 클래스를 사용하여 정의됩니다.

## 3. 재사용 가능한 코드를 위한 인터페이스 Stack의 필요성

`cdk.Stack` 클래스를 직접 사용할 수도 있지만, 프로젝트 복잡해질수록 코드의 중복을 줄이고 일관성을 유지하는 것이 필요합니다.
이러한 이유로 `lib/template/stack/base-stack.ts` 와 같은 재사용 가능한 베이스 스택을 정의하는 인터페이스를 도입하였습니다.

### 3.1 base-stack.ts 코드

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3'

import { AppContext } from '../../../app-context'
import { AppConfig, StackConfig } from '../../../app-config'
import { CommonHelper, ICommonHelper } from '../../common/common-helper'
import { CommonGuardian, ICommonGuardian } from '../../common/common-guardian'


export function Override(target: any, propertyKey: string, descriptor: PropertyDescriptor){}


// 클래스 내에서 일반적으로 사용되는 기본 스택 설정을 정의합니다.
// 이 설정은 애플리케이션 전체에서 공통적으로 사용될 기본 값들을 포함하며,
// 일반적으로 애플리케이션 시작 시 한 번 로드되어 여러 스택에 걸쳐 재사용됩니다.
export interface StackCommonProps extends cdk.StackProps {

    // 리소스 명명 시 사용
    projectPrefix: string;
    appConfig: AppConfig;
    appConfigPath: string;

    // 스택 생성 시 필요한 변수 목록
    variables: any;
}

export class BaseStack extends cdk.Stack implements ICommonHelper, ICommonGuardian {
    protected stackConfig: StackConfig;
    protected projectPrefix: string;
    protected commonProps: StackCommonProps;

    private commonHelper: ICommonHelper;
    private commonGuardian: ICommonGuardian;

    constructor(appContext: AppContext, stackConfig: any) {

        console.log("Received stackConfig:", stackConfig);

        let newProps = BaseStack.getStackCommonProps(appContext, stackConfig);

        // 스택 명칭 정하는 곳
        super(appContext.cdkApp, stackConfig.StackName, newProps);

        this.stackConfig = stackConfig;
        this.commonProps = newProps;

        // 현재 projectPrefix 설정 내용 `${projectName}-${projectStage}`;
        this.projectPrefix = appContext.stackCommonProps.projectPrefix;

        this.commonHelper = new CommonHelper({
            construct: this,
            env: this.commonProps.env!,
            stackName: this.stackName,
            projectPrefix: this.projectPrefix,
            variables: this.commonProps.variables
        });

        this.commonGuardian = new CommonGuardian({
            construct: this,
            env: this.commonProps.env!,
            stackName: this.stackName,
            projectPrefix: this.projectPrefix,
            variables: this.commonProps.variables
        });
    }

    private static getStackCommonProps(appContext: AppContext, stackConfig: any): StackCommonProps{

        // 특정 스택 인스턴스에 대한 설정을 수정하거나 확장할 때 사용됩니다.
        // 예를 들어, 특정 스택이 다른 리전에 배포되어야 하거나 특별한 환경 변수가 필요한 경우,
        // newProps는 이러한 요구 사항에 맞추어 조정될 수 있습니다.
        // 이는 BaseStack.getStackCommonProps 함수를 통해 appContext의 stackCommonProps를 기반으로 생성되지만,
        // stackConfig에 따라 일부 속성이 변경되거나 추가될 수 있습니다.
        let newProps = appContext.stackCommonProps;
        if (stackConfig.UpdateRegionName) {
            console.log(`[INFO] Region is updated: ${stackConfig.Name} ->> ${stackConfig.UpdateRegionName}`);
            newProps = {
                ...appContext.stackCommonProps,
                env: {
                    region: stackConfig.UpdateRegionName,
                    account: appContext.appConfig.Project.Account
                }
            };
        } else {
            console.log('not update region')
        }

        return newProps;
    }
    // ...
}
```

### 3.1.1 base-stack 기능 1 - 공통 설정 및 기능 중앙 집중화

앱 내의 여러 스택에서 공통적으로 사용되는 설정과 기능을 중앙에서 관리할 수 있습니다.  이를 통해 모든 스택이 일관된 설정을 가질 수 있도록 합니다.

BaseStack 클래스에 있는 `StackCommonProps` 인터페이스는 스택을 생성할 때 필요한 공통 속성을 정의합니다. 
해당 속성은 앞서 설명한 AppContext 클래스에서 불러온 설정파일을 통해 구성되게 됩니다.

[재활용 할 수 있는 CDK 모듈 생성하기 - app, context 2편](../cdk-module-context)

```typescript
export interface StackCommonProps extends cdk.StackProps {
    projectPrefix: string;
    appConfig: AppConfig;
    appConfigPath: string;
    variables: any;
}
```

### 3.1.2 base-stack 기능 2 - 재사용 가능한 헬퍼 메소드 제공

스택에서 자주 사용되는 기능을 헬퍼 메서드로 정의해 두면 각 스택에서 이를 재 사용할 수 있게 됩니다.
자세한 메서드 구현은 `lib/template/common/common-helper.ts` 에 구현되어있습니다. 

```typescript
export class BaseStack extends cdk.Stack implements ICommonHelper, ICommonGuardian {
    // ... other methods ...

    findEnumType<T extends object>(enumType: T, target: string): T[keyof T] {
        return this.commonHelper.findEnumType(enumType, target);
    }

    exportOutput(key: string, value: string, prefixEnable=true, prefixCustomName?: string) {
        this.commonHelper.exportOutput(key, value, prefixEnable, prefixCustomName);
    }

    putParameter(paramKey: string, paramValue: string, prefixEnable=true, prefixCustomName?: string): string {
        return this.commonHelper.putParameter(paramKey, paramValue, prefixEnable, prefixCustomName);
    }

    // ... other methods ...
}
```

### 3.1.3 base-stack 기능 3 - 코드 가독성 및 유지보수 향상

공통된 로직을 baseStack에 정의함으로써 각 스택 파일이 간결해지게 됩니다. 아래는 실제 codebuild 배포 스택입니다.

```typescript
import * as base from '../../../lib/template/stack/base/base-stack';
import { AppContext } from '../../../lib/app-context';
import * as codebuild from "../../pattern/codebuild/codebuild-simple-pattern";

export class CodeBuildSimplePatternStack extends base.BaseStack {

   private simpleCodeBuild: codebuild.CodeBuildSimplePattern;

   public onPostConstructor(codebuild: codebuild.CodeBuildSimplePattern): void {}

   constructor(appContext: AppContext, stackConfig: any) {
      super(appContext, stackConfig);

      // Initialize CodeBuildSimplePattern
      this.simpleCodeBuild = new codebuild.CodeBuildSimplePattern(this, 'CodeBuildSimplePattern', {
         stackName: this.stackName,
         projectPrefix: this.projectPrefix,
         env: this.commonProps.env!,
         stackConfig: stackConfig,
         variables: this.commonProps.variables,
      });

      this.onPostConstructor(this.simpleCodeBuild);
   }
}
```

## 4. base-stack을 이용한 실제 스택 구현 코드 상세 설명 (CodeBuildSimplePatternStack)

CodeBuildSimplePatternStack 클래스는 BaseStack을 상속받아 AWS CodeBuild 프로젝트를 정의합니다.

### 4.1 부모 객체 초기화

`CodeBuildSimplePatternStack` 클래스의 `constructor`에서 가장 먼저 부모 클래스(`BaseStack`)를 초기화합니다.

`infra/stack/codebuild/codebuild-simple-pattern-stack.ts`

```typescript
import { AppContext } from '../../lib/app-context';
import { BaseStack } from '../../lib/template/stack/base/base-stack';

export class CodeBuildSimplePatternStack extends BaseStack {
    constructor(appContext: AppContext, stackConfig: any) {
        super(appContext, stackConfig);
        // 추가 초기화 작업 수행
    }
// ...
}
```

### 4.2 BaseStack 클래스

마찬가지로 BaseStack 클래스는 cdk.Stack 클래스를 상속받아 구현되었습니다. 

아래는 `cdk.Stack` 클래스의 `constructor` 정의 입니다.

* **scope**: 스택의 상위 컨텍스트를 나타냅니다. 보통은 App 또는 Stage이지만, 다른 Construct도 될 수 있습니다.
* **id**: 스택의 고유 식별자입니다. 이 값이 스택의 물리적 ID를 결정하는 데 사용됩니다.
* **props:** 스택 속성을 포함하는 객체입니다. 여기에는 스택의 환경 설정, 설명, 태그 등이 포함될 수 있습니다.

`aws-cdk-lib/stack.d.ts`

```typescript
/**
 * Creates a new stack.
 *
 * @param scope Parent of this stack, usually an `App` or a `Stage`, but could be any construct.
 * @param id The construct ID of this stack. If `stackName` is not explicitly
 * defined, this id (and any parent IDs) will be used to determine the
 * physical ID of the stack.
 * @param props Stack properties.
 */
constructor(scope?: Construct, id?: string, props?: StackProps) {
    // CDK 애플리케이션에 스택을 등록하고 초기화 작업을 수행
}
```

### 4.3 CodeBuildSimplePattern 인스턴스 생성

BaseStack 클래스는 `StackCommonProps` 인터페이스를 사용하여 공통 속성을 정의하고,
정의된 속성을 이용해 `CodeBuildSimplePattern` 인스턴스를 만듭니다. 이 과정을 통해 각 스택은 일관된 설정을 가지게 됩니다.

`infra/stack/codebuild/codebuild-simple-pattern-stack.ts`
```typescript
// Initialize CodeBuildSimplePattern
this.simpleCodeBuild = new codebuild.CodeBuildSimplePattern(this, 'CodeBuildSimplePattern', {
   stackName: this.stackName,
   projectPrefix: this.projectPrefix,
   env: this.commonProps.env!,
   stackConfig: stackConfig,
   variables: this.commonProps.variables,
});
```


