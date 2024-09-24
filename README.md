# <p align="center"> Docker-Image-Optimization
### 도커 이미지 최적화 실습
---

<h2 style="font-size: 25px;"> 개발팀원👨‍👨‍👧‍👦<br>
<br>

|<img src="https://avatars.githubusercontent.com/u/127727927?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/90971532?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/98442485?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/66353700?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|:-:|
|[@부준혁](https://github.com/BooJunhyuk)|[@이승언](https://github.com/seungunleeee)|[@신혜원](https://github.com/haewoni)|[@이연희](https://github.com/LeeYeonhee-00)|

---

<br>

## 프로젝트 목적 🌷
서비스를 Docker Image로 빌드하기 전, Docker Image 최적화 방법을 학습하였습니다.<br>
그 중 3가지 방법을 실제 적용하여 빌드 속도와 이미지 크기가 줄여보는 것을 목표로 진행하였습니다.

<br>

## Docker image 최적화 방법 🔎
1. 최소한의 베이스 이미지 선택
2. 단일 책임 원칙 → 각 도커 이미지는 하나의 역할만 수행하는 것이 좋음
3. 다중 스테이비 빌드 → 빌드와 컴파일 스테이지 분할하는게 빌드 타임 줄임
4. layer 줄이기 → 하나의 RUN 명령어에 묶어서 layer를 줄이기
5. .dockerignore 파일 사용 → 불필요한 파일을 .dockerignore 파일에 작성해서 빌드 시간 줄이기
6. 특정 tag 사용 → latest tag를 사용하는 것보다 특정 버전 지정하는게 더 빠름
7. dockerfile instruction 최적화
8. artifact 압축 → 바이너리나 정적 파일과 같은 artifact를 이미지 빌드 전에 압축하기
9. ```docker history``` | ```docker inspect``` 명령어를 사용해서 불필요한 부분들 미리 확인
10. ```docker system prune``` 을 사용하여 불필요한 자원 삭제

<br>

## 실습 과정💻
: <br>
### case 1. 최소한의 베이스 이미지 선택

#### Dockerfile.original
```
FROM openjdk:17
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
RUN javac Main.java
CMD ["java", "Main"]
```

#### build 결과
![image](https://github.com/user-attachments/assets/ba5483b7-11f1-4015-b172-922a01d79f13)


#### Dockerfile.apline
```
FROM openjdk:17-jdk-alpine
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
RUN javac Main.java
CMD ["java", "Main"]
```

#### build 결과
![image](https://github.com/user-attachments/assets/e039b21d-7a3b-41c6-a895-978654800376)

#### 결론
최소한의 Base image를 사용하였을 때 빌드 시간이 줄어든 것을 확인할 수 있었다.
openjdk:17 (27.3s) -> 17-jdk-alpine (22.8s)
<br>

----
### case 2. .dockerignore 파일 사용

#### Dockerfile
```
FROM openjdk:17
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
RUN javac Main.java
CMD ["java", "Main"]
```

#### .dockerignore
```
largefile.bin
Main.class
```

#### build 결과
![image](https://github.com/user-attachments/assets/1c9e7a6b-fb73-401d-aa92-79deec2149c6)
![image](https://github.com/user-attachments/assets/97660c8c-97fa-429c-bb24-b2365c582a80)

#### 결론

`.dockerignore` 파일을 적절히 사용하면 Docker 빌드 성능을 크게 향상시킬 수 있습니다.<br>
특히 대용량 파일이나 불필요한 파일들이 포함된 프로젝트에서는 **빌드 시간을 단축**하고, **빌드 컨텍스트 전송 크기**를 줄이며, 최종적으로 **이미지 크기**까지 감소시키는 효과를 얻을 수 있습니다.

이번 실험에서는 작은 규모의 프로젝트로 인해 큰 차이가 나타나지 않았지만, 대규모 프로젝트에서 `.dockerignore`를 활용하면 다음과 같은 장점이 더욱 명확해질 것입니다.

- **빌드 성능 최적화**: 빌드 시간 단축 및 리소스 절약
- **이미지 크기 최소화**: 불필요한 파일이 포함되지 않아 더 작고 효율적인 Docker 이미지 생성
- **보안성 강화**: 민감한 파일을 빌드 과정에서 제외하여 보안 향상

따라서,`.dockerignore`를 사용하는 것을 추천하며, 프로젝트의 크기와 복잡성에 상관없이 더 빠르고 효율적인 개발 환경을 만들 수 있습니다.
<br>


### case 3. multi-stage 사용

### 실습환경 

```
git clone https://github.com/code-rks/docker-demo.git
cd docker-demo/

출처: https://www.linkedin.com/pulse/optimize-docker-builds-rohit-kumar-shaw/
```


### Dockerfile.v1
```
FROM node:18 as build
WORKDIR /usr/code
COPY package* .
RUN npm install && npm i -g serve
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["serve", "-s", "build", "-l", "3000"]
```

Dockerfile.v1 파일은 현재
build stage만으로 구성 되어 있습니다.
정상적으로 프로젝트 빌드를 진행하지만
production환경에 필요없는 빌드 결과물도 복사되며
이미지 파일의 용량을  증가시킵니다.


### Dockerfile.v2
```
FROM node:18 as build
WORKDIR /usr/code
COPY package* .
RUN npm install
COPY . .
RUN npm run build

FROM node:alpine as main
WORKDIR /usr/code
COPY --from=build /usr/code/build /usr/code/build
RUN npm i -g serve
EXPOSE 5000
CMD ["serve","-s","build","-l","5000"]
```

Dockerfile.v2 파일의 경우 
main 스테이지가 추가되었습니다.
main stage에서는 Build 과정을 완료한 이후
production 환경에 필수적인 파일들만
COPY명령어를 통해 복사하여
최종 이미지에 필수적인 파일들만 포함되도록 하였습니다


#### 이미지 생성 후 size 비교 
![image](https://github.com/user-attachments/assets/7a2cc2d4-27d2-4bbe-b3e0-fcb844f14c1f)



## 참고 자료
https://overcast.blog/docker-image-optimization-tips-tricks-6a17f687162b <br>
https://faun.pub/reduce-the-size-of-the-docker-image-e6895b653419 <br>
https://www.linkedin.com/pulse/optimize-docker-builds-rohit-kumar-shaw/ <br>

