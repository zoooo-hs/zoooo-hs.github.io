---
layout: post
title:  "JPA Entity cascade=REMOVE, orphanRemoval=true 왜 헷갈렸을까 - 2"
excerpt: "cascade는 자식으로 작업 전이, orphanRemoval은 관계가 끊어질때 자식을 삭제"
date: "2022-03-18"


author: zoooo-hs
tags: [JPA, Java, Spring Data JPA, Hibernate]

---

* TOC
{:toc}

# 앞서
- [시리즈에 사용된 소스코드](https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/tree/main/jpa_cascade_orphan_removal)
- [1부로 이동](/2022/03/11/JPA-Entity-cascade=REMOVE,-orphanRemoval=true-왜-헷갈렸을까-1.html)

## 이전 내용

- 부모 - 자식 엔티티 간 연결되어 있으면 관계를 끊지 않고 부모를 삭제할 때 참조 무결성을 지키지 못해 예외가 발생한다. 이를 위해 연결 관계를 제거하고 부모를 제거하거나, 자식 엔티티를 함께 지워야 한다.
- 부모 엔티티와 함께 자식 엔티티를 일일이 지울 필요 없이 cascade=REMOVE, orphanRemoval-true를 통해 한 번에 삭제할 수 있다.
- casacde는 부모의 영속성 관리 행위를 자식에게 전달하고, orphanRemoval은 부모가 자식과의 객체 관계를 끊었을 때 고아가 된 자식을 지운다. 그러므로 cascade=REMOVE, orphanRemoval=true 모두 부모 객체를 지웠을 때 자식 노드가 삭제된다.

[이전 게시글](/2022/03/11/JPA-Entity-cascade=REMOVE,-orphanRemoval=true-왜-헷갈렸을까-1.html)에서 다음과 같은 내용으로 cascade=REMOVE, orphanRemoval=true를 통해 자식 엔티티의 참조 무결성 예외를 피해 자식과 부모 엔티티를 한 번에 삭제하는 방법을 소개했다. 두 옵션 중 무얼 사용하여도 잘 삭제되는 같은 결과를 보이는데, 이번 글에서 둘은 어떤 차이가 있을지 소개한다.

# cascade=REMOVE

```java
// https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/blob/ed9212a4cf42097cd609bb2f66556102cd382414/jpa_cascade_orphan_removal/src/main/java/io/github/zoooohs/entity/cascade/Post.java#L24

@OneToMany(mappedBy = "post", fetch = FetchType.LAZY, cascade = CascadeType.REMOVE)
private List<Comment> comments = new ArrayList<>();
```

cascade는 폭포라는 뜻의 영어단어인데, 폭포처럼 떨어진다는 의미와 이러한 폭포의 이미지를 비유하여 폭포가 떨어지듯 단계적으로 일을 처리한다는 의미가 있다. HTML DOM의 스타일 변경을 위해 사용하는 [CSS](https://ko.wikipedia.org/wiki/CSS)도 Cascade Style Sheets의 약자로 특정 요소에 정의한 스타일 정보가 하위 요소에도 단계적으로 전이되는 특징을 갖는다.

이러한 cascade라는 키워드를 JPA에선 어떤 용도로 사용되는지 알아보기 위해 [hibernate의 API 문서](https://javadoc.io/doc/org.hibernate.javax.persistence/hibernate-jpa-2.0-api/1.0.0.Final/index.html)를 찾아보았다.

> The operations that must be cascaded to the target of the association. Defaults to no operations being cascaded. - hibernate-jpa-2.0-api
> 

연결 대상에 단계적으로 작업이 전달된다고 한다. 그렇기 때문에 cascade=REMOVE는 삭제 작업이 단계적으로 전달된다.

```java
// https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/blob/ed9212a4cf42097cd609bb2f66556102cd382414/jpa_cascade_orphan_removal/src/test/java/io/github/zoooohs/entity/cascade/CascadeTest.java#L38

entityManager.remove(post);
entityManager.flush();

Comment actual = entityManager.find(Comment.class, comment.getId());
Assertions.assertNull(actual);
```

# orphanRemoval=true

```java
// https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/blob/ed9212a4cf42097cd609bb2f66556102cd382414/jpa_cascade_orphan_removal/src/main/java/io/github/zoooohs/entity/orphan/Post.java#L24

@OneToMany(mappedBy = "post", fetch = FetchType.LAZY, orphanRemoval = true)
private Collection<Comment> comments = new ArrayList<>();
```

orphan은 고아라는 뜻인데, JPA에선 부모-자식 관계의 엔티티가 있을 때 부모와의 관계가 끊어진 자식 엔티티를 고아라고 한다. orphanRemoval=true 옵션을 쓰게 되면 이러한 고아가 발생했을 때 해당 엔티티에 대해 계단식 제거를 한다.

> Whether to apply the remove operation to entities that have been removed from the relationship and to cascade the remove operation to those entities. - hibernate-jpa-2.0-api
> 


여기서 관계를 갖는 것과 끊는 것에 대한 의미가 모호할 수 있다. 객체의 필드로 다른 객체를 갖고 있다면, 두 객체 간 연관 관계가 있다고 볼 수 있다. 일대다 관계를 JPA 엔티티로 구현할 때 부모 엔티티 필드 중 OneToMany 어노테이션의 Collection 필드로 여러 자식 엔티티와 연관 관계를 갖고, 자식 엔티티 필드 중 ManyToOne 어노테이션으로 부모 엔티티 필드로 하나의 부모와의 연관 관계를 맺을 수 있다.

- 이렇게 자식 엔티티의 ManyToOne 뿐만 아니라 부모 엔티티의 OneToMany까지 사용하여 양방향으로 연결 관계를 갖는 경우가 있을 수 있다. 실제로 양방향 관계로 작동하는 것은 아니지만 부모 엔티티에서 자식 엔티티를 참조해야 하는 경우 유용하게 사용될 수 있다. 이와 관련해서도 이슈가 있었는데 아래 내용 중 부모 엔티티에서 자식 엔티티 참조가 안되는 경우 [이전 글](/2022/03/04/당황스러웠던-JPA의-양방향-매핑과-영속성-컨텍스트.html)을 참조해보면 도움이 될 수도 있다.

단순히 필드를 선언하는 거로 관계를 갖는 게 아니라, 실제로 필드에 해당 객체 값이 있어야 관계를 갖는데, 다음과 같이 추가가 가능하다.

```java
// https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/blob/ed9212a4cf42097cd609bb2f66556102cd382414/jpa_cascade_orphan_removal/src/test/java/io/github/zoooohs/entity/cascade/CascadeTest.java#L25

// 게시글 작성
Post post = Post.builder().content("오늘은 날씨가 좋다.").build();
entityManager.persist(post);

// 게시글에 댓글 작성
Comment comment = Comment.builder().content("비 오던데요?").build();
comment.setPost(post); // post - comment가 연결됨
entityManager.persist(comment);
```
관계를 끊는 것은 반대로 객체 값을 지워 객체 간 참조 가능한 연결이 없도록 하는 것이다. orphanRemoval을 발동시키기 위해서는 부모 엔티티에서 자식 엔티티와의 관계를 끊어야 하는데, Collection에서 자식 엔티티를 삭제하는 것으로 관계를 지울 수 있다.
```java
// https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/blob/ed9212a4cf42097cd609bb2f66556102cd382414/jpa_cascade_orphan_removal/src/test/java/io/github/zoooohs/entity/orphan/OrphanRemovalTest.java#L38

post.getComments().clear();
entityManager.flush();
entityManager.clear();

Comment actual = entityManager.find(Comment.class, comment.getId());
Assertions.assertNull(actual);
```

끝으로 부모 엔티티를 삭제하는 행위도 자식 엔티티와의 관계를 끊는 행위가 되기 때문에 orphanRemoval을 통해 자식 엔티티를 단계적으로 삭제 할 수 있다. 이는 cascade=REMOVE와 같은 결과를 보인다.

```java
// https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/blob/ed9212a4cf42097cd609bb2f66556102cd382414/jpa_cascade_orphan_removal/src/test/java/io/github/zoooohs/entity/cascade/CascadeTest.java#L37

entityManager.remove(post);
entityManager.flush();

Comment actual = entityManager.find(Comment.class, comment.getId());
Assertions.assertNull(actual);
```

마지막에 언급한 부모 엔티티의 삭제 경우만 알고 있었기 때문에 cascade=REMOVE와 어떤 차이가 있는지 몰라 아무렇게나 사용할 수 있는데, orphanRemoval이 작동하는 다른 경우를 고려하여 알맞은 옵션을 사용해야겠다.

# 이슈?

- orphanRemoval=true와 함께 cascade=PERSIST를 사용하지 않으면 관계 제거에 대한 고아 제거가 발동하지 않는다
 - 이와 관련하여 몇 년 전 [스레드](https://github.com/jyami-kim/Jyami-Java-Lab/issues/1)를 보았지만, 명쾌한 해답은 찾지 못했다
 - cascade=PERSIST를 함께 옵션으로 전달하면 잘 작동한다
 - [다른 자료](https://www.baeldung.com/jpa-cascade-remove-vs-orphanremoval)에서도 orphanRemoval과 PERSIST를 함께 사용한다. 그러나 이는 실제로 PERSIST를 cascade 하기 위해서 사용했지, orphanRemoval과 어떤 연관이 있는지 언급하지 않는다

# 참고자료

- https://javadoc.io/doc/org.hibernate.javax.persistence/hibernate-jpa-2.0-api/1.0.0.Final/index.html
- https://www.baeldung.com/jpa-cascade-remove-vs-orphanremoval
- https://github.com/jyami-kim/Jyami-Java-Lab/issues/1