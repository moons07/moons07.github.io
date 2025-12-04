---
title: 06 JpaPagingItemReader로 DB내용을 읽고, JpaItemWriter로 DB에 쓰기
tags: SpringBatch
---
# 06 JpaPagingItemReader로 DB내용을 읽고, JpaItemWriter로 DB에 쓰기

## **JpaPagingItemReader 개요**

- JpaPagingItemReader는 Spring Batch에서 제공하는 ItemReader로, JPA를 사용하여 DB 데이터를 페이지 단위로 읽음
- JPA 기능 활용 : JPA 엔티티 기반 데이터 처리, 객체 매핑 자동화 등 JPA의 다양한 기능을 활용
- 쿼리 최적화: JPA 쿼리 기능을 사용하여 최적화된 데이터 읽기가 가능
- 커서 제어 : JPA Criteria API를 사용하여 데이터 순회를 제어

## **JpaPagingItemReader 주요 구성 요소**

- EntityManagerFactory : JPA 엔티티 매니저 팩토리를 설정
- JpaQueryProvider : 데이터를 읽을 JPA 쿼리를 제공
- PageSize : 페이지 크기를 설정
- SkippableItemReader : 오류 발생 시 해당 Item을 건너뜀
- ReadListener : 읽기 시작, 종료, 오류 발생 등의 이벤트를 처리
- SaveStateCallback : 잡 중단 시 현재 상태를 저장하여 재시작 시 이어서 처리

## **JpaPagingItemReader 샘플 코드**

### Customer 클래스

```java
@Entity
@Table(name = "customer")
@NoArgsConstructor
@AllArgsConstructor
@Data
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;
    private String name;
    private int age;
    private String gender;
}
```

### **JapPagingItemReader**

#### 방법1  : **JapPagingItemReader 생성자를 이용**

```java
@Bean
public JpaPagingItemReader<Customer> customerJpaPagingItemReader() {
		JpaPagingItemReader<Customer> jpaPagingItemReader = new JpaPagingItemReader<>();

		jpaPagingItemReader.setQueryString(
				"SELECT c FROM Customer c WHERE c.age > :age order by id desc"
		);

		jpaPagingItemReader.setEntityManagerFactory(entityManagerFactory);
		jpaPagingItemReader.setPageSize(CHUNK_SIZE);
		jpaPagingItemReader.setParameterValues(Collections.singletonMap("age", 20));

		return jpaPagingItemReader;
}
```

- setQueryString: JPQL 쿼리를 이용
- setEntityManagerFactory : JPA를 위한 엔터티 매니저를 지정
- setPageSize : 한번에 읽어올 페이지 크기, 청크 크기와 맞춰주는 것이 일반적
- setParameterValues: JPQL쿼리에 전달할 파라미터를 지정
- JpaPagingItemReader는 영속성이 manged 상태이므로 processor에 로직에서 엔티티 변경시 DB에 변경
    - 새로운 객체를 만들어서 반환하거나 disfetch? 상태로 만들어서 비영속성으로 만들어줘야한다.

#### 방법2 : **JpaPagingItemReaderBuilder 이용**

```java
@Bean
public JpaPagingItemReader<Customer> customerJpaPagingItemReader() {
		return new JpaPagingItemReaderBuilder<Customer>()
				.name("customerJpaPagingItemReader")
				.queryString("SELECT c FROM Customer c WHERE c.age > :age order by id desc")
				.pageSize(CHUNK_SIZE)
				.entityManagerFactory(entityManagerFactory)
				.parameterValues(Collections.singletonMap("age", 20))
				.build();
}
```

- 방법1 생성자 방식과 동일

### ItemProcessor

```java
@Slf4j
public class CustomerItemProcessor implements ItemProcessor<Customer, Customer> {
	@Override
	public Customer process(Customer item) throws Exception {
		log.info("Item Processor ------------------- {}", item);
		return item;
	}
}
```

- 단순 item를 로깅

## JpaItemWriter 개요

- JpaItemWriter는 Spring Batch에서 제공하는 ItemWriter 인터페이스를 구현하는 클래스
- 데이터를 JPA를 통해 DB에 저장

## **JpaItemWriter 구성 요소**

- EntityManagerFactory : JPA EntityManager 생성을 위한 팩토리 객체
- JpaQueryProvider : 저장할 엔터티를 위한 JPA 쿼리를 생성하는 역할

### 장점

- ORM 연동 : JPA를 통해 다양한 DB에 데이터를 저장
- 객체 매핑 : 엔터티 객체를 직접 저장하여 코드 간결성을 높임
- 유연성 : 다양한 설정을 통해 원하는 방식으로 데이터를 저장

### 단점

- 설정 복잡성 : JPA 설정 및 쿼리 작성이 복잡해질 수 있음
- 오류 가능성 : 설정 오류 시 데이터 손상 가능성이 있음

## **JpaItemWriter 샘플 코드**

### JPA 설정

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa' // JPA 실습을 위해 추가
    implementation 'org.springframework.boot:spring-boot-starter-batch' // spring batch 관련 의존성
    //implementation 'com.h2database:h2:2.2.224' // H2 database 의존성 추가
    implementation 'mysql:mysql-connector-java:8.0.33'
    implementation 'org.projectlombok:lombok:1.18.34' // log 출력을 위해 lombok 추가
    annotationProcessor 'org.projectlombok:lombok:1.18.34'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.batch:spring-batch-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

```
```java
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://db-m6kkf.pub-cdb.ntruss.com:3306/coloring_book?characterEncoding=UTF-8
spring.datasource.hikari.username=test
spring.datasource.hikari.password=test
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.connection-test-query=SELECT 1
spring.batch.jdbc.initialize-schema=always

spring.batch.job.enabled=false
logging.level.org.springframework.jdbc.core.JdbcTemplate=DEBUG

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect

logging.level.org.springframework.batch=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type=TRACE
logging.level.org.springframework.orm.jpa=DEBUG
```

### 엔티티

```java
@Entity
@Table(name = "customer")
@NoArgsConstructor
@AllArgsConstructor
@Data
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String name;
    private int age;
    private String gender;
}
```

### **JpaItemWriter**

```java
@Bean
public JpaItemWriter<Customer> jpaItemWriter() {
    return new JpaItemWriterBuilder<Customer>()
            .entityManagerFactory(entityManagerFactory)
            .usePersist(true)
            .build();
}
```

`usePersist(true)`는 `JpaItemWriter`에서 사용하는 설정으로, `persist()` 메서드를 사용하여 엔티티를 저장하도록 지정하는 옵션입니다. 기본적으로 `JpaItemWriter`는 `EntityManager`의 `merge()` 메서드를 사용하여 엔티티를 저장하는데, `usePersist(true)`를 설정하면 `merge()` 대신 `persist()`를 사용합니다.

#### `persist()` vs `merge()`

- **`persist()`**: 새로 생성된 엔티티를 저장하는 데 사용됩니다. 만약 엔티티가 이미 존재하는 경우, `persist()`는 예외를 발생시킵니다. 즉, 데이터베이스에 이미 존재하는 엔티티를 업데이트하는 기능은 없습니다. `persist()`는 주로 새로운 엔티티를 삽입할 때 사용합니다.
- **`merge()`**: 주어진 엔티티가 영속성 컨텍스트에 이미 존재하면 해당 엔티티를 갱신하고, 존재하지 않으면 새로 삽입하는 방식입니다. 따라서 `merge()`는 엔티티의 업데이트 및 삽입을 모두 처리할 수 있습니다.
