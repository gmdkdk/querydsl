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
빌드, 어노테이션쪽 main을  Add Content Root로 잡아주자<be>
2. settings > Build, Execution, Deployment에서 Build and run using을 gradle로 세팅해주자.
<br>


테스트를 위해 Member(N) : Team(1) 의 구조를 잡아주자.
