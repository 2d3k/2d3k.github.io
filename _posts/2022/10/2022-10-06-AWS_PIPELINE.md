---
title: AWS Pipeline 배포 자동화
tags: AWS Deploy
---



#  CodeDeploy-Agent 설치 `(EC2 인스턴스)`

<br>

## EC2 개발 환경 구축
<br>

### 1. JAVA 설치

> 1. 패키지 정보를 업데이트
>
>    `sudo apt update`
>
>    
>
> 2. java 설치
>
>    `sudo apt install openjdk-11-jre-headless`
>
>    
>
> 3. 설치 확인
>
>    `java -version`
>    자바 버전이 정상적으로 나오면 설치 완료



<br>

### 2. AWS CLI 설치



설치

```
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ sudo apt install unzip
$ unzip awscliv2.zip
$ sudo ./aws/install
```



>설치 확인
>
>`aws --version`
>
>역시 버전이 보인다면 설치 완료!

<br>

### 3. CodeDeploy Agent 설치



>설치
```
sudo apt update
sudo apt install ruby-full
sudo apt install wget
cd /home/ubuntu
sudo wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
sudo chmod +x ./install
sudo ./install auto > /tmp/logfile
```



> 설치 확인
>
> `sudo service codedeploy-agent status`
>
> `active(runnung)`가 있다면 정상 설치 후 실행중이다!



---



# EC2 인스턴스 역할 부여

<br>
### 1. IAM 역할 수정



> 인스턴스 선택 -> 보안 -> IAM
>
> ![EC2](https://user-images.githubusercontent.com/102038283/195581871-b9e45870-2490-410e-ad33-0f2b176a08e8.png)



### 2. IAM 권한 추가

> ![IAM](https://user-images.githubusercontent.com/102038283/195581883-4df9004b-5dca-4197-bab2-3831de4d11a7.png)



### 3. 신뢰관계 편집

> ![IAM2](https://user-images.githubusercontent.com/102038283/195581886-bedcea3d-78d1-44a8-8627-f9f87bac6974.png)



---



# EC2를 사용한 파이프라인 구축

<br>

### 1. 파이프라인에 올릴 소스코드 준비 `(로컬)`
<br>

🚨주의할 점🚨
>   단순한 문제로 시간이 많이 끌릴 수 있다. 실수하지 않도록 조심해야 된다.
> 1. 파일명에서 오타가 나지 않게 주의한다.
> 2. 경로를 제대로 확인한다

<br>

appspec.yml 파일 추가

> 배포 자동화를 도와주는 CodeDeploy-Agent가 인식하는 파일
>
> 경로: [프로젝트명]/DeployServer

```java
version: 0.0
os: linux

files:
  - source: /
    destination: /home/ubuntu/build

hooks:
  BeforeInstall:
    - location: server_clear.sh
      timeout: 3000
      runas: root
  AfterInstall:
    - location: initialize.sh
      timeout: 3000
      runas: root
  ApplicationStart:
    - location: server_start.sh
      timeout: 3000
      runas: root
  ApplicationStop:
    - location: server_stop.sh
      timeout: 3000
      runas: root
```
<br>

buildspec.yml 파일 추가

> 배포 자동화에서 빌드를 담당하는 CodeBuild-Agent가 인식하는 파일
>
> 경로: [프로젝트명]/DeployServer

```java
version: 0.2

phases:
  install:
    runtime-versions:
      java: corretto11
  build:
    commands:
      - echo Build Starting on `date`
      - cd DeployServer
      - chmod +x ./gradlew
      - ./gradlew build
  post_build:
    commands:
      - echo $(basename ./DeployServer/build/libs/*.jar)
artifacts:
  files:
    - DeployServer/build/libs/*.jar
    - DeployServer/scripts/**
    - DeployServer/appspec.yml
  discard-paths: yes
```



---



쉘 파일 생성

> 최상위에 scripts 디렉토리를 생성 -> initialize.sh, server_clear.sh, server_start.sh, server_stop.sh
>
> 파일 4개를 생성

<br>
scripts/initialize.sh
```java
#!/usr/bin/env bash
chmod +x /home/ubuntu/build/**
```



scripts/server_clear.sh

```java
#!/usr/bin/env bash
rm -rf /home/ubuntu/build
```



scripts/server_start.sh

```java
#!/usr/bin/env bash
cd /home/ubuntu/build
sudo nohup java -jar DeployServer-0.0.1-SNAPSHOT.jar > /dev/null 2> /dev/null < /dev/null &
```



scripts/server_stop.sh

```java
#!/usr/bin/env bash
sudo pkill -f 'java -jar'
```



---



# AWS CodeDeploy 대시보드



### 1. 애플리케이션 생성



> <span style="font-size:120%"> AWS CodeDeploy 대시보드 -> 애플리케이션 -> 애플리케이션 생성 </span>
> ![EC2 2](https://user-images.githubusercontent.com/102038283/195588155-031a6de4-68c4-4c0d-bbf1-50b3e3a9497f.png)

<br>

>  <span style="font-size:120%"> 이름입력 -> EC2/온프레스 옵션 선택 -> 애플리케이션 생성 </span>



---



### 2. 배포 그룹 생성

>  <span style="font-size:120%"> 배포 그룹 선택 -> 배포 그룹 생성 </span>
> ![app](https://user-images.githubusercontent.com/102038283/195592658-7d883aac-57bf-45c1-88d9-19fa5e68a912.png)

<br>

>  <span style="font-size:120%"> 배포 그룹 이름 입력(리소스이름-group) -> 서비스 역할-> IAM 역할 선택 (EC2 인스턴스에 연결O)  </span>

<br>

>  <span style="font-size:120%"> 환경구성 -> EC2 인스턴스 선택 -> 태그 그룹 -> Name 태그 -> 발급받은 인스턴스 선택   </span>
> ![app 2](https://user-images.githubusercontent.com/102038283/195591696-48490dc2-6ccd-4c35-924c-0ced1332985d.png)

<br>

>  <span style="font-size:120%"> 로드 밸런싱 활성화 `해제` -> 배포 그룹 생성  </span>



---



# 파이프라인 구축



### 1. 파이프라인 설정 선택
>  <span style="font-size:120%"> AWS CodePipeline -> 파이프라인 생성 </span>
> ![pipe](https://user-images.githubusercontent.com/102038283/195593564-f95bd5b9-7d08-47f5-84d4-e3945096bc50.png)



>  <span style="font-size:120%"> 파이프라인 이름과 역할 이름을 규칙에 맞게 입력 </span>



---



### 2. 소스 스테이지 추가

> <span style="font-size:120%"> 소스: Github(버전2) </span>
> ![pipe2](https://user-images.githubusercontent.com/102038283/195594131-362b6532-e87e-4c8c-8081-21c90c7d9704.png)



>  <span style="font-size:120%"> GitHub 연결 버튼 -> 연결 이름 임의로 입력 -> 연결 </span>

<br>

> <span style="font-size:120%"> 새 앱 설치 클릭 </span>
> ![pipe3](https://user-images.githubusercontent.com/102038283/195594855-6e69db42-c8f9-4f2e-acff-1353f7432bdc.png)



>  <span style="font-size:120%"> github 계정 클릭 -> Only select repositories -> 소스코드 사용할 리포지토리 선택 -> Install 클릭 </span>

<br>

>  <span style="font-size:120%"> 리포지토리와 브랜치 지정 -> CodePipeline 기본값  선택 -> 다음 </span>



---



### 3. 빌드 스테이지 추가



>  <span style="font-size:120%"> 빌드 공급자 -> AWS CodeBuild 선택 </span>
> ![build](https://user-images.githubusercontent.com/102038283/195595786-6ade70fb-4464-4d28-a160-198894cf46a2.png)



>  <span style="font-size:120%"> 프로젝트 생성 -> 프로젝트 이름 입력 </span>

<br>

> <span style="font-size:120%"> 환경설정 </span>
> ![build2](https://user-images.githubusercontent.com/102038283/195596428-70857d7f-ee7a-4921-8120-1ff71bd63789.png)



>  <span style="font-size:120%"> Buildspec 이름: "DeployServer/buildspec.yml </span>

<br>

>  <span style="font-size:120%"> CodePipeline으로 계속 -> 다음 클릭 </span>



---



### 4. 배포 스테이지 추가



> <span style="font-size:120%"> 빨간 상자 안에 정보를 모두 입력 </span>
> ![pipe_deploy](https://user-images.githubusercontent.com/102038283/195597120-7a304b59-6abe-4b1a-bf2f-79d503e48440.png)

 

> <br><span style="font-size:120%"> 설정을 확인 -> 파이프라인 생성 </span>
> ![pipe_deploy2](https://user-images.githubusercontent.com/102038283/195597651-0b578baa-bb9e-4999-ba71-2d418aa4492b.png)

<br>

>  <span style="font-size:130%"> 파이프라인 생성 & 배포 자동 실행!!! </span>
> ![pipe_deploy3](https://user-images.githubusercontent.com/102038283/195597904-e80f5e0e-1735-4a03-aac7-636839b81b9e.png)



---



## 파이프라인 deploy 관련 로그 확인



```
$ cd  (경로가 홈에 있어야 됩니다)
$ cd /opt/codedeploy-agent/deployment-root/deployment-logs
```


log 파일 읽는 방법

> 1. Vim이나 nano와 같은 편집기를 사용
> 2. cat [파일명]: cat 명령어를 사용해서 파일을 바로 읽기



---

























