---
layout: default
title: 2-1. TypeScript CDK로 재사용가능한 모듈 구조 작성하기 - Construct (4편)
nav_order: 21
permalink: docs/02_Tech/03_CICD/CDK/cdk-module-construct
parent: 03_CICD
grand_parent: Tech
---

# 재활용 할 수 있는 CDK 모듈 생성하기 - Construct 4편

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

## 2. Construct 란?

![img-1.png](img-1.png)

AWS Cloud Development Kit (AWS CDK)의 요소인 Construct는 AWS 리소스를 구성하고 관리하는 기본 단위입니다.

[construct hub](https://constructs.dev/packages/aws-cdk-lib/v/2.142.1?lang=typescript)을 통해 cdk resource를 쉽게 찾고 활용할 수 있습니다.

## 3. Construct 사용법

`Construct`는 AWS CDK 앱의 구성 요소로, 다음과 같은 방식으로 사용됩니다:

### 3.1 Construct를 인스턴스화 하여 사용

AWS CDK에서는 이미 정의된 Construct를 직접 인스턴스화하여 사용할 수 있습니다. 
예를 들어, `s3.Bucket` 같은 `Construct`를 바로 인스턴스화하여 사용할 수 있습니다

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

class MyBucketStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new s3.Bucket(this, 'MyFirstBucket', {
      versioned: true,
    });
  }
}

const app = new cdk.App();
new MyBucketStack(app, 'MyBucketStack');
```

### 3.2 기존 Construct 조합하기

여러 Construct를 조합하여 더 복잡한 구성 요소를 만들 수 있습니다. 
예를 들어, S3 버킷과 SNS Topic을 조합하여 특정 이벤트에 대한 알림을 설정할 수 있습니다.

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as s3notifications from 'aws-cdk-lib/aws-s3-notifications';

class MyNotificationBucketStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const bucket = new s3.Bucket(this, 'MyBucket');
    const topic = new sns.Topic(this, 'MyTopic');

    bucket.addEventNotification(s3.EventType.OBJECT_CREATED, new s3notifications.SnsDestination(topic));
  }
}

const app = new cdk.App();
new MyNotificationBucketStack(app, 'MyNotificationBucketStack');
```

### 3.3 직접 construct 작성하기

새 구문을 선언하려면 `Construct` 기본 클래스를 확장하는 클래스를 만들고, 초기화 할때 필요한 매개변수를 전달해 주면 됩니다. 

다음은 Construct를 작성하고 배포하는 방법에 대한 예제입니다.

### 3.4 CodeBuildSimplePattern Construct 예제

이 예제 클래스에서는 `Construct`를 확장하여 CodeCommit 리포지토리, IAM 역할, 정책, S3 버킷을 포함하는 `CodeBuildSimplePattern`을 정의합니다. 
이를 통해 복잡한 워크로드를 하나의 커스텀 `Construct`로 구성하고 배포할 수 있습니다.

```typescript
import * as codebuild from 'aws-cdk-lib/aws-codebuild';
import * as codecommit from 'aws-cdk-lib/aws-codecommit';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as path from "node:path";
import * as fs from "node:fs";
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as s3check from "../../pattern/s3/s3-check-pattern";
import { CodeBuildSimplePatternProps , Policy} from "../../common/interfaces"

export class CodeBuildSimplePattern extends Construct {
    public readonly repository: codecommit.Repository;
    public Bucket: s3.IBucket;
    private stackConfig: any;
    private projectPrefix: string;

    constructor(scope: Construct, id: string, props: CodeBuildSimplePatternProps) {
        super(scope, id);

        this.stackConfig = props.stackConfig;
        this.projectPrefix = props.projectPrefix;

        const projectName = `${props.projectPrefix}-${this.stackConfig.ProjectName}`
        const repositoryName = `${props.projectPrefix}-${this.stackConfig.RepositoryName}`;
        const roleName = `${props.projectPrefix}-${this.stackConfig.RoleName}`;
        const policyName = `${props.projectPrefix}-${this.stackConfig.PolicyName}`;
        const bucketName = `${props.projectPrefix}-${this.stackConfig.BucketName}`;

        // Create CodeCommit Repository
        this.repository = new codecommit.Repository(this, repositoryName, {
            repositoryName: repositoryName,
            description: this.stackConfig.RepositoryDescription,
        });

        const iamRole = new iam.Role(this, roleName, {
            assumedBy: new iam.ServicePrincipal('codebuild.amazonaws.com'),
            roleName: roleName,
        });

        const policyPath = path.join(__dirname, "codebuild-pol.json");
        const policyData: { Statement: Policy[] } = JSON.parse(fs.readFileSync(policyPath, 'utf8'));

        const Policy = new iam.Policy(this, policyName, {
            policyName: policyName,
        });

        policyData.Statement.forEach((policy) => {
            Policy.addStatements(new iam.PolicyStatement({
                effect: iam.Effect[policy.Effect as keyof typeof iam.Effect],
                resources: policy.Resource,
                actions: policy.Action,
            }));
        });

        iamRole.attachInlinePolicy(Policy);

        // Set output values
        new cdk.CfnOutput(this, roleName, { value: iamRole.roleArn });

        // Initialize and use CheckS3Pattern
        this.initializeBucket(props, bucketName, iamRole);
    }

    private async initializeBucket(props: CodeBuildSimplePatternProps, bucketName: string, iamRole: iam.IRole) {
        const checkS3Pattern = new s3check.CheckS3Pattern(this, bucketName, { bucketName: bucketName });
        this.Bucket = await checkS3Pattern.bucket;
        this.createCodeBuildProject(this.stackConfig.Name, this.repository, iamRole, this.Bucket, props);

    }

    private createCodeBuildProject(
        projectName: string,
        repository: codecommit.IRepository,
        role: iam.IRole,
        bucket: s3.IBucket,
        props: CodeBuildSimplePatternProps
    ): codebuild.Project {
        return new codebuild.Project(this, projectName, {
            projectName: projectName,
            source: codebuild.Source.codeCommit({ repository }),
            environment: {
                buildImage: codebuild.LinuxBuildImage.AMAZON_LINUX_2_5,
            },
            role,
            cache: codebuild.Cache.bucket(bucket, {
                prefix: `cache/${projectName}`,
            }),
            timeout: cdk.Duration.minutes(30),
        });
    }
}
```

#### 3.4.1 CodeBuildSimplePattern 클래스 코드 상세 설명

**생성자**

* **scope** - 현재 `Construct`가 어디에 포함되는지를 정의합니다.
* **id** - Construct의 고유 식별자입니다. 같은 scope 내에서 Construct를 구분하기 위한 이름입니다. 동일한 종류의 여러 리소스를 생성할 때 유용합니다.
* **props** - 생성자에 전달된 추가 매개변수들을 포함합니다.

```typescript
constructor(scope: Construct, id: string, props: CodeBuildSimplePatternProps) {
    super(scope, id);
```

**CodeBuildSimplePattern 클래는 아래와 같은 작업을 수행합니다.**

1. CodeCommit 리포지토리 생성: 새로운 코드 저장소를 생성합니다.
2. IAM 역할 및 정책 설정: CodeBuild가 사용할 역할을 생성하고, 설정된 정책을 연결합니다.
3. S3 버킷: CodeBuild에서 캐시 버킷으로 사용할 버킷을 조회하고 없으면 생성합니다.
4. CodeBuild 프로젝트 생성: CodeCommit 리포지토리, IAM 역할, S3 버킷을 사용하여 CodeBuild 프로젝트를 생성하는 워크로드를 구성합니다.

### 4. BaseConstruct 활용

`construct`도 stack과 마찬가지로 construct간의 공통된 속성을 적용하기 위해 BaseConstruct를 사용할 수 있습니다.