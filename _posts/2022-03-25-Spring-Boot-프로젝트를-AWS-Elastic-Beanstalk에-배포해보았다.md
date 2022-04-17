---
layout: post
title:  "Spring Boot 프로젝트를 AWS Elastic Beanstalk에 배포해보았다"
excerpt: "AWS Elastic Beanstalk으로 쉽게 spring boot app 배포"
date: "2022-03-25"


author: zoooo-hs
tags: [Spring Boot, AWS, Elastic Beanstalk, Cloud, Deployment]

---

* TOC
{:toc}

# 들어가기 앞서
- 본 내용은 시행착오 과정도 포함하고 있어, 무작정 따라가기만 하면 같은 문제를 만날 수도 있습니다.
- [게시글에 사용된 소스 코드](https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/tree/main/2022-03-25-eb-test)

# 테스트 프로젝트
본 글에서는 GET /hello 요청이 오면 "Hello World" 문자열을 반환하는 API 앱을 만들고 AWS Elastic Beanstalk으로 배포한다. 이를 위한 Spring Boot 프로젝트를 하나 만들고 다음과 같은 컨트롤러를 추가한다.
```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String helloWorld() {
        return "Hello World";
    }
}
```




# Elastic Beanstalk(이하 eb) Console
- [AWS Management Console](https://aws.amazon.com/ko/console/)에서 Elastic Beanstalk을 검색하여 해당 관리 콘솔로 이동한다.

![Beanstalk 검색](/assets/img/20220325/eb-a-search.png)

# 새 환경 생성
- 화면 오른쪽에 주황색 *새 환경 생성* 버튼을 클릭한다.
![Beanstalk 메인](/assets/img/20220325/eb-console-main.png)

- 웹 서버 환경 선택
![Beanstalk 환경 티어 선택](/assets/img/20220325/eb-create-1.png)

- 애플리케이션 이름 및 환경 이름 입력
![Beanstalk 앱 이름](/assets/img/20220325/eb-create-2.png)
![Beanstalk 환경 이름](/assets/img/20220325/eb-create-3.png)

- 관리형 플랫폼 선택
    - Java
    - JDK 11
![Beanstalk 플랫폼](/assets/img/20220325/eb-create-4.png)

- 애플리케이션 코드: 코드 업로드 선택
    - 로컬 파일 선택 후 
![Beanstalk 애플리케이션 코드](/assets/img/20220325/eb-create-5.png)

- 프로젝트를 bootJar로 빌드한 jar 파일 선택 후 환경 생선 클릭
![Beanstalk 파일 업로드](/assets/img/20220325/eb-create-6.png)

- 환경 구성까지 어느정도 시간이 소요된 뒤 다음과 같은 화면이 나오면서 환경 구성 끝
![Beanstalk 업로드 완료](/assets/img/20220325/eb-create-7.png)
![Beanstalk 환경 생성 로딩](/assets/img/20220325/eb-create-8.png)

- 새로고침 옆에 있는 URL이 우리 앱에 요청 보낼 수 있는 URL이다.
![Beanstalk 환경 생성 롼료](/assets/img/20220325/eb-create-9.png)

# 배포 후 앱 테스트
이렇게 Spring Boot를 eb에 배포했다. 그러나 앞서 만든 GET /hello를 테스트해보면 다음과 같은 결과가 나온다.
- 502 Bad Gateway
![GET /hello error](/assets/img/20220325/eb-deploy-1.png)

로컬에서 이런 일이 있으면 로그를 통해 원인을 알 수 있다. eb 콘솔에서 앱의 로그를 확인할 수 있는데, 마지막 100줄 혹은 전체 로그를 확인 할 수 있다. 이번엔 마지막 100줄 결과를 보도록 한다.

eb에서 보여주는 로그는 앱의 로그와 nginx의 로그가 있다. java 플랫폼 기반 eb에서는 nginx와 jdk 환경이 갖춰진 ec2 인스턴스에 사용자의 앱 패키지를 올리는 방식으로 구성되기 때문에, http 요청에 대해서 nginx로그를 보거나 앱의 로직 오류는 앱 로그를 통해 확인 할 수 있다.

- 콘솔에서 로그를 보는 방법
![GET /hello error log](/assets/img/20220325/eb-deploy-2.png)
![GET /hello error log 다운로드 완료](/assets/img/20220325/eb-deploy-3.png)

- GET /hello 요청에 502가 발생
![GET /hello error log 화면](/assets/img/20220325/eb-deploy-4.png)
/hello 말고도 / 에도 오류가 발생하지만, 이 부분은 뒤에서 다시 보도록 하고 일단 /hello에 대해서 좀 더 자세히 로그를 살펴보도록 한다.


- http://127.0.0.1:5000/hello ?
![GET /hello error log 5000 포트](/assets/img/20220325/eb-deploy-5.png)

eb에서는 기본적으로 80 포트로 온 요청을 nginx를 통해 [localhost:5000 으로 리버스 프록시하고 있다](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/java-se-nginx.html). 그러나 스프링 부트 프로젝트는 기본적으로 8080 포트를 사용하여 http 요청을 listen한다. 이러한 포트 차이로 우리의 요청이 스프링 앱에 전달되지 못한 것이다. 이를 해결하는 방법은 두 가지가 있는데
1. applciation.properties의 server.port를 5000으로 설정
2. eb 설정에서 환경 속성으로 SERVER_PORT 5000을 설정

1번의 경우 프로젝트 파일을 고쳐서 다시 재배포 해야 하기 때문에 2번을 통해 바꿔보도록 한다. eb의 환경 속성 중 SERVER_PORT를 설정하면 nginx에서 프록시하는 포트를 바꿀 수 있다고 한다.

- 콘솔에서 앱의 환경 속성을 설정하기 위해서는 다음 메뉴를 선택한다.
![eb console env](/assets/img/20220325/eb-deploy-6.png)

- 환경 속성에서 다음과 같이 SERVER_PORT를 설정 후 적용
![eb console env 설정](/assets/img/20220325/eb-deploy-7.png)

- 환경 속성을 적용하면 앱이 재시작된다.
![eb console env 적용 완료](/assets/img/20220325/eb-deploy-8.png)

- 앱이 재시작되고 다시 요청을 보내보면 잘 응답한다.
![eb console env 적용 후 응답](/assets/img/20220325/eb-deploy-9.png)

## GET / 404 error

GET /hello에 대해서는 잘 응답하지만, 여전히 로그에선 GET / 에 대해선 404 에러를 보인다. 이는 당연한 게 우리가 만든 코드에는 GET /에 대한 컨트롤러 메소드가 없기 때문이다. 우리가 직접 보낸 요청도 아닌데 왜 이렇게 많은 GET / 요청이 존재할까?
- GET /404 error
![get / 404 error](/assets/img/20220325/eb-deploy-10.png)

404 에러와 함께 적혀있는 ELB-HealthChecker로 알 수 있듯, 이는 **eb 환경을 구축하며 함께 생성된 로드밸런서의 인스턴스 health check 기능이다.** 로드밸런서에서 주기적으로 인스턴스가 살아있는지 확인하는 요청을 보내는 것이다. 200 OK 응답을 보이면 살아있다고 판단하는 것이다. 프로젝트로 돌아와 다음과 같은 코드를 추가하고, jar 패키징 후 앱을 재배포해 보자. 앱 재배포는 콘솔에서 업로드 및 배포 버튼을 눌러 다시 jar를 업로드 할 수 있다.


```java
@RestController
public class HealthController {
    @GetMapping("/")
    public String healthCheck() {
        return "건강합니다^^";
    }
}
```

- 앱이 재시작되고 난 뒤에 상태가 기분 좋은 초록 아이콘으로 바뀌었다.
![get / 200](/assets/img/20220325/eb-deploy-12.png)

- 로그에서도 GET / 요청에 잘 응답하는 것을 알 수 있다.
![get / 200 log](/assets/img/20220325/eb-deploy-13.png)

## 그럼 GET / 요청은 무조건 health check로 사용해야 하나요?
그렇지 않다. 로드밸런서 설정에서 health check URL을 변경할 수 있다.
- 먼저 ec2 콘솔에서 로드밸런서항목 중 대상 그룹을 선택한다.
![ec2 target group](/assets/img/20220325/eb-deploy-14.png)

[대상 그룹](https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-type)은 로드밸런서에서 부하를 전달하는 대상을 말하는데, AWS 람다, 인스턴스, IP 등이 있다. eb 환경을 생성할때 ec2 인스턴스와 해당 인스턴스를 대상 그룹으로 부하를 전달하는 로드밸런서가 생성되는데, 우리는 해당 타겟 그룹의 health check 설정을 바꿔줘야 한다.



위 그림과 같이 검색란에서 우리가 만든 환경의 이름을 입력해 필터링하고, 대상 그룹을 선택한다.

- 중간에 있는 탭 메뉴 중 Health Check를 선택하면 Path가 /로 지정되어 있는걸 볼 수 있다. Edit을 선택하여 편집
![ec2 target group setup](/assets/img/20220325/eb-deploy-15.png)

- Path를 원하는 값으로 변경 후 적용
![ec2 target group setup health check](/assets/img/20220325/eb-deploy-16.png)


이제 프로젝트에서도 변경한 경로에 맞는 health check 컨트롤러 메소드로 수정한다.
```java
@RestController
public class HealthController {
    @GetMapping("/custom/health/path/you/want")
    public String healthCheck() {
        return "건강합니다^^";
    }
}
```

- 이후 앱 재배포 후 로그를 확인하면 바뀐 경로로 health check가 잘 적용된 걸 확인할 수 있다.
![eb setup fininshed](/assets/img/20220325/eb-deploy-17.png)

### 서비스: spring-boot-starter-actuator
spring boot 의존성으로 actuator를 추가할 수 있는데, 원래는 스프링 앱의 상태 모니터링 정보를 API로 제공한다. 그중에 /actuator/health와 같이 앱의 health check를 위한 API도 존재하기 때문에 위처럼 코드를 직접 작성하지 않아도 health check API를 추가할 수 있다.
```gradle
// build.gradle
 ...
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
 ...
```
기본 actuator API의 prefix는 /actuator인데 아래 설정으로 원하는 prefix를 부여할 수 있다.
```properties
server.servlet.contextPath: /some/prefix
```
이후 대상 그룹 설정에서 변경한 path에 맞춰 health check 를 수정해주면 된다.

# 끝
이렇게 Spring Boot 프로젝트를 AWS Elastic Beanstalk으로 배포하는 방법을 정리해보았다. 중간의 시행착오를 제외하면 매우 간단한 방법으로 앱을 배포하고 테스트할 수 있는 환경을 제공받는다. Elastic Beanstalk은 Spring Boot뿐만 아니라 node, python 등 여러 환경을 제공하기 때문에 추후 다른 환경의 앱을 배포할 때도 유용하게 사용할 수 있다.

# 참고자료
- https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/java-se-nginx.html
- https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-type