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
