---
title: "[JPA] 실무 트러블슈팅 2건 정리 (OneToOne/FK/Lazy, fetch join + pagination)"
excerpt_separator: "<!--more-->"
categories:
  - development

tags:
  - jpa
  - querydsl
  - oneToOne
  - fetchJoin
  - pagination
---

저도 아직 JPA 에 적응 중이지만, 지금 회사에서 실무를 처음 겪으면서 기존 코드에서 JPA 잘못된 사용으로 발생한 트러블슈팅 2건 관련해서 정리해둡니다.

둘 다 JPA 를 사용하면서 실수할 수 있는 기초적인 케이스라, 처음 실무를 하는 분들이 유사한 실수를 하지 않게끔 기록하려고 합니다.

<!--more-->

### 1. OneToOne 단방향 + FK 위치 때문에 LAZY가 기대대로 동작하지 않은 케이스

저희 서비스에서 사용하는 여러 DB 중 메인 DB 의 CPU 가 서비스 성장과 함께 날로 증가하고 있었던 중 CPU 를 줄일 수 있는 방법을 찾고 있었습니다. 

그러던 중 호출 빈도가 가장 높은 쿼리 중 서비스에 크게 사용되지 않는 테이블을 조회하는 쿼리가 있는 것을 발견했고, 해당 쿼리가 왜 많이 호출되는지 분석을 시작했습니다.

초기 코드는 아래와 같았습니다.

```java
// Content.java
// ... 기존코드
@OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
@JoinColumn(name = "contentId")
private ContentTree contentTree;
// ... 기존코드
```

```java
// ContentTree.java
@Table(name = "`content_tree`")
@Entity
@Getter
@Setter
public class ContentTree {
  @Id
  private Long contentId;
  private Long parentContentId;
  private Long rootContentId;
}
```

간략히 설명하면 Content 는 서로 자식/부모 관계로 이뤄질 수 있고, ContentTree 테이블을 통해서 부모 content 와, root content 의 ID 를 관리를 하는 구조였습니다.

Content 테이블이 업무에 사용되는 메인 테이블이기에 조회할 때 마다 ContentTree 테이블을 조회하지 않기 위해 `fetch = FetchType.LAZY` 설정을 해두었지만, 실제로는 Content 테이블 조회 시 마다 조회가 되는 문제가 발생하고 있었습니다.

원인은 매핑 의도와 실제 DB 구조가 어긋나 있었다는 점입니다.

- 코드상으로는 `Content`가 FK를 가진 것처럼 보이지만
- 실제 테이블은 `ContentTree.contentId`가 PK/FK 역할을 수행
- 즉, 주인(`Content`)/외래키 위치(`ContentTree`)가 현재 단방향 매핑 의도와 맞지 않음

저도 이번에 알았지만 아래 참고 자료에 잘 설명된 것과 같이 JPA 의 OneToOne 관계에서 FK가 주인 테이블(`Content`)이 아닌 대상 테이블(`ContentTree`)에 존재하는 경우 기능적 한계로 LAZY 로딩 불가한 것이었습니다.

아마 이런 지식을 잘 모르는 상태에서 막연하게 LAZY 로딩을 기대하며 코드를 짜고 그 이후 모니터링을 별도로 하지 않았던 게 아닐까 싶네요.

> 참고 자료 : [[JPA] @OneToOne, 일대일[1:1] 관계](https://ict-nroo.tistory.com/126)

#### 대응

대응을 위해 몇 가지 안을 찾아 아래와 같이 정리했습니다.

1. `content -> contentTree` 직접 연관은 끊고 필요 시 별도 조회
2. `contentTree`에 `contentId` 외 별도 `id` PK 추가 -> `content` 테이블에 `contentTreeId` FK 컬럼 추가 
3. 실제 데이터는 1:1 이지만 코드 영향 최소화를 위해 일부 접근부는 1:N 형태를 수용하는 방식으로 완충

위에 설명드린 대로 해당 테이블은 서비스의 메인 테이블이라, 1,2번 작업을 하기엔 데이터 양도 많고, 연관된 코드가 많아 수정 범위가 넓어 side effect 이 우려되었습니다.

그래서 3번 방향으로 수정하는 것으로 가닥을 잡고 아래와 같이 수정하였습니다.

```java

@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
@JoinColumn(name = "contentId")
@Builder.Default
private Set<ContentTree> contentTree = Collections.emptySet();  // TODO: Set 으로 한 이유 찾아 추가 기입

//-- 기존 코드

public ContentTree getContentTree() {
  return this.contentTree.stream()
      .findFirst()
      .orElse(null);
}
```

위와 같이 수정 후 기존 대비 45% 가량 호출 건수가 줄어드는 것을 확인하였습니다.

### 2. fetch join + pagination(limit) Warn 로그 발생

두 번째는 가끔 이슈 확인을 위해 로그를 확인할 때 아래 로그가 너무 빈번하게 발생하여 필요한 로그 확인이 어려운 이슈였습니다.

```text
HHH000104: firstResult/maxResults specified with collection fetch; applying in memory
```

저희는 Datadog 로 모든 로그를 확인하고 있고, 로그가 많아지면 비용이 증가하기 때문에 해당 로그가 왜 남는지 파악해 보기로 했습니다.

원인은 명확했습니다. 이슈 코드는 아래와 같았습니다.

```kotlin
// Content.kt
// ... 기존코드
@OneToMany(fetch = FetchType.LAZY)
@JoinColumn(name = "contentId")
val comments: List<Comment> = emptyList(),
// ... 기존코드
```
```kotlin
@Entity
@Table(name = "comment")
class Comment(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    val contentId: Long,
// ... 기존코드
```
```kotlin
class ContentRepositoryImpl : QuerydslRepositorySupport(Content::class.java), ContentRepositoryCustom {

    override fun findByContentId(contentId: Long): Content? {
        val qContent = QContent.Content
        val qComment = QComment.Comment

        return from(qContent)
            .leftJoin(qContent.comments, qComment).fetchJoin()
            .where(
                qContent.contentId.eq(contentId)
            )
            .distinct()
            .fetchFirst()
    }
// ... 기존코드
```

해당 코드는 리팩토링 중 N+1 우려되어 Query DSL 변환해서 수정됐는데, 단건의 `Content` 조회에 과한 `fetchJoin`과 `fetchFirst` 이 들어간 상태였습니다.

- `oneToMany` 컬렉션(`comment`)을 `fetchJoin`
- 동시에 `fetchFirst`(limit/pagination 성격) 사용
- Hibernate가 DB 레벨 페이징 대신 메모리에서 적재 후 페이징 작업을 애플리케이션에서 후처리

하나의 Content 에 N건의 Comment 가 있는 경우 메모리에 적재하는 데이터가 많아지기 때문에 이런 Warn 로그를 자동으로 남겨주는 것이었습니다.

#### 대응

`fetchJoin + fetchFirst` 조합 쿼리를 점검해서, 실제 N+1 위험이 낮거나 불필요한 경우 `Comment` 조인을 제거했습니다.

정리하면 아래 기준으로 판단했습니다.

1. 단건 조회에서 컬렉션을 반드시 즉시 로딩해야 하는가
2. 해당 시점에 컬렉션 필드 접근이 실제로 발생하는가
3. 필요 시점에 지연 조회 또는 별도 조회로 분리 가능한가

불필요한 컬렉션 fetch join을 걷어내니 Warn가 사라지고, 쿼리 의도도 훨씬 명확해졌습니다.

### 마무리

이번 두 이슈는 공통적으로 막연하게 기술을 사용하는 게 아니라, 본인의 의도대로 해당 코드가 지원을 하는지, 본인이 작성한 코드가 어떤 결과를 야기할 수 있는지 정확히 알고 사용해야 한다는 점을 다시 확인한 계기였습니다.

- 연관관계는 DB FK 소유 구조와 함께 봐야 하고
- fetch join은 N+1 해결 도구이지 기본 선택지가 아님

다음에는 매핑 단계에서 FK 소유를 먼저 고정하고, 조회 쿼리는 실제 접근 패턴 기준으로 최소 조인부터 시작하려고 합니다.

그럼 이만. 🥕👋🏼🖐🏼

### 참고자료
[[JPA] @OneToOne, 일대일[1:1] 관계](https://ict-nroo.tistory.com/126)
[fetch join 과 pagination 을 같이 쓸 때 [HHH000104: firstResult/maxResults specified with collection fetch; applying in memory]](https://javabom.tistory.com/104)
