---
layout: post
title:  "JPA Entity cascade=REMOVE, orphanRemoval=true 왜 헷갈렸을까 - 1"
excerpt: "부모 Entity와 연결된 자식 Entity 관계를 건들지 않고 삭제하는 방법엔 대표적으로 두가지 있는데"
date: "2022-03-11"


author: zoooo-hs
tags: [JPA, Java, Spring Data JPA]

---

* TOC
{:toc}

# TL;DR

- casacde는 부모의 영속성 관리 행위를 자식에게 전달하고, orphanRemoval은 부모가 자식과의 객체 관계를 끊었을 때 고아가 된 자식을 지운다.
- cascade=REMOVE, orphanRemoval=true 모두 부모 객체를 지웠을 때 자식 노드가 삭제된다.

# 배경

게시글 - 댓글과 같은 관계를 엔티티를 이용해 연결할 때 다음과 같이 연결하곤 한다.

```java
// 게시글 엔티티
@Entity
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @Column
    private String content;

    @OneToMany(mappedBy = "post")
    private List<Comment> comments;

    @Builder
    public Post(String content) {
        this.content = content;
    }
}

// 댓글 엔티티Post post = Post.builder().content("오늘은 날씨가 좋다.").build();
@Entity
public class Comment {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @Column
    private String content;

    @ManyToOne
    @JoinColumn(nullable = false, name = "post_id")
    private Post post;

    @Builder
    public Comment(String content, Post post) {
        this.content = content;
        this.post = post;
    }
}
```

댓글은 ManyToOne으로 게시글 엔티티를 참조하여 FK를 갖게 된다. 게시글의 경우 게시글에 달린 댓글 리스트를 확인하기 위해  OneToMany로 댓글 엔티티 Collection을 참조한다. 

이후 다음과 같이 특정 게시글에 댓글이 몇 개 작성된 후에

```java
Post post = Post.builder().content("오늘은 날씨가 좋다.").build();
entityManager.persist(post);

// 게시글에 댓글 작성
Comment comment = Comment.builder().post(post).content("비 오던데요?").build();
entityManager.persist(comment);
```

게시글 엔티티를 삭제하면 다음과 같은 예외가 발생한다.

```java
entityManager.remove(post);
entityManager.flush();

/* 다음과 같은 에러 로그
Caused by: java.sql.SQLIntegrityConstraintViolationException: (conn=363) Cannot delete or update a parent row: a foreign key constraint fails (`spring-playground-demo`.`comment`, CONSTRAINT `FKs1slvnkuemjsq2kj4h3vhx7i1` FOREIGN KEY (`post_id`) REFERENCES `post` (`id`))
*/

// 아래는 테스트 코드 전체
@DisplayName("Comment의 참조 무결성으로 인해 삭제시 예외 발생")
@Test
void ReferenceIntegrityFailure() {
    // 게시글 작성
    Post post = Post.builder().content("오늘은 날씨가 좋다.").build();
    entityManager.persist(post);

    // 게시글에 댓글 작성
    Comment comment = Comment.builder().post(post).content("비 오던데요?").build();
    entityManager.persist(comment);

    try {
        entityManager.remove(post);
        entityManager.flush();
    } catch (Throwable e) {
        while (e.getCause() != null) {
            if (e.getClass() == SQLIntegrityConstraintViolationException.class) {
                Assertions.assertTrue(true);
                return;
            }
            e = e.getCause();
        }
    }
    Assertions.fail();
}
```

이는 댓글 엔티티에 fk로 연결된 게시글이 삭제되어 [참조 무결성](https://ko.m.wikipedia.org/wiki/참조_무결성)을 지킬 수 없기 때문이다. 이런 예외를 피해 게시글을 삭제하기 위해서는 게시글 pk를 fk로 갖는 댓글들을 삭제하거나 fk 연결을 끊어줘야 한다. 본 예제에서는 ManyToOne으로 연결된 게시글 엔티티 필드가 nullable=false 속성을 갖기 때문에 연결을 끊는 것으론 제거할 수 없다. 그렇다면 다음과 같은 코드를 앞에 추가하여 댓글 삭제가 가능하다.

```java
@DisplayName("참조 무결성을 지키기 위해 자식 엔티티를 지우고 부모 엔티티를 삭제")
@Test
void deleteChildFirst() {
    // 게시글 작성
    Post post = Post.builder().content("오늘은 날씨가 좋다.").build();
    entityManager.persist(post);

    // 게시글에 댓글 작성
    Comment comment = Comment.builder().post(post).content("비 오던데요?").build();
    entityManager.persist(comment);

    entityManager.remove(comment);
    entityManager.remove(post);
    entityManager.flush();

    Assertions.assertTrue(true);
}
```

참조 무결성을 지켜 게시글을 삭제할 수 있게 되었다. 그러나 OneToMany 관계를 갖는 엔티티가 여러 종류일 경우 매번 사전에 자식 엔티티(1:n 엔티티 관계에서 주로 1쪽이 부모, n쪽이 자식)를 제거해야 하는 코드를 작성해야 한다. 중복도 많고 실수로 삭제 코드를 작성하지 못하면 런타임에 오류가 날것이다.

분명 무결성을 지키기 위해 꼭 해줘야 할 작업이지만, 매우 귀찮았다. entityManager를 그대로 사용하는 코드도 중복이 많다며 Spring Data JPA를 만든 게 Guru 들인데 이런 귀찮은 일을 할 리 없다고 생각했다. 그래서 바로 구글링해보았다.

```
Google : JPA Remove Child Entity Automaticallly
--> 
https://stackoverflow.com/questions/19607954/how-to-automatically-remove-child-entities
https://stackoverflow.com/questions/23925322/delete-child-from-parent-and-parent-from-child-automatically-with-jpa-annotation
```

공통으로 OneToMany 어노테이션에 casecade=REMOVE를 추가하라고 하며, 추가로 orpahRemoval=true를 제안해주었다. 그래서 아래와 같이 cascade를 적용해 보았다.

```java
// cascade
@OneToMany(mappedBy = "post", cascade = CascadeType.REMOVE)
private List<Comment> comments;

// orphanRemoval
@OneToMany(mappedBy = "post", orphanRemoval = true)
private List<Comment> comments;

// 테스트 코드
@DisplayName("cascade = REMOVE로 자식 노드도 함께 삭제")
@Test
void cascadeRemove() {
    // 게시글 작성
    Post post = Post.builder().content("오늘은 날씨가 좋다.").build();
    entityManager.persist(post);

    // 게시글에 댓글 작성
    Comment comment = Comment.builder().post(post).content("비 오던데요?").build();
    entityManager.persist(comment);
    Long commentId = comment.getId();

    entityManager.remove(post);
    entityManager.flush();

    Comment actual = entityManager.find(Comment.class, commentId);
    Assertions.assertNull(actual);
}

/** 또 오류난다!!!!
* Caused by: java.sql.SQLIntegrityConstraintViolationException: (conn=493) Cannot delete or update a parent row: a foreign key constraint fails (`spring-playground-demo`.`comment`, CONSTRAINT `FKs1slvnkuemjsq2kj4h3vhx7i1` FOREIGN KEY (`post_id`) REFERENCES `post` (`id`))
*/
```

안된다. 안될 수밖에. **cascade, orphanRemoval은 자식 엔티티에 행위를 전파하거나, 자식 엔티티와 연결이 끊어질 때 작동**한다. 그런데 위 코드에서는 **부모 엔티티와 자식 엔티티가 연결되지 않았다**. 자식 엔티티에서 부모를 참조하고 있을 뿐, 부모 엔티티에서 자식 엔티티를 연결한 부분이 없다. 이러한 JPA의 양방향 매핑과 관련해서는 [이전 글](/2022/03/04/%EB%8B%B9%ED%99%A9%EC%8A%A4%EB%9F%AC%EC%9B%A0%EB%8D%98-JPA%EC%9D%98-%EC%96%91%EB%B0%A9%ED%96%A5-%EB%A7%A4%ED%95%91%EA%B3%BC-%EC%98%81%EC%86%8D%EC%84%B1-%EC%BB%A8%ED%85%8D%EC%8A%A4%ED%8A%B8.html)을 참고해보면 좋다. 이번엔 바로 코드에 적용해보았다. 다음과 같이 댓글 엔티티를 수정한다.

```java
// Post.java
// 혹은 @OneToMany(mappedBy = "post", orphanRemoval = true)
@OneToMany(mappedBy = "post", cascade = CascadeType.REMOVE)
private List<Comment> comments = new ArrayList<>();

// Comment.java
public void setPost(Post post) {
        if (this.post != null) {
            this.post.getComments().remove(this);
        }
        this.post = post;
        if (post == null) return;
        this.post.getComments().add(this);
}

// 테스트 코드
@DisplayName("cascade = REMOVE로 자식 노드도 함께 삭제")
@Test
void cascadeRemove() {
    // 게시글 작성
    Post post = Post.builder().content("오늘은 날씨가 좋다.").build();
    entityManager.persist(post);

    // 게시글에 댓글 작성
    Comment comment = Comment.builder().content("비 오던데요?").build();
    // setPost를 호출해서 post.comments에 comment 추가되게 
		comment.setPost(post);
    entityManager.persist(comment);
    Long commentId = comment.getId();

    entityManager.remove(post);
    entityManager.flush();

    Comment actual = entityManager.find(Comment.class, commentId);
    Assertions.assertNull(actual);
}
```

위와 같이 코드를 작성하면 삭제가 잘 된다.

글을 작성하다 보니 배경 설명이 길어졌다. 여기서 한 번 끊고 다음 글에 이어서 작성한다. 다음 글에서는 cascade, orphanRemoval이 같은 결과를 낸다면 왜 둘 다 존재하는지, 둘이 실제로 어떤 차이를 갖는지 설명한다.