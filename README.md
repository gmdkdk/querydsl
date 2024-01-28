# querydsl
## [2024.01.22]<br>
스프링부트 3.2.1버전 기준으로 설정을 해보자.<br>
다음 의존성을 추가해주자.
```groovy
	implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
	annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jakarta"
	annotationProcessor "jakarta.annotation:jakarta.annotation-api"
	annotationProcessor "jakarta.persistence:jakarta.persistence-api"
```
gradle > Tasks > other > compileJava를 하면 Entity에 매칭되는 Q파일이 생성되는데 컴파일을 하지 못하는 문제가 있다. <br>
1. file > Project Structure > Modules에서 
빌드, 어노테이션쪽 main을  Add Content Root로 설정
2. settings > Build, Execution, Deployment에서 Build and run using을 gradle로 세팅


테스트를 위해 Member(N) : Team(1) 의 구조를 잡아주자.

---
## [2024.01.23]
jqpl과 querydsl 비교  
* jpql
```java
 @Test
public void startJPQL() {
    String qlString = "select m " +
            "from Member m " +
            "where m.username = :username";

    Member findMember = em.createQuery(qlString, Member.class)
            .setParameter("username", "member1")
            .getSingleResult();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}
```

* querydsl
```java
@Test
public void startQuerydsl() {
//        QMember m = new QMember("m");
//        QMember m = QMember.member;

    Member findMember = queryFactory
            .select(member)
            .from(member)
            .where(member.username.eq("member1"))
            .fetchOne();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}
```
jpql은 컴파일 시점에 오류를 잡을 수 없지만 querydsl은 컴파일 시점에 오류를 알 수 있음 !  
select와 from에 들어가는 엔티티가 같다면 selectFrom()을 사용할 수 있다.  
> fetch() : 리스트 조회, 없으면 빈 리스트  
fetchOne(): 단건 조회, 없으면 null, 2개 이상이면 NonUniqueResultException  
fetchFirst(): 처음 한 건 조회 limit(1).fetchOne();  
fetchResults(): 페이징 정보 포함, count쿼리 추가 실행(쿼리 두번 나감)  
fetchCount(): count쿼리로 변경해서 count만 조회  

---
##[2024.01.24]  
기본적으로 sql문법과 너무 유사해서 한번씩만 써보면 익숙하게 사용 할 수 있을거 같다.

```java
@Test
public void aggregation() {
    List<Tuple> result = queryFactory
            .select(
                    member.count(),
                    member.age.sum(),
                    member.age.avg(),
                    member.age.max(),
                    member.age.min()
            )
            .from(member)
            .fetch();
    Tuple tuple = result.get(0);
    assertThat(tuple.get(member.count())).isEqualTo(4);
    assertThat(tuple.get(member.age.sum())).isEqualTo(100);
    assertThat(tuple.get(member.age.avg())).isEqualTo(25);
    assertThat(tuple.get(member.age.max())).isEqualTo(40);
    assertThat(tuple.get(member.age.min())).isEqualTo(10);
}
```  

집계함수도 기존 sql을 알고 있다면 쉽게 사용할 수 있다.  
반환 타입이 여러가지일 경우 Tuple을 사용 할 수 있다..  
dto를 하나 생성 해 사용하는 것이 좋지 않을까 ?  

---
##[2024-01-27]
페치 조인은 간단하다.
```java
Member findMember = queryFactory
        .selectFrom(member)
        .join(member.team, team).fetchJoin()
        .where(member.username.eq("member1"))
        .fetchOne();
```
join 뒤에 .fetchJoin()을 사용해주면 된다.  
  
querydsl을 사용하면서 가장 궁금했던 것은 서브쿼리였다.
`com.querydsl.jpa.JPAExpressions `를 사용하여 쓸 수 있다.  

```java
@Test
public void subQuery( ) {
    QMember memberSub = new QMember("memberSub");

    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.eq(
                    JPAExpressions
                            .select(memberSub.age.max())
                            .from(memberSub)
            ))
            .fetch();

    assertThat(result).extracting("age").containsExactly(40);
}
```
기존에 사용중인 엔티티와 alias가 중복되지 않게 다시 QMember를 생성해 주었다.  
static import를 사용하여 더 깔끔하게 사용할 수 있다.
  
```java
List<Tuple> result = queryFactory
                .select(member.username,
                        select(memberSub.age.avg())
                                .from(memberSub)
                )
                .from(member)
                .fetch();
```
  
querydsl에서 projection은 다음 세가지 방법을 사용한다.
1. 프로퍼티 접근
2. 필드 접근
3. 생성자 접근

```java
Projections
    .bean() // 세터 메서드를 통해 사용
    .field() // 필드에 접근
    .constructor() //생성자에 접근
```

```java
List<UserDto> result = queryFactory
            .select(Projections.fields(UserDto.class,
                    member.username.as("name"),
                    member.age))
            .from(member)
            .fetch();
```
  
## [2024-01-28]
