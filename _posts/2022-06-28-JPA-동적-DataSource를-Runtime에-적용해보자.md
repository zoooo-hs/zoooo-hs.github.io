---
layout: post
title:  "JPA 동적 DataSource를 Runtime에 적용해보자"
excerpt: "EntityManagerFactory를 통해 적용가능하다."
date: "2022-06-28"


author: zoooo-hs
tags: [JPA,Spring, Spring boot, Spring Data JPA, DataSource, Dynamic-Datasource, Hibernate, Java]

---

* TOC
{:toc}

[게시글에 사용된 소스코드](https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/tree/main/2022-06-28-jpa-dynamic-datasource)

# 배경 및 문제점

회사에서 새로운 기능 구현에 있어 **매 서비스 요청마다 다양한 DataSource 에 DB Query를 실행**해야 할 일이 있었다. 제품은 Spring Boot + Spring Data JPA를 사용 중인데, 이미 단일 DataSource 에 연결되어 비즈니스 로직에 필요한 데이터들을 저장하고 불러오고 있었다. 이런 상황에서 기존의 DataSource 와 연결된 JPA Repository 및 기타 Configuration 수정 없이 새로운 기능을 구현할 방법을 찾아야 했다.

## 시도해 본 것

JPA Dynamic DataSource, JPA Change DataSource 등 JPA 를 사용하는 환경에서 여러 DataSource를 사용할 방법을 구글링해 보았다. 주로 나오는 글들은 HIbernate의 Multi tenancy 기능을 사용해 요청별 DataSource를 switching 하는 방법<sup>[1](#footnote_1)</sup>을 소개하거나, Repository 별 다른 DataSource를 주입해 사용하는 방법<sup>[2](#footnote_2)</sup>을 소개했다. 그러나 내 문제에 적용하기엔 문제가 있었다. 내 상황에 적용하지 못한 이유는

1. 미리 DataSource를 configuration에 저장해두고 사용할 수 없고 동적으로 다른 DataSource를 추가할 수 있어야 했다.
2. 위 방법들은 대체로 앱 전체에서 사용할 DataSource 설정을 다루기 때문에, 이미 기존에 사용하던 Main DataSource 설정을 수정할 수 없는 내 상황에 맞지 않았다.

# 해결 방법

그래도 앞서 시도해본 방법들에서 얻어간 게 없진 않다. 공통으로 **EntityManagerFactory를 생성할 때 DataSource를 주입**하는 과정이 있었다.

```java
@Bean
private LocalContainerEntityManagerFactoryBean productEntityManager(DataSource dataSource) {
	LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
	
	em.setDataSource(dataSource); // 핵심
	
	em.setPackagesToScan(new String[] { "com.base.pakcage.for.entity" });
	HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
	em.setJpaVendorAdapter(vendorAdapter);
	em.afterPropertiesSet();
	return em;
}
```

이러한 EntityManagerFactory를 이용하면 EntityManager를 만들 수 있는데, 사실 EntityManager만 있으면 JPA를 쓸 수 있다. 이런 점들을 문제 상황과 조합해보면 **매 서비스 요청마다 바뀌는 DataSource 정보에 맞춰 EntityManagerFactory를 만들어 EntityManager를 생성해 사용할 수 있다**는 것이다. 

DataSource는 다음과 같이 정의해 생성할 수 있다. 

```java
public DataSource productDataSource(String driverName, String url, String username, String password) {
	DataSourceBuilder dataSourceBuilder = DataSourceBuilder.create(this.getClass().getClassLoader());
	dataSourceBuilder.driverClassName(driverName)
	  .url(url)
	  .username(username)
	  .password(password);
	return dataSourceBuilder.build();
}
```

이제 아래처럼 EntityManager를 생성할 수 있다.

```java
EntityManagerFactory emf = productEntityManager(productDataSource(driverName, url, username, password))
															.getNativeEntityManagerFactory();
EntityManager em = emf.createEntityManager();
```

이젠 자유롭게 EntityManager를 통해 원하는 Entity의 영속성을 불러오고 저장하고 지지고 볶을 수 있다.

```java
SomeEntity someEntity = em.find(SomoeEntity.class, 1L);
```

Spring Data JPA의 JpaRepository가 아닌 **JPA의 EntityManager로 직접 영속성 관리를 해서, persist의 경우엔 직접 transaction을 잡아줘야 한다.**

```java
someEntity.setName("another name");

try {
	em.getTransaction().begin();
	em.persist(someEntity);
	em.getTransaction().commit();
}
catch (Exception e) {
	em.getTransaction().rollback();
}
```

이렇게 하면 요청마다 다른 Datasource여도 Query 실행이 가능하다. 그런데 조금 아쉽다.

## Spring Data JPA의 JpaRepository를 쓰고 싶다면

EntityManager 자체로도 이미 JPA를 사용할 수 있지만, 우리는 이미 Spring Data JPA에서 제공하는 JpaRepository의 꿀을 빨고 있는 상태이다. 사람이 서 있으면 앉고 싶고 앉으면 눕고 싶더라, 그래서 방법을 찾아보았다. 아래 코드<sup>[3](#footnote_3)</sup>로 EntityManager를 통해 JpaRepository 인스턴스를 만들어 쓸 수 있다. 

```java
SomeRepository someRepository = 
	new JpaRepositoryFactory(em).getRepository(SomeRepository.class);

SomeEntity someEntity = someRepository.findById(1L);
```

아직 명확한 이유를 찾지 못했지만, transaction 관리가 필요한 save (persist, merge를 요구하는)와 같은 메소드의 경우 EntityManager를 사용할 때와 마찬가지로 transaction을 직접 잡아줘야 한다.

```java
someEntity.setName("another name");

try {
	em.getTransaction().begin();
	someRepository.save(someEntity);
	em.getTransaction().commit();
}
catch (Exception e) {
	em.getTransaction().rollback();
}
```

이로써 JpaRepository 까지 사용 가능한 상황이 되었다. 

## 한계점, 이슈

- EntityManager를 통해 직접 transaction을 잡아줘야 한다. @Transactional이 먹히지 않는다.
    - 추측하건대 런타임에서 임의로 repository를 만들어 사용하기때문에 transcation과 관련된 AOP가 적용되지 않는 게 아닐까 싶다. 추가 조사 필요
    - @Transcational을 사용하면 기존에 Main Datasource로 잡아둔 Repository, Entity에만 적용된다.

# 부록: EntityManagerFactory

EntityManager는 매 서비스 메소드에 만들어 사용하고 서비스 메소드가 끝나면 close 해 사용하도록 권장된다.<sup>[4](#footnote_4)</sup> EntityManager를 만들기 위해서 EntityManagerFactory를 사용하는데 같은 Datasource에 대해서는 EntityManagerFactory를 한번 만들어서 여러 번 사용해도 괜찮다. 

오히려 EntityManagerFactory를 생성할 때 성능 이슈 <sup>[4](#footnote_4),[5](#footnote_5)</sup> (레퍼찾기)가 있으므로 같은 DataSource에 대해서는 하나의 EntityManagerFactory를 사용하는 게 좋다. 이를 위해 DataSource, EntityManagerFactory를 key value 쌍으로 메모리에 들고 있으면 중복하여 다시 생성하는 일이 없을 것이다.


# 참고자료
<ol>
  <li><a name="footnote_1">https://www.baeldung.com/hibernate-5-multitenancy</a></li>
  <li><a name="footnote_2">https://www.baeldung.com/spring-data-jpa-multiple-databases</a></li>
  <li><a name="footnote_3">https://stackoverflow.com/questions/22116005/how-to-create-jpa-repository-dynamically-inside-a-class/66564339#66564339</a></li>
  <li><a name="footnote_4">https://docs.jboss.org/hibernate/stable/entitymanager/reference/en/html_single/#d0e98</a></li>
  <li><a name="footnote_5">https://stackoverflow.com/questions/22116005/how-to-create-jpa-repository-dynamically-inside-a-class/66564339#66564339</a></li>
</ol>
