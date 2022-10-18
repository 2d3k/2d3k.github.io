---
title: Git Actions 배포 자동화
tags: AWS Deploy
---



# 사전 준비

> 1. [Spring Initializr](https://start.spring.io/)을 사용해 포로젝트 준비
>
> 2. AWS S3 버킷 비우기(객체 영구 삭제, 정적 웹 사이트 호스팅 비활성화)



---



# Github Actions?

> 빌드, 테스트 및 배포 파이프라인을 자동화할 수 있는 **CI/CD 플랫폼**
>
> 워크플로는 `.yml`에 의해 구성된다.



---



# 리소스 설정



### 1. Github Actions 생성

![repo](https://user-images.githubusercontent.com/102038283/196362770-e2d0112a-c43a-44fc-8878-a1643aeb6adf.png)

> 1) 리포지토리를 생성하고, Actions를 누른다.
>
> 2. Java wuth Gradle로 생성한다.
>
>    
>
>    ![image-20221018162132050](C:\Users\32150944\AppData\Roaming\Typora\typora-user-images\image-20221018162132050.png)
>
> 3. gradle.yml을 변경하지 않고 Commit new file을 클릭한다.
>
> 4. Actions에서 워크플로의 단계별 진행상황을 볼 수 있다.
>
> 5. 지금은 yml을 설정하지 않았기 때문에 실패가 나온다.



---



### 2. Github Action 수정

![2](https://user-images.githubusercontent.com/102038283/196364397-a1def12a-95d0-42fe-98d0-5b426ea95088.png)



> 1. 리포지토리의 Settings 클릭
> 2. Security>Actions 선택
> 3. New repository secret으로 액세서 키 생성



- 보안을 위해 yml 파일에 직접 입력하는 것이 아니라 sercret으로 생성
- Name에는 변수 이름을, Value엔 변수의 값을 저장
- IAM User를 생성할 때 있는 액세스 키 ID 값과, 비밀 액세스 키 값을 각각 저장



---



### 3. 빌드파일 배포 및 실행을 위한 준비

![EC2 2](https://user-images.githubusercontent.com/102038283/196368578-8ee75b2f-b6ce-4960-a907-70be4edf34d6.png)

> 1. 애플리케이션 생성
> 2. Github Actions 워크플로에서 사용해야 되기 때문에 규칙에 맞게 작성



![3](https://user-images.githubusercontent.com/102038283/196370045-b874d75e-d51e-4f95-b251-6cdcf4d6cebc.png)



![app 2](https://user-images.githubusercontent.com/102038283/196370290-a2b4238d-9bf2-4a75-bcf7-00a1c90d25f3.png)



> 3. 배포 그룹 생성
>
> 4. 로드 밸런싱 비활성화
>
> 5. 배포 그룹 생성



---



### 4. gradle.yml 수정

```java
# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.
# This workflow will build a Java project with Gradle and cache/restore any dependencies to improve the workflow execution time
# For more information see: https://help.github.com/actions/language-and-framework-guides/building-and-testing-java-with-gradle


name: Java CI with Gradle

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

permissions:
  contents: read

# 본인의 S3 이름으로 설정
env:
  S3_BUCKET_NAME: be-26-2d3edith

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Set up JDK 11
      uses: actions/setup-java@v3
      with:
        java-version: '11'
        distribution: 'temurin'
    # gradle script의 권한 문제를 해결하기 위해 파일을 실행할 수 있는 권한을 부여
    - name: Run chmod to make gradlew executable
      run: chmod +x ./gradlew
    - name: Build with Gradle
      uses: gradle/gradle-build-action@67421db6bd0bf253fb4bd25b31ebb98943c375e1
      with:
        arguments: build

      # build한 후 프로젝트를 압축합니다.
    - name: Make zip file
      run: zip -r ./practice-deploy.zip .
      shell: bash

      # Access Key와 Secret Access Key를 통해 권한을 확인
      # 아래 코드에 Access Key와 Secret Key를 직접 작성하지 않음
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
		# 등록한 Github Secret을 자동으로 불러옴
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }} 
		# 등록한 Github Secret을 자동으로 불러옴
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }} 
        aws-region: ap-northeast-2

    # 압축한 프로젝트를 S3로 전송
    - name: Upload to S3
      run: aws s3 cp --region ap-northeast-2 ./practice-deploy.zip s3://$S3_BUCKET_NAME/practice-deploy.zip
        
    # CodeDeploy에게 배포 명령을 내림
    # CodeDepky 애플리케이션 이름, 배포그룹 이름 확인!!!
    - name: Code Deploy
      run: >
        aws deploy create-deployment --application-name be-26-2D3Edith
        --deployment-config-name CodeDeployDefault.AllAtOnce
        --deployment-group-name be-26-2D3Edith-group
        --s3-location bucket=$S3_BUCKET_NAME,bundleType=zip,key=practice-deploy.zip
```



- env와 CodeDeploy에서 이름을 제대로 입력하지 않으면 오류가 발생합니다.



![S3](https://user-images.githubusercontent.com/102038283/196367764-e79d2fc2-1612-4c48-93f2-51d87c8de13a.png)

> 워크플로가 성공적으로 완료되면, S3 버킷에 압축파일이 전송됩니다!



---



### 5. `.yml` 파일 설정



> **appspec.yml**
>
> 경로:  최상위 디렉토리

```java
version: 0.0
os: linux
files:
  - source:  /
    destination: /home/ubuntu/action
    overwrite: yes

permissions:
  - object: /
    pattern: "**"
    owner: ubuntu
    group: ubuntu

hooks:
  ApplicationStart:
    - location: scripts/deploy.sh
      timeout: 60
      runas: ubuntu
```



>  **deploy.sh**
>
> 경로:  scripts> deploy.sh



```java
#!/bin/bash
BUILD_JAR=$(ls /home/ubuntu/action/build/libs/practice-githubAction-deploy-0.0.1-SNAPSHOT.jar)
JAR_NAME=$(basename $BUILD_JAR)

echo "> 현재 시간: $(date)" >> /home/ubuntu/action/deploy.log

echo "> build 파일명: $JAR_NAME" >> /home/ubuntu/action/deploy.log

echo "> build 파일 복사" >> /home/ubuntu/action/deploy.log
DEPLOY_PATH=/home/ubuntu/action/
cp $BUILD_JAR $DEPLOY_PATH

echo "> 현재 실행중인 애플리케이션 pid 확인" >> /home/ubuntu/action/deploy.log
CURRENT_PID=$(pgrep -f $JAR_NAME)

if [ -z $CURRENT_PID ]
then
  echo "> 현재 구동중인 애플리케이션이 없으므로 종료하지 않습니다." >> /home/ubuntu/action/deploy.log
else
  echo "> kill -9 $CURRENT_PID" >> /home/ubuntu/action/deploy.log
  sudo kill -9 $CURRENT_PID
  sleep 5
fi


DEPLOY_JAR=$DEPLOY_PATH$JAR_NAME
echo "> DEPLOY_JAR 배포"    >> /home/ubuntu/action/deploy.log
sudo nohup java -jar $DEPLOY_JAR >> /home/ubuntu/deploy.log 2>/home/ubuntu/action/deploy_err.log &
```



---



## 배포 로그 확인



```java
$ cd	#home으로 이동
$ cd action
$ ll	#deploy.log와 deply_err.log파일을 확인
```



