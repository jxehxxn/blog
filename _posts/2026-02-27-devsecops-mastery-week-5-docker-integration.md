---
layout: post
title: "DevSecOps Mastery: 5주차 - Docker와 Jenkins, 완벽한 커플"
---

DevSecOps 엔지니어 여러분, 환영합니다.
지금까지 우리는 Jenkins가 설치된 서버(또는 컨테이너)에 직접 Node.js나 Java를 설치해서 빌드를 돌렸습니다.
이 방식에는 치명적인 단점이 있습니다.

> "프로젝트 A는 Node 14가 필요한데, 프로젝트 B는 Node 18이 필요해!"
> "젠킨스 에이전트에 파이썬 깔려 있어? 버전은 뭐야?"

이른바 **Dependency Hell(의존성 지옥)**입니다. 오늘은 **Docker**를 이용해 이 지옥에서 탈출하고, 언제나 깨끗한 **Ephemeral(일회용)** 빌드 환경을 만드는 법을 배웁니다.

---

## 🏗️ Why Docker in CI?

도커를 CI 파이프라인에 도입하면 두 가지 혁명이 일어납니다.

1.  **도구 설치 불필요:** Jenkins 에이전트에 `npm`, `mvn`, `python`을 설치할 필요가 없습니다. 그냥 도커 이미지만 있으면 됩니다.
2.  **일관성 (Consistency):** 개발자 노트북에서 돌던 환경과 CI 서버 환경이 100% 일치하게 됩니다. "내 컴퓨터에선 되는데요?"라는 변명이 사라집니다.

---

## 🐳 Docker Agent 사용하기

Jenkinsfile에서 `agent any` 대신 `agent { docker ... }`를 사용하면 마법이 일어납니다.

```groovy
pipeline {
    agent {
        docker { 
            image 'node:18-alpine' 
            // 젠킨스가 자동으로 현재 작업 공간(Workspace)을 
            // 컨테이너 안에 마운트합니다.
        }
    }
    stages {
        stage('Check Version') {
            steps {
                // 이 명령어는 node:18 컨테이너 안에서 실행됩니다.
                sh 'node --version' 
            }
        }
    }
}
```
빌드가 끝나면 컨테이너는 자동으로 삭제됩니다. 항상 깨끗한 상태에서 시작하는 것이죠.

---

## 🚢 Building Docker Images (도커 빌드 자동화)

이제 파이프라인 안에서 애플리케이션을 도커 이미지로 구워봅시다.
이것이 바로 현대적인 배포의 시작입니다.

### 1. Dockerfile 작성
프로젝트 루트에 `Dockerfile`을 만듭니다.

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

### 2. Jenkinsfile 수정

```groovy
pipeline {
    agent any // 여기선 도커 명령어를 써야 하니 호스트(또는 도커가 설치된 에이전트)를 씁니다.
    
    stages {
        stage('Build Image') {
            steps {
                script {
                    // 도커 이미지 빌드
                    sh 'docker build -t my-app:v1 .'
                }
            }
        }
        stage('Test Image') {
            steps {
                // 이미지가 잘 떴는지 잠깐 실행해보기
                sh 'docker run -d --name test-container my-app:v1'
                sh 'docker ps | grep test-container'
                sh 'docker rm -f test-container'
            }
        }
    }
}
```

---

## 🛠️ 실습: Containerized Build

1.  여러분의 프로젝트에 간단한 `Dockerfile`을 추가하세요.
2.  Jenkinsfile을 수정하여 `node:18` 이미지를 에이전트로 사용하여 `npm install`을 수행해보세요.
3.  그 다음 단계에서 `docker build` 명령어로 애플리케이션 이미지를 만들어보세요.

(주의: Jenkins를 Docker로 띄웠다면, **Docker-in-Docker (DinD)** 또는 **Docker-out-of-Docker (DooD)** 설정이 필요합니다. 우리는 실습 1주차에서 소켓 볼륨 마운트(`-v /var/run/docker.sock:/var/run/docker.sock`)를 했다고 가정합니다.)

---

## 📝 5주차 과제: 다른 베이스 이미지 사용하기

Jenkinsfile의 `agent` 섹션을 수정하여 `python:3.9` 이미지를 사용하도록 변경하세요.
그리고 `python --version`을 출력하여 파이썬 3.9 버전이 맞는지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** CI 파이프라인이 끝날 때마다 빌드 환경(컨테이너)을 삭제하고 새로 만드는 방식을 무엇이라고 하나요? (영어로 E로 시작)
2.  **Q2.** Jenkins 파이프라인 안에서 도커 이미지를 에이전트로 사용할 때 쓰는 문법은 무엇인가요? `agent { ??? { image ... } }`
3.  **Q3.** 도커 컨테이너 안에서 다시 도커 명령어를 실행하기 위해 사용하는 기술적 용어(약어)는 무엇인가요?

---

다음 주는 **CI(지속적 통합)**의 꽃, **테스트 자동화**와 **아티팩트 관리**를 배웁니다.
만들어진 도커 이미지가 정말 안전한지, 기능은 정상인지 검증하지 않으면 쓰레기를 배포하는 것과 같으니까요.

**Containerize Everything.**
