---
layout: post
title:  "AWS S3에 stataic web server 구성하기"
excerpt: "AWS S3에 내가 만든 웹 프로젝트를 올려보자"
date: "2022-04-02"

author: zoooo-hs
tags: [Frontend, Web, HTML, AWS, S3, Cloud, Deployment, React]

---

* TOC
{:toc}

지난주엔 [AWS Elastic Beanstalk을 통해 Spring boot 프로젝트를 배포](/2022/03/25/Spring-Boot-프로젝트를-AWS-Elastic-Beanstalk에-배포해보았다.html)하여, API 호출까지 확인해 보았다. 그렇다면 이번엔 프론트엔드에 해당하는 웹 정적 파일을 서비스하는 방법을 알아보고자 한다.


# 서론
이번 글에서는 AWS의 S3를 사용하여 정적 웹 페이지를 제공하는 방법을 소개한다. AWS S3는 클라우드 스토리지 서비스로, 원하는 파일을 업로드, 다운로드 할 수 있다. 기본적으로 API를 통해 오브젝트를 업로드, 다운로드 할 수 있지만 S3에서 제공하는 정적 웹 서버 호스팅 기능을 제공하여 특정 URL에 대해서 사용자가 지정한 HTML 및 그 외 정적 파일을 제공할 수 있게 구성할 수 있다. 이번 글에선 이러한 정적 웹 서버 호스팅 기능을 이용해 HTML, JS, CSS를 배포해본다. 아래와 같이 HTML, CSS, JS를 구성했다. 모든 예제 코드는 [github 레포지토리](https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/tree/main/2022-04-02-static-web-server)에서 확인할 수 있다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8" />
    <title>나의 멋진 웹 사이트</title>
    <link rel="stylesheet" href="style.css"/>
</head>
<body>
    <h1>반갑습니다!</h1>
    <p>저는 건강하시고 다정하신 부모님 아래 2남 중 장남으로 태어나 ... 블라블라 블라</p>
    <button id="push-button">눌러보고 싶네</button>
    <script src="script.js" ></script>
</body>
</html>
```
```css
p {
    font-size: 30px;
}
```
```js
document.getElementById("push-button").onclick = () => {
    alert("자.. 이게 클릭이야");
}
```

## 들어가기 앞서: React.js

React, Vue와 같이 웹 프론트엔드 라이브러리로 처음 웹을 공부하고 개발하면 HTML, JS, CSS 파일만으로 웹 프론트엔드를 배포하여 호스팅한다는 게 잘 와닿지 않을 수 있다. (모두가 그런 건 아니겠지만…. 적어도 내가 처음 웹을 배울 땐 그랬다….) 왜냐면 보통 이런 라이브러리들은 JS도 아닌 HTML도 아닌 파일에 JS와 HTML 심지어 CSS까지 같이 저장하고, 마법 같은 npm start 주문을 통해 localhost:3000에서 결과를 확인할 수 있기 때문이다. 자세한 내용을 알지 못하고 이런 식으로 개발을 진행하다 보면, 실제 배포를 어떻게 해야 할지 고민이 될 수 있다.

사실 npm start로 확인하는 방법 그대로 배포해도 웹 페이지를 제공할 순 있다. Elastic Beanstalk 혹은 순수 EC 2 인스턴스 + node.js 플랫폼 위에 프로젝트를 올려 npm start 명령을 실행시키면 된다. 이후 80 입력을 3000으로 프록시 하여 웹 페이지를 호스팅 할 순 있다. 그러나 우리가 직접 node.js 환경을 구축하고 소스 코드로 직접 서버를 구성해야 하는 번거로움이 있다. 바로 이때 우리가 항상 보고 지나친 명령인 npm build (혹은 yarn build)를 사용할 때이다. [create-react-app](https://create-react-app.dev)과 같은 각 라이브러리에 맞는 CLI로 프로젝트를 만들면 package.json에 build 명령이 정의되어 있을 것이다. 우리는 이 명령을 통해 우리 프로젝트를 HTML, JS, CSS 파일로 만들 수 있다.
```json
// package.json
...
  "scripts": {
    "start": "...",
    "build": "react-scripts build",
    ...
  },
...
```

react의 경우 npm build를 통해 build/ 디렉토리가 생성되는데, 여기에 있는 index.HTML을 브라우저를 통해 열면 우리가 npm start를 통해 보면 화면이 그대로 나온다. index.html외에도 여러 JS 파일들이 존재하는데 babel과 webpack의 설정에 따라 알맞게 알아보기 어려운 형태로 JS 파일이 구성되어 있을 것이다. 여기엔 이제 우리가 각 컴포넌트 및 정의한 함수들이 일반 JS로 변경되어 담겨있다. 우리는 이러한 build/ 디렉토리에 있는 파일을 s3에 올려 정적 웹 서버를 호스팅 하면 된다!
```bash
npm build
...
ls build/
asset-manifest.json favicon.ico         logo.png            robots.txt
default-profile.png index.html          manifest.json       static
```


# S3 버킷 만들기
S3는 Bucket이라는 단위로 여러 오브젝트 스토리지를 구성할 수 있는데, 사용자는 각 프로젝트에 맞게 오브젝트 스토리지를 구성하여 사용할 수 있다. 우리는 이번 정적 웹 호스팅을 위한 S3 Bucket을 구성한다. 

## 이름 짓기
- 버킷 생성 첫 화면에서 버킷 이름을 지정해준다.
![이름 짓기](/assets/img/20220402/1-main.png)
- 버킷 이름은 URL 구성에도 사용되기 때문에 전 세계 버컷의 이름과 겹치면 안 된다. 중복 체크를 진행하기 때문에 잘 선택하면 된다. 사실 이후에 https, domain 연결을 하게 되면 위 내용은 크게 중요하지 않지만, 그래도 프로젝트 성격에 맞게 알맞은 이름을 선택하면 된다.
- 리전은 ap-norhteast-2(서울)을 선택한다.



## 퍼블릭 엑세스 허용
- [모든 퍼블릭 액세스 차단] 항목을 체크 해제한다.
![퍼블릭 액세스 허용](/assets/img/20220402/2-public-access.png)
- 다른 사람이 URL을 통해 우리 웹 서버에 요청을 보내야 하기 때문에 해당 항목을 체크 해제한다.


## 버킷 생성 완료
- 다른 항목은 수정하지 않고 최하단에 [버킷 만들기] 버튼을 눌러 생성을 완료한다.
![버킷 생성 완료](/assets/img/20220402/3-create-finish.png)


# 정적 웹 호스팅 허용

## 상세 페이지 및 속성 화면
- 완료 후 S3 콘솔 메인 화면에서 생성한 버킷을 확인 할 수 있다. 해당 버킷을 클릭해보자.
![버킷 생성 완료 결과](/assets/img/20220402/4-finish-result.png)

- 클릭하면 다음과 같은 버킷 상세 화면을 볼 수 있는데, 여러 탭 중 [속성]을 클릭한다.
![버킷 상세 화면 속성](/assets/img/20220402/5-detail-properties.png)

- 속성 페이지 하단에 [정적 웹 사이트 호스팅] 항목이 있는 것을 확인 할 수 있다. [편집] 버튼을 클릭해보자.
![정적 웹 사이트 호스팅 항목](/assets/img/20220402/6-static-website-hosting.png)

## 정적 웹 사이트 호스팅
- 첫 번째 항목인 [정적 웹 사이트 호스팅]의 라디오 버튼 중 [활성화]를 선택한다. [호스팅 유형]은 [정적 웹 사이트 호스팅]을 그대로 유지한다.
![정적 웹 사이트 호스팅 활성화](/assets/img/20220402/7-static-website-hosting-activation.png)

- 조금 내려가보면 [인덱스 문서] 항목이 있는데 이 항목에 "index.html"을 입력한다. 이후 설정을 완료한다.
![정적 웹 사이트 호스팅 인덱스문서](/assets/img/20220402/8-static-website-hosting-index-doc.png)
- 이 항목은 앞으로 우리의 정적 웹서버에 / URL 요청을 보냈을 때 보내줄 HTML파일을 의미한다.
- React, Vue 와 같은 SPA를 위한 라이브러리로 개발한 프로젝트는 build 결과로 나온 index.html을 입력하면 된다.

## 파일 업로드
- 이제 다시 버킷 상세페이지를 열어 이번엔 [객체] 탭을 선택한다. 이후 [업로드] 버튼을 클릭한다.
![파일 업로드](/assets/img/20220402/9-updload.png)
- [파일 추가] 버튼을 통해 정적 웹 서버에서 제공할 HTML, CSS, JS를 업로드한다.
![파일 업로드 화면](/assets/img/20220402/10-upload-page.png)

## URL 확인하기
- 이제 다시 버킷 상세 페이지 중 [속성] 탭으로 이동한다. 이전에 확인한 [정적 웹 사이트 호스팅] 항목에 [버킷 웹 사이트 엔드포인트] 항목이 생성된 것을 알 수 있다.
![버킷 웹 사이트 엔드포인트](/assets/img/20220402/11-bucket-web-endpoint.png)
- 지금까지 모든 설정의 결과 여기에 바로 접속해보면 index.html이 나와야 할 것 같다!!

# 그런데? ..
- 403?? 내가 만든 웹페이지 접근이 거절당했다.
![403](/assets/img/20220402/12-403.png)
- 이는 우리가 버킷에 GetObject 기능을 열어주지 않아서 그렇다. URL 접근은 열어두었지만, 해당 URL로 요청할 수 있는 기능들을 열지 않아서 그렇다.

# 버킷 정책 생성
- 위 문제를 해결하기 위해서는 버킷 정책을 수정해야 한다. 버킷 상세 페이지 중 [권한] 탭을 선택하여 아래에 있는 [버킷 정책] 항목을 확인한다.
![버킷 정책](/assets/img/20220402/13-bucket-policy.png)

- 아무런 정책이 없는 상태이다. [편집] 버튼을 클릭하여, 아래와 같은 [버킷 정책 편집] 페이지로 이동한다.
![버킷 정책 편집](/assets/img/20220402/14-bucket-policy-edit.png)

- 정책은 JSON으로 선언할 수 있는데, 어떤 항목을 채워야 하는지 우리가 알 수 없어서 AWS에서 [정책 생성기]를 제공한다. [정책 생성기] 버튼을 클릭하자.


## 정책 생성기
- 우리는 정책 생성기를 통해 S3 버킷의 오브젝트에 접근 할 수 있는 GetObject의 권한만 열어준다. 
![버킷 정책 편집](/assets/img/20220402/15-bucket-policy-edit-1.png)
- [Step 1: Select Policy Type] 에서 S3 Bucket Policy 선택
- [Step 2: Add Statement(s)]
  - Effect : Allow
  - Principal: *
  - AWS Service: 그냥 둡니다
  - Actions: GetObject를 찾아 선택합니다.
  - ARN: 버킷 정책 편집 페이지에 [버킷 ARN]이 있는데 이를 복사해 붙여넣습니다. 이때 붙여넣은 ARN 뒤에 "/*" 를 추가한다.
    - ex) arn:aws:s3:::static-web-test-zoooo/*

- 이후 [Add Statement] 버튼을 클릭하면, 아래에 Statement가 추가된다. 이후 [Generate Policy] 버튼을 클릭하면 생성한 정책이 선언된 JSON이 나오는데 해당 내용을 복사해둔다.
![버킷 정책 생성](/assets/img/20220402/16-bucket-policy-created.png)
![버킷 정책 생성](/assets/img/20220402/17-bucket-policy-created-1.png)

- 이후 정책 편집기 화면에서 복사한 내용을 붙여넣기 하고 [변경 사항 저장] 버튼을 클릭하면 정책이 갱신된다.

# 진짜 끝
- 이제 다시 접속해보면 잘 나온다.
![끝](/assets/img/20220402/18-end.png)
