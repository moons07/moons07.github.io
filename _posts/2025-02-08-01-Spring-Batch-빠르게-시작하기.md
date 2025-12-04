# **Spring Batch 초기화 및 스프링배치 필요정보**

## 기본 프로젝트 구성하기

- IntelliJ에서 Spring Boot를 사용한 Spring Batch 프로젝트 Setting시 다음과 같은 방법을 구성 가능
  - File > New > Project… > Generators > Spring Boot

  ![new_project.png](01%20Spring%20Batch%20%EB%B9%A0%EB%A5%B4%EA%B2%8C%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/new_project.png)


> **Spring Boot란?**
>
>
> Java Spring Boot(Spring Boot)는 세 가지 핵심 기능을 통해 Spring Framework를 사용하여 더 빠르고 쉽게 웹 애플리케이션과 마이크로서비스를 개발하도록 돕는 툴
>
> 1. 자동 구성
> 2. 구성에 대한 독선적 접근 방식
> 3. 독립형 애플리케이션을 만드는 능력
>
> 참고 : [https://www.ibm.com/kr-ko/topics/java-spring-boot](https://www.ibm.com/kr-ko/topics/java-spring-boot)
>

- 위 프로젝트 구성시 application, package name 등을 작성하고 Next 버튼을 클릭하면 프로젝트 구성시 의존성을 선택 가능
  - Spring Batch 프로젝트를 생성하는 것이므로 Spring Batch 의존성을 선택 후 Create

![dependency.png](01%20Spring%20Batch%20%EB%B9%A0%EB%A5%B4%EA%B2%8C%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/dependency.png)

## 배치를 위한 기본 설정

- Spring Boot를 통해 프로젝트 구성을 완료하면 아래와 같이 **build.gradle 파일이 생성 (Project Type을 maven으로 지정했을 경우 pom.xml)**
  - `implementation 'org.springframework.boot:spring-boot-starter-batch'`을 통해 Spring Batch에 대한 의존성이 추가된 것을 알수 있음

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
}

group = 'org.study'
version = '0.0.1-SNAPSHOT'

java {  // 빌드와 관련된 java 설정 정의
    toolchain { // 프로젝트에서 특정 Java 버전을 사용하도록 명시
            //java 버전 17로 빌드 및 컴파일
        languageVersion = JavaLanguageVersion.of(17)
    }
}

repositories {
    mavenCentral()
}

dependencies {
        // spring batch 관련 의존성
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.batch:spring-batch-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform() // JUnit5 플랫폼을 사용하도록 설정
}
```

## 배치 기동시키기

- Spring Batch로 실행시키기 위해선 `@EnableBatchProcessing`  애노테이션을 Spring Boot Application 클래스에 붙여 실행

```groovy
package org.study.spring_batch_study;

import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@EnableBatchProcessing  // spring batch 모드로 동작하게 해주는 애노테이션
@SpringBootApplication
public class SpringBatchStudyApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBatchStudyApplication.class, args);
    }

}
```

> `@EnableBatchProcessing` 애노테이션

Spring Batch에서 배치 처리를 위한 설정을 간편하게 활성화하기 위해 사용하는 애노테이션, Spring Boot 애플리케이션에 이 애노테이션을 추가하면, 배치 관련 여러 기능을 자동으로 설정하고 초기화

**주요 역할**
1. Spring Batch의 기본 구성 활성화
- 배치 작업을 정의하고 실행하기 위한 다양한 빈을 자동으로 생성 (`JobRepository`, `JobLauncher`, `JobRegistry`, `JobBuilderFactory`, `StepBuilderFactory` 등)
- 이러한 기본 구성 요소들은 배치 작업(Job) 및 스텝(Step)을 정의하고 관리하는 데 필요한 필수 요소

2. 기본 BatchConfigurer 제공
- `BatchConfigurer` 인터페이스의 기본 구현체를 제공하여, 개발자가 명시적으로 구성을 하지 않아도 배치 작업이 동작
- 필요에 따라 커스터마이징이 가능하며, 데이터베이스를 사용한 작업 이력 저장소 설정이나 `JobRepository`의 트랜잭션 관리 등을 변경

3. 배치 작업(Job) 정의의 간소화
- `@EnableBatchProcessing`을 사용하면, 간단한 Java 기반 구성만으로 배치 작업을 정의하고 실행
- 예를 들어, `@Configuration` 클래스에서 `JobBuilderFactory`와 `StepBuilderFactory`를 이용해 `Job`과 `Step`을 정의

참고 : Chat GPT
>

## 실행해보기

- Application을 실행시키면 아래와 같은 오류 발생
- Spring Batch를 실행시키기 위해선 실행한 Job, Step 등의 정보를 데이터베이스에 저장하며, 이를 위해 DataSource 설정이 안되어 있으므로 오류가 발생

![error.png](01%20Spring%20Batch%20%EB%B9%A0%EB%A5%B4%EA%B2%8C%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/error.png)

## **DataSource 구성하기**

- Spring Batch를 간단한 테스트 용도로 사용할 것이기 때문에 메모리 기반의 DB를 사용 (H2 Database)
- 아래와 같이 **build.gradle에 의존성을 추가**

```groovy
// H2 database 의존성 추가
implementation 'com.h2database:h2:2.2.224'
```

- **application.properties(또는 application.yml)에 DataSource 정보를 작성**

```java
# H2 DataBase
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=test
spring.datasource.password=test
```

```groovy
# H2 DataBase용
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: test
    password: test
```

# Spring Batch 스키마 구조

- Spring Batch를 수행하면 아래와 같이 Spring Batch를 위한 테이블이 자동으로 생성

![[https://docs.spring.io/spring-batch/reference/schema-appendix.html](https://docs.spring.io/spring-batch/reference/schema-appendix.html)](01%20Spring%20Batch%20%EB%B9%A0%EB%A5%B4%EA%B2%8C%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/spring_batch_schema.png)

[https://docs.spring.io/spring-batch/reference/schema-appendix.html](https://docs.spring.io/spring-batch/reference/schema-appendix.html)

## **BATCH_JOB_INSTANCE 테이블**

- 스키마중 가장 기본이 되는 테이블
- 배치가 수행되면 Job이 생성이 되고, 해당 잡 인스턴스에 대해서 관련된 모든 정보를 가진 최상위 테이블

| 컬럼명 | 타입 | Nullable | 설명 |
| --- | --- | --- | --- |
| JOB_INSTANCE_ID | `BIGINT` | NOT NULL | 인스턴스에 대한 유니크 아이디.<br>JobInstance 객체의 getId로 획득이 가능 |
| VERSION | `BIGINT` | NULL | 버전정보.<br>특정 잡 인스턴스에 대한 업데이트 발생시 증가하며 동일 버전의 잡 인스터스가 동시에 수행하는 것을 방지 |
| JOB_NAME | `VARCHAR(100)` | NOT NULL | 배치 잡 객체로 획득한 잡 이름, 인스턴스를 식별하기 위해 필요 |
| JOB_KEY | `VARCHAR(32)` | NOT NULL | JobParameter를 직렬화한 데이터 값이자 동일한 잡을 다른 잡과 구분하는 값.<br>잡은 이 JobParameter가 동일할 수 없으며, JOB_KEY는 구별될수 있도록 달라야한다. |

## **BATCH_JOB_EXECUTION 테이블**

- JobExecution과 관련된 모든 정보를 저장
- Job이 매번 실행될때, JobExecution이라는 새로운 객체가 있으며, 이 테이블에 새로운 row로 생성

| 컬럼명 | 타입 | Nullable | 설명 |
| --- | --- | --- | --- |
| JOB_EXECUTION_ID | `BIGINT` | NOT NULL | 배치 잡 실행 아이디.<br>실행을 유니크하게 구분.<br>JobExecution 객체의 getId메소드로 획득이 가능 |
| VERSION | `BIGINT` | NULL | 버전정보 |
| JOB_INSTANCE_ID | `BIGINT` | NOT NULL | BATCH_JOB_INSTANCE 테이블의 PK로 FK |
| CREATE_TIME | `TIMESTAMP` | NOT NULL | execution이 생성된 시간 |
| START_TIME | `TIMESTAMP` | NULL | execution이 시작된 시간 |
| END_TIME | `TIMESTAMP` | NULL | execution이 종료된 시간.<br>성공/실패든 시간은 저장.<br>실행중엔 NULL이며 비어 있다면, 배치 잡 수행 오류가 발생하여 종료된 시간을 저장을 수행할 수 없었을 경우에도 NULL |
| STATUS | `VARCHAR(10)` | NULL | execution의 현재 상태 (COMPLETED, STARTED 등..) |
| EXIT_CODE | `VARCHAR(20)` | NULL | execution의 종료 코드 |
| EXIT_MESSAGE | `VARCHAR(2500)` | NULL | job이 종료되는 경유 어떻게 종료 되었는지 저장 |
| LAST_UPDATED | `TIMESTAMP` | NULL | execution이 마지막으로 지속된 시간 |

## **BATCH_JOB_EXECUTION_PARAMS 테이블**

- JobParameter에 대한 정보를 저장하는 테이블
- 하나 이상의 key/value 쌍으로 Job에 전달되며, job이 실행될 때 전달된 파라미터 정보를 저장
- 각 파라미터는 IDENTIFYING이 true로 설정되면, JobParameter 생성시 유니크한 값으로 사용된 경우라는 의미
- 기본키가 존재하지 않음

| 컬럼명 | 타입 | Nullable | 설명 |
| --- | --- | --- | --- |
| JOB_EXECUTION_ID | `BIGINT` | NOT NULL | 배치 잡 실행 아이디.<br>BATCH_JOB_EXECUTION 테이블의 PK로 FK.<br>실행마다 여러 key, value를 row로 저장 |
| PARAMETER_NAME | `VARCHAR(100)` | NOT NULL | 파라미터 이름 |
| PARAMETER_TYPE | `VARCHAR(100)` | NOT NULL | 파라미터 타입 |
| PARAMETER_VALUE | `VARCHAR(2500)` | NOT NULL | 파라미터 값 |
| IDENTIFYING | `CHAR(1)` | NOT NULL | 파라미터가 JobInstance의 유일하게 사용되는 파라미터라면 'Y' |

## **BATCH_JOB_EXECUTION_CONTEXT 테이블**

- Job의 ExecutionContext에 대한 정보 저장
- JobExecution마다 하나의 JobExecutionContext 가지며, 특정 작업 실행에 필요한 작업 수준 데이터 포함
- 일반적으로 실패 후 중단된 부분부터 시작될 수 있도록 실패후 검색해야하는 상태를 나타냄

| 컬럼명 | 타입 | Nullable | 설명 |
| --- | --- | --- | --- |
| JOB_EXECUTION_ID | `BIGINT` | NOT NULL | 배치 잡 실행 아이디.<br>BATCH_JOB_EXECUTION 테이블의 PK로 FK |
| SHORT_CONTEXT | `VARCHAR(2500)` | NOT NULL | 직렬화된 context의 문자으로된 버전 |
| SERIALIZED_CONTEXT | `CLOB` | NULL | 직렬화된 전체 context |

## **BATCH_STEP_EXECUTION 테이블**

- StepExecution과 관련된 정보를 저장
- BATCH_JOB_EXECUTION 테이블과 유사하며 생성된 JobExecution에 대한 step이 하나이상 존재

| 컬럼명 | 타입 | Nullable | 설명 |
| --- | --- | --- | --- |
| STEP_EXECUTION_ID | `BIGINT` | NOT NULL | step 실행의 유니크한 아이디.<br>StepExecution 객체의 getId를 통해 조회가 가능 |
| VERSION | `BIGINT` | NOT NULL | 버전정보 |
| STEP_NAME | `BIGINT` | NOT NULL | BATCH_JOB_INSTANCE 테이블의 PK로 FK |
| JOB_EXECUTION_ID | `BIGINT` | NOT NULL | BATCH_JOB_EXECUTION 테이블에 대한 FK.<br>JobExecution에 StepExecution 속한다는 의미.<br>JobExecution에 대해 Step명은 유니크해야함 |
| START_TIME | `TIMESTAMP` | NULL | execution이 시작된 시간 |
| END_TIME | `TIMESTAMP` | NULL | execution이 종료된 시간.<br>성공/실패든 시간은 저장.<br>실행중엔 NULL이며 비어 있다면, 배치 잡 수행 오류가 발생하여 종료된 시간을 저장을 수행할 수 없었을 경우에도 NULL |
| STATUS | `VARCHAR(10)` | NULL | execution의 현재 상태 (COMPLETED, STARTED 등..) |
| COMMIT_COUNT | `BIGINT` | NULL | execution 동안 트랜잭션 커밋된 카운트 |
| READ_COUNT | `BIGINT` | NULL | 실행된 동안 읽어들인 아이템 수 |
| FILTER_COUNT | `BIGINT` | NULL | 실행동안 필터된 아이템 수 |
| WRITE_COUNT | `BIGINT` | NULL | 실행동안 쓰기된 아이템 수 |
| READ_SKIP_COUNT | `BIGINT` | NULL | 실행동안 읽기시 스킵된 아이템수 |
| WRITE_SKIP_COUNT | `BIGINT` | NULL | 실행동안 쓰기시 스킵된 아이템수 |
| PROCESS_SKIP_COUNT | `BIGINT` | NULL | 실행동안 프로세서가 스킵된 아이템 수 |
| ROLLBACK_COUNT | `BIGINT` | NULL | 실행동안 롤백된 아이템수 |
| EXIT_CODE | `VARCHAR(20)` | NULL | execution의 종료 코드 |
| EXIT_MESSAGE | `VARCHAR(2500)` | NULL | job이 종료되는 경유 어떻게 종료 되었는지 저장 |
| LAST_UPDATED | `TIMESTAMP` | NULL | execution이 마지막으로 지속된 시간 |

## **BATCH_STEP_EXECUTION_CONTEXT 테이블**

- Step의 ExecutionContext 과 관련된 모든 정보를 가짐
- StepExecution마다 하나의 ExecutionContext
- Step execution에 대해서 저장될 필요가 있는 모든 데이터가 저장
- JobInstance가 중단된 위치에서 시작 할 수 있도록 실패 후 검색해야 하는 상태

| 컬럼명 | 타입 | Nullable | 설명 |
| --- | --- | --- | --- |
| STEP_EXECUTION_ID | `BIGINT` | NOT NULL | StepExecution의 키.<br>execution에 연관된 모든 row가 존재 |
| SHORT_CONTEXT | `VARCHAR(2500)` | NOT NULL | 직렬화된 context의 문자으로된 버전 |
| SERIALIZED_CONTEXT | `CLOB` | NULL | 직렬화된 전체 context |

# SpringBatch Sequences

- Spring Batch는 기본적으로 시퀀스 테이블 존재
- 역할
  - 고유 ID 생성 및 관리
    - `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` 등의 테이블에서 고유한 ID를 사용하여 각 배치 작업과 스텝 실행을 식별
    - `BATCH_JOB_SEQ`와 `BATCH_STEP_SEQ` 테이블은 이러한 ID를 생성하기 위한 시퀀스 값을 관리하며, **동시성 제어 및 고유 ID 생성을 보장**하기 위해 사용
  - 동시 실행 시 ID 충돌 방지
    - 다수의 배치 작업이 동시에 실행될 때, 각 작업에 대해 고유한 `JOB_EXECUTION_ID`나 `STEP_EXECUTION_ID`를 부여
    - 시퀀스 테이블에서 **현재 사용 가능한 다음 ID를 조회하고 증가시켜 사용**
    - 데이터베이스 트랜잭션을 이용해 시퀀스 값을 증가시키고 이를 사용함으로써, 다중 프로세스나 다중 스레드 환경에서도 ID의 **중복 없이 일관된 값을 제공**
  - 데이터베이스 호환성
    - 일부 데이터베이스는 기본적으로 `SEQUENCE`를 지원하지 않기 때문에, 스프링 배치는 `BATCH_JOB_SEQ`와 `BATCH_STEP_SEQ` 테이블을 통해 이 기능을 대신 구현

## **BATCH_JOB_SEQ 테이블**

- 배치 잡에 대한 시퀀스 테이블

| 컬럼명 | 타입 | 설명 |
| --- | --- | --- |
| ID | `BIGINT` | 배치 잡의 기본키 |
| UNIQUE_KEY | `CHAR(1)` | 배치 잡 시퀀스를 구별하는 유니크 키 |

## **BATCH_JOB_EXECUTION_SEQ 테이블**

- 배치 잡 execution의 시퀀스 테이블

| 컬럼명 | 타입 | 설명 |
| --- | --- | --- |
| ID | `BIGINT` | 배치 잡 execution의 기본키 |
| UNIQUE_KEY | `CHAR(1)` | 배치 잡 execution 시퀀스를 구별하는 유니크 키 |

## **BATCH_STEP_EXECUTION_SEQ 테이블**

- 배치 step의 execution 시퀀스 테이블

| 컬럼명 | 타입 | 설명 |
| --- | --- | --- |
| ID | `BIGINT` | 배치 step execution의 기본키 |
| UNIQUE_KEY | `CHAR(1)` | 배치 step execution 시퀀스를 구별하는 유니크 키 |
