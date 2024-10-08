---
layout: default
title: JCASC plugin
nav_order: 3
permalink: docs/03_CICD/Jenkins/jenkins-jcasc-installation/jenkins-jcasc-installation
parent: Jenkins
grand_parent: CICD
---

# Jenkins Configuration as Code 플러그인을 사용하여 사용자 정의 구성이 적용된 Jenkins Docker 서비스 배포하기

{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---
## 글을 쓴 배경

단순한 Jenkins 배포가 아닌 유저 프로비저닝 등 다양한 젠킨슨 설정을 자동으로 구성할 필요성을 느끼게 되었습니다.

## 글 요약

**사용된 플러그인**

- **Jenkins Configuration as Code**:
- **Role-based-Authorization Strategy**:


**프로세스 요약**

1. userdata를 포함한 ec2 생성 하위는 userdata에 포함된 내용
1. 역할 기반 설정과 관리자 사용자의 자격 증명을 포함하는 jenkins.yml 파일을 생성합니다.
2. 환경 변수 CASC_JENKINS_CONFIG를 설정하여 첫 번째 단계에서 생성한 jenkins.yml 파일을 배치할 위치를 지정합니다.
3. jenkins.yml 파일을 CASC_JENKINS_CONFIG의 위치에 바인드 마운트합니다.
Jenkins 설정을 YAML 파일로 정의함으로써 사용자 지정 젠킨슨 구성을 자동화 합니다.

## 시작하기 전

Jenkins와 AWS에 대한 기본적인 사용 경험을 가진 DevOps 엔지니어를 대상으로 합니다.
Jenkins에 대한 기본적인 이해가 필요합니다.

참고 문서 : [configuration-as-code-plugin](https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos)

# EC2 인스턴스에 사용자 정의 구성내용을 포한한 Jenkins 인스턴스 배포

awscli를 활용한 ec2 배포 부분은 [앞에서 설명한 방식](https://blog.kimkm.com/docs/02_Tech/CICD/Jenkins/jenkins-installation)과 동일합니다.
본 내용에서는 최초 젠킨슨 이미지를 배포할 때 유저 프로비저닝 등 다양한 환경구성을 함께 적용하여 배포하는 방식을 공유합니다.

## awscli를 활용한 EC2 배포
```shell
aws ec2 run-instances \
  --image-id ${UBUNTU_AMI_ID} \
  --count 1 \
  --instance-type t3.large \
  --key-name ${KEY_NAME} \
  --iam-instance-profile Name=${INSTANCE_PROFILE_ROLE} \
  --subnet-id ${SUBNET_ID} \
  --security-group-ids ${JENKINS_SG} \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=jenkins-aws-cli-generate}]' \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=100}' \
  --user-data file://userdata.txt
``` 

## userdata를 이용한 Jenkins 자동배포

## userdata.txt

아래 스크립트는 EC2 인스턴스 생성 시 실행되며 Jenkins을 설치하기 위한 환경을 구축합니다.
~ 등으로 구성되었습니다.

```shell
#!/bin/bash

# 사용자 변수 설정
USER_NAME=ubuntu

# 사용자 디렉토리 생성 및 디렉토리 소유권변경
mkdir -p /home/ubuntu/jenkins/jenkins_home
mkdir -p /home/ubuntu/jenkins/data
chown -R ${USER_NAME}:${USER_NAME} /home/ubuntu/jenkins

# Docker 설치
echo "1. [docker program installation] start"
apt-get update -y
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
usermod -aG docker ${USER_NAME}
newgrp docker

# Docker 서비스 활성화 및 시작
systemctl enable docker
systemctl start docker

# Docker Compose 설치
apt list docker docker-compose
apt install docker-compose -y


cd ~/jenkins/data

# Jenkins Plugin 설치
su - ${USER_NAME} -c "cat > ~/jenkins/data/plugins.txt <<EOF
configuration-as-code:1775.v810dc950b_514
role-strategy:713.vb_3837801b_8cc
EOF"

## Jenkins Configuration as Code (JCasC) 정의
su - ${USER_NAME} -c "cat > ~/jenkins/data/jenkins.yaml <<EOF
jenkins:
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
            entries:
              - user: "user1"
              - user: "admin"
          - name: "readonly"
            permissions:
              - "Overall/Read"
            entries:
              - group: "authenticated"
              - user: "anonymous"
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "1234"
        - id: "user1"
          password: "1234"
EOF"

# Dockerfile 생성
su - ${USER_NAME} -c "cat > ~/jenkins/data/Dockerfile <<EOF
FROM jenkins/jenkins:jdk17
USER root
RUN apt-get update -y && apt-get upgrade -y
USER jenkins
COPY ./plugins.txt /usr/share/jenkins/plugins.txt
RUN  jenkins-plugin-cli -f /usr/share/jenkins/plugins.txt
COPY ./jenkins.yaml /usr/share/jenkins/jenkins.yaml
EOF"


# Docker-compose 파일 생성
su - ${USER_NAME} -c "cat > ~/jenkins/data/docker-compose.yml <<EOF
version: '3.9'
services:
  jenkins:
    build: ~/jenkins/data
    container_name: jenkins
    user: jenkins
    environment:
      JAVA_OPTS:
        -Djenkins.install.runSetupWizard=false
        -Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Seoul
      CASC_JENKINS_CONFIG: /usr/share/jenkins/jenkins.yaml
    restart: always
    ports:
      - \"80:8080\"
      - \"50000:50000\"
    volumes:
      - /home/ubuntu/jenkins/jenkins_home:/var/jenkins_home
EOF"

su - ${USER_NAME} -c "cd ~/jenkins/data; docker-compose up -d"
```


## userdata.txt 상세 설명

**사용자 변수 지정**
* Jenkins과 Docker를 실행할 사용자 이름을 설정합니다.
```bash
USER_NAME=ubuntu
 ```

**사용자 디렉토리 생성 및 디렉토리 소유권변경**
* jenkins_home 디렉토리 : Jenkins 어플리케이션 중요 데이터 저장소입니다. 컨테이너 내부의 핵심 데이터를 외부에 저장하여 컨테이너 재시작 시 데이터 손실 없이 재 가동할 수 있도록 합니다.
* data 디렉토리 : Jenkins 설치에 필요한 추가 파일을 보관합니다. 
* 디렉토리 소유권 변경: 생성한 디렉토리와 하위 모든 파일의 소유권을 ${USER_NAME}로 설정합니다.
```bash
mkdir -p /home/ubuntu/jenkins/jenkins_home
mkdir -p /home/ubuntu/jenkins/data
chown -R ${USER_NAME}:${USER_NAME} /home/ubuntu/jenkins
 ```

**Docker 설치 및 시작**
* 공식 Docker 설치 스크립트를 다운받아 실행합니다. 설정한 USER_NAME에 해당하는 사용자를 Docker 그룹에 추가합니다.
* 기본적으로 도커 실행 권한은 root에만 있습니다. ${USER_NAME} 사용자를 Docker 그룹에 추가하여, sudo 없이 Docker 명령을 실행할 수 있도록 합니다.
* systemctl을 사용하여 Docker 서비스를 시스템 부팅 시 자동으로 시작되도록 설정합니다.
```bash
# Docker 설치
echo "1. [docker program installation] start"
apt-get update -y
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
usermod -aG docker ${USER_NAME}
newgrp docker

# Docker 서비스 활성화 및 시작
systemctl enable docker
systemctl start docker
```

**Docker Compose 설치**
* docker-compse는 yml 파일에 Jenkins 설치 시 필요한 환경 변수와 볼륨 등 다양한 설정을 담음으로써, 
간단한 명령으로 docker를 실행시킬 수 있도록 합니다.
```bash
apt list docker docker-compose
apt install docker-compose -y
```

**Jenkins Plugin 설치**
* configuration-as-code:1775.v810dc950b_514 : jenkins.yml 파일을 사용하여 Jenkins 구성을 코드 형태로 자동화하고 관리할 수 있게 해줍니다.
* role-strategy:713.vb_3837801b_8cc: 사용자에게 역할 및 권한 부여를 통해 사용자의 접근 수준을 설정하여 Jenkins의 보안을 강화할 수 있습니다.


* 설치 후 아래와 같은 항목이 새롭게 생겨나게 됩니다.
![img-1.png](img-1.png)

```bash
cd ~/jenkins/data

su - ${USER_NAME} -c "cat > ~/jenkins/data/plugins.txt <<EOF
configuration-as-code:1775.v810dc950b_514
role-strategy:713.vb_3837801b_8cc
EOF"
```

**Jenkins Configuration as Code (JCasC) jenkins.yaml 파일 정의**

설치된 플러그인을 활용하여, Jenkins의 구성을 jenkins.yaml 파일로 자동화합니다. 


**설정내용**
* **역할 기반 권한 부여**: admin 역할에는 Jenkins의 전반적인 관리 권한(Overall/Administer)을 부여하고, readonly 역할에는 읽기 권한(Overall/Read)만 부여합니다.
* **사용자 관리**: securityRealm 설정을 통해 Jenkins 로그인에 필요한 사용자 계정을 생성합니다.


* **jenkins.yaml 적용전**
  * 최초 플러그인을 실행시 현재 접속해 있는 admin 유저만 권한을 가지게 설정되어있습니다.
![img-3.png](img-3.png)

* **jenkins.yaml 적용 후**
  * 새롭게 readonly 역할을 만들어 admin 유저가 아니더라도 로그인 된 인증된 유저가 젠킨슨 view 권한을 갖도록 
  * admin 권한에 생성한 유저를 추가할 수 있습니다.
  * readonly 역할에 anonymous 
![img-2.png](img-2.png)

```bash
su - ${USER_NAME} -c "cat > ~/jenkins/data/jenkins.yaml <<EOF
jenkins:
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
            entries:
              - user: "user1"
              - user: "admin"
          - name: "readonly"
            permissions:
              - "Overall/Read"
            entries:
              - group: "authenticated"
              - user: "anonymous"
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "1234"
        - id: "user1"
          password: "1234"
EOF"
```


**Docker File 생성**

이 파일은 Jenkins 서버의 이미지를 생성하고, 앞서 정의한 플러그인을 설치하며, jenkins.yaml 파일을 Jenkins Configuration as Code 플러그인이 파일을 수행할 수 있는 위치에 복사하는 역할을 합니다.

* 패키지 업데이트 및 업그레이드를 위해 root 사용자로 전환하여 이미지 내의 패키지를 최신 상태로 유지합니다.
* USER jenkins 명령어를 통해, 플러그인 설치와 관련된 작업을 root 가 아닌 Jenkins 사용자로 전환하여 실행합니다.
* jenkins-plugin-cli 를 통해 정의된 플러그인들을 설치합니다.
* Jenkins 이 정의된 jenkins.yaml 구성파일을 로드할 수 있도록 구성합니다.

```bash
su - ${USER_NAME} -c "cat > ~/jenkins/data/Dockerfile <<EOF
FROM jenkins/jenkins:jdk17
USER root
RUN apt-get update -y && apt-get upgrade -y
USER jenkins
COPY ./plugins.txt /usr/share/jenkins/plugins.txt
RUN  jenkins-plugin-cli -f /usr/share/jenkins/plugins.txt
COPY ./jenkins.yaml /usr/share/jenkins/jenkins.yaml
EOF"
```

**Docker Compose 파일 생성 및 실행**

* 컨테이너 빌드 및 설정: build: ~/jenkins/data: Dockerfile이 위치한 디렉토리 경로를 지정하여 Jenkins 이미지를 빌드합니다.

* container_name: 생성될 컨테이너의 이름을 jenkins로 설정합니다.

* user: 컨테이너 내에서 jenkins 유저로 프로세스를 실행합니다.

* 환경 변수 설정:

  * JAVA_OPTS: Jenkins 설치 마법사를 건너뛰고, 시간대를 설정합니다.
  * CASC_JENKINS_CONFIG: JCasC 플러그인이 jenkins.yaml 파일을 찾을 경로를 지정합니다.
  
* restart: Docker 호스트가 재부팅되거나 컨테이너가 멈췄을 때 자동으로 컨테이너를 재시작하도록 설정합니다.
* 포트 매핑: 호스트와 컨테이너 간의 포트를 매핑하여, 외부에서 Jenkins에 접근할 수 있도록 합니다. 
* 볼륨 마운트: - /home/ubuntu/jenkins/jenkins_home:/var/jenkins_home 옵션은 호스트의 jenkins_home 디렉토리를 컨테이너의 /var/jenkins_home에 마운트하여, Jenkins 데이터를 영구적으로 보존합니다.

```bash
# Docker-compose 파일 생성
su - ${USER_NAME} -c "cat > ~/jenkins/data/docker-compose.yml <<EOF
version: '3.9'
services:
  jenkins:
    build: ~/jenkins/data
    container_name: jenkins
    user: jenkins
    environment:
      JAVA_OPTS:
        -Djenkins.install.runSetupWizard=false
        -Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Seoul
      CASC_JENKINS_CONFIG: /usr/share/jenkins/jenkins.yaml
    restart: always
    ports:
      - \"80:8080\"
      - \"50000:50000\"
    volumes:
      - /home/ubuntu/jenkins/jenkins_home:/var/jenkins_home
EOF"

su - ${USER_NAME} -c "cd ~/jenkins/data; docker-compose up -d"
```


# jenkins 접속하기

http://<호스트의 IP 주소 또는 도메인>:8080 에 접속합니다.

앞서 설치한 플러그인으로 새롭게 생겨난 아래 두 섹션을 확인할 수 있습니다.

![img-1.png](img-1.png)

# Advanced Jenkins Configuration as Code

## 세분화된 Role-Based Access Control ( RBAC ) 설정

**Global roles**
![img-5.png](img-5.png)

* **적용 범위**: Jenkins 어플리케이션 전체에 대한 Role을 정의합니다. 예를 들어 "Administer" 권한을 가진 admin Role은 Jenkins 설정을 변경할 수 있습니다.

**Item roles**
![img-4.png](img-4.png)

* **목적**: 특정 Jenkins 항목(예: 작업, 뷰, 폴더)에 대한 접근 권한을 세분화하여 정의합니다.
* **적용 범위**: 특정 패턴이나 경로에 일치하는 항목에 대한 권한을 제어합니다. 이는 폴더 구조 내에서 미세한 접근 제어를 가능하게 하여, 특정 프로젝트 또는 팀의 작업에 대한 접근을 제한할 수 있습니다.
* **예시**: 'FolderA' 역할은 "A/." 패턴에 일치하는 모든 항목에 대한 특정 권한을 가지며, 'FolderB' 역할은 "B." 패턴에 일치하는 항목에 대해 다른 권한 세트를 가집니다.

### Agent roles

Jenkins에서 "Manage Roles" 페이지는 역할 기반 접근 제어(Role-Based Access Control, RBAC)를 설정하는 중요한 부분입니다. 여기서 관리자는 세 가지 유형의 역할을 정의할 수 있습니다: "Global roles", "Item roles", 그리고 "Agent roles". 이 역할들은 Jenkins 내에서 다양한 자원과 작업에 대한 접근 권한을 세밀하게 제어할 수 있게 합니다. 각 역할 유형의 차이점을 이해하는 것은 권한 관리와 보안 정책을 효과적으로 적용하는 데 중요합니다.

**Global Roles**

* **목적**: Jenkins 인스턴스 전체에 대한 권한을 정의합니다.
* **적용 범위**: Jenkins의 모든 부분에 걸쳐 일반적인 권한을 제공합니다. 예를 들어, "Administer" 권한을 가진 사용자는 Jenkins 설정을 변경할 수 있는 반면, "Read" 권한을 가진 사용자는 정보를 볼 수만 있습니다.
* **예시**: 'admin' 역할에는 시스템 전체를 관리할 수 있는 권한이 부여될 수 있으며, 'readonly' 역할에는 시스템 전체를 볼 수 있는 권한만 부여됩니다.

**Item Roles**

* **목적**: 특정 Jenkins 항목(예: 작업, 뷰, 폴더)에 대한 접근 권한을 세분화하여 정의합니다.
* **적용 범위**: 특정 패턴이나 경로에 일치하는 항목에 대한 권한을 제어합니다. 이는 폴더 구조 내에서 미세한 접근 제어를 가능하게 하여, 특정 프로젝트 또는 팀의 작업에 대한 접근을 제한할 수 있습니다.
* **예시**: 'FolderA' 역할은 "A/." 패턴에 일치하는 모든 항목에 대한 특정 권한을 가지며, 'FolderB' 역할은 "B." 패턴에 일치하는 항목에 대해 다른 권한 세트를 가집니다.

**Agent Roles**
* **목적**: Jenkins 에이전트에 대한 접근 권한을 정의합니다. Jenkins 에이전트는 빌드와 관련 작업을 실행하는 데 사용되는 시스템입니다.
* **적용 범위**: 특정 에이전트 또는 에이전트 그룹에 대한 접근 권한을 제어합니다. 이를 통해 어떤 사용자가 에이전트를 구성하거나 조작할 수 있는지 결정할 수 있습니다.
* **예시**: 'Agent1' 역할은 "agent1" 패턴에 일치하는 특정 에이전트에 대한 권한을 제어합니다.

각 역할 유형은 Jenkins 내에서 특정 자원에 대한 접근과 관리를 세밀하게 조절할 수 있는 유연성을 제공합니다. 이를 통해 조직은 필요에 따라 보안 정책을 세분화하고 엄격하게 적용할 수 있습니다.

```bash
jenkins:
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
            entries:
              - user: "admin"
          - name: "readonly"
            permissions:
              - "Overall/Read"
              - "Job/Read"
            entries:
              - user: "authenticated"
        items:
          - name: "FolderA"
            pattern: "A/.*"
            permissions:
              - "Job/Configure"
              - "Job/Build"
              - "Job/Delete"
            entries:
              - user: "user1"
              - user: "user2"
          - name: "FolderB"
            pattern: "B.*"
            permissions:
              - "Job/Configure"
              - "Job/Build"
            entries:
              - user: "user2"
        agents:
          - name: "Agent1"
            description: "Agent 1"
            pattern: "agent1"
            permissions:
              - "Agent/Build"
            entries:
              - user: "user1"
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "1234"
        - id: "user1"
          password: ""
        - id: "user_hashed"
          # password is password
          password: "#jbcrypt:$2a$10$3bnAsorIxhl9kTYvNHa2hOJQwPzwT4bv9Vs.9KdXkh9ySANjJKm5u"

```

# FAQ

Q: 설치된 젠킨슨에서 플러그인을 재대로 다운받지 못하면 어떻게 해야 하나요?

A: 보안 그룹 설정에 플러그인을 다운받기위한 HTTP, HTTPS 포트가 열려 있는지 확인합니다.




