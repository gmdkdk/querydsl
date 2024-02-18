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
@QueryProjection  
dto의 생성자에 해당 애노테이션을 써주고 gradle > compileJava를 하면 dto의 Q파일이 생성된다.  
```java
List<MemberDto> result = queryFactory
        .select(new QMemberDto(member.username, member.age))
        .from(member)
        .fetch();
```
사용법은 정말 간단하다.  
Projection.construct()는 실행해봐야 오류를 잡을 수 있고, @QueryProjection은 실행 전에 잡을 수 있다.  
@QueryProjection의 단점으로는 Q파일을 생성해야 하고 dto가 querydsl에 의존해야 한다.  
  
동적 쿼리는 BooleanBuilder와 다중 파라미터 사용으로 해결할 수 있다.  
* BooleanBuilder 사용
```java
BooleanBuilder builder = new BooleanBuilder();
        if(usernameCond != null) {
            builder.and(member.username.eq(usernameCond));
        }
        
        if(ageCond != null) {
            builder.and(member.age.eq(ageCond));
        }
        
        return queryFactory
                .selectFrom(member)
                .where(builder)
                .fetch();
    }
```
<br>
  * 다중 파라미터 사용

```java
    public void dynamicQuery_WhereParam() {
        String usernameParam = /*null*/"member1";
        Integer ageParam = /*null*/10;

        List<Member> result = searchMember2(usernameParam, ageParam);
        assertThat(result.size()).isEqualTo(1);
    }

    private List<Member> searchMember2(String usernameCond, Integer ageCond) {
        return queryFactory
                .selectFrom(member)
                .where(usernameEq(usernameCond), ageEq(ageCond))
                .fetch();
    }

    private Predicate usernameEq(String usernameCond) {
        if(usernameCond == null) return null;
        return member.username.eq(usernameCond);
    }

    private Predicate ageEq(Integer ageCond) {
        if(ageCond == null) return null;
        return member.age.eq(ageCond);
    }
```
<br>
반환 타입을 BooleanExpression으로 하고 

```java
    private BooleanExpression allEq(String usernameCond, Integer ageCond) {
        return usernameEq(usernameCond).and(ageEq(ageCond));
    }
```
이런식으로 조합해서 사용 할 수도 있다.  
yml파일을 쪼개 `spring.profiles.active` 속성을 test와 local로 설정하였다.  
  
##[2024-02-01]  
* 사용자 정의 리포지토리  
1. 사용자 정의 인터페이스 작성
2. 사용자 정의 인터페이스 구현
3. 스프링 데이터 리포지토리에 사용자 정의 인터페이스 상속

```java
public interface MemberRepositoryCustom {
    List<MemberTeamDto> search(MemberSearchCondition condition);
}
```
```java
public class MemberRepositoryImpl implements MemberRepositoryCustom {

    private final JPAQueryFactory queryFactory;

    // 스프링 bean으로 등록되어 있다면 이렇게 써도 됨
//    public MemberRepositoryImpl(JPAQueryFactory jpaQueryFactory) {
//        this.jpaQueryFactory = jpaQueryFactory;
//    }

    public MemberRepositoryImpl(EntityManager em) {
        this.queryFactory = new JPAQueryFactory(em);
    }

    @Override
    public List<MemberTeamDto> search(MemberSearchCondition condition) {

        return queryFactory
                .select(new QMemberTeamDto(
                        member.id.as("memberId"),
                        member.username,
                        member.age,
                        team.id.as("teamId"),
                        team.name.as("teamName")
                ))
                .from(member)
                .leftJoin(member.team, team)
                .where(
                        usernameEq(condition.getUsername()),
                        teamNameEq(condition.getTeamName()),
                        ageGoe(condition.getAgeGoe()),
                        ageLoe(condition.getAgeLoe())
                )
                .fetch();
    }

    private BooleanExpression usernameEq(String username) {
        return hasText(username) ? member.username.eq(username) : null;
    }

    private BooleanExpression teamNameEq(String teamName) {
        return hasText(teamName) ? team.name.eq(teamName) : null;
    }

    private BooleanExpression ageGoe(Integer ageGoe) {
        return ageGoe != null ? member.age.goe(ageGoe) : null;
    }

    private BooleanExpression ageLoe(Integer ageLoe) {
        return ageLoe != null ? member.age.loe(ageLoe) : null;
    }
}
```
```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {
    List<Member> findByUsername(String username);
}
```