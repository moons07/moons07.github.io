---
title: 02 Spring Batch 코드 설명 및 아키텍처 알아보기
tags: SpringBatch
---
# 02. Spring Batch 코드 설명 및 아키텍처 알아보기

## Spring Batch 아키텍처

- Spring Batch는 Spring DI와 AOP를 지원하는 배치 프레임워크

### Spring Batch 모델

- Tasklet Model
    - 단순한 처리 모델를 가지고 있으며, 로직 자체가 단순한 경우에 주로 사용
    - 다양한 데이터소스나 파일을 한번에 처리하는 경우 사용
    - 기본적으로 `execute` 가 하나의 트렌젝션으로 처리
- Chunk Model
    - 대량의 데이터를 처리할 때 효과적으로 처리
    - Reader - Processor - Writer의 구성으로 처리
    - 기본적으로 읽기, 처리, 쓰기가 하나의 트렌젝션으로 이루어져 성공하면 커밋

### Spring Batch 기본 아키텍처

![spring_batch_doc_sterotyes.png](/assets/images/posts/2025-02-16-02-Spring-Batch-코드-설명-및-아키텍처-알아보기/spring_batch_doc_sterotyes.png)

#### JobLauncher

- 주어진 파라미터들로 Job을 수행하기 위한 인터페이스
- 클라이언트(사용자)에 의해 수행
- 자바 커멘드를 통해서 CommandLineJobRunner를 실행하여 간단하게 배치 프로세스 수행 가능

#### Job

- Spring Batch에서 배치 작업 프로세스의 큰 단위
- Step 인스턴스들을 담는 단순한 컨터이너
- 여러 Step을 논리적으로 묶어 flow를 조합

#### Step

- Job을 구성하는 처리 단위
- 하나의 Job에 여러 Step이 존재
- 여러 Step을 재사용, 병렬처리, 조건분기 등을 수행 가능
    - 재사용 : 정의한 Step을 재사용
    - 병렬처리 : Parallel step, Partitioning step등을 사용하여 여러 step을 동시에 처리
        - Parallel step : 각기 다른 step을 동시에 처리
        - Partitioning step : 동일한 step을 데이터를 나누어서 처리할 때 동시에 처리
    - 조건분기 : JobExecutionDecider, Flow API, StepExecution의 상태 등을 사용하여 어느 Step을 실행할지 분기 실행 가능
- Step은 tasklet model, chunk model의 구현체가 실행

#### Tasklet

- 단순하고 유연하게 배치 처리를 수행

#### ItemReader

- 처리할 데이터를 읽어들이는 역할을 수행

#### ItemProcessor

- 읽은 데이터를 처리
- 데이터를 변환 또는 정제하는 역할
- 생략 가능

#### ItemWriter

- 읽거나 처리한 데이터를 쓰기위한 역할
- DB에 CUD를 하거나 파일로 결과로 출력할 수도 있음

#### JobRepository

- Job과 Step의 상태를 관리
- Spring Batch에서 정의한 테이블 스키마를 기반으로 상태정보를 저장 및 관리

### Spring Batch 흐름

![spring_batch_job_flow.jpeg](/assets/images/posts/2025-02-16-02-Spring-Batch-코드-설명-및-아키텍처-알아보기/spring_batch_job_flow.jpeg)

#### 주요 처리 흐름

1. JobLauncher가 Job 스케쥴러에 의해 시작
2. JobLauncher에 의해 Job 실행
3. Job에서 Step 실행
4. ItemReader에서 데이터를 가져옴
5. IteamProcessor에서 읽은 데이터를 변환 또는 가공
6. ItemWriter에서 처리된 데이터를 출력 (DB에 CUD를 하거나 파일로 생성)

#### Job 정보의 흐름

1. JobLauncher는 JobRepository를 통해 데이터베이스에 JobInstance를 등록
2. JobLauncher는 JobRepository를 통해 Job 실행이 DB에 시작되었음을 등록
3. JobStep은 JobRepository를 통해 데이터베이스의 I/O 레코드 수, 상태 등의 다양한 정보를 업데이트
4. JobLauncher는 JobRepository를 통해 Job 실행이 완료되었음을 DB에 업데이트

#### Spring Batch 저장 정보

| 구성 요소                            | 역할                                                                                                                                                                                                                  |
|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| JobInstance                      | - Job의 논리적 실행 <br>- Job 이름과 인수로 식별 (동일한 Job이 이름과 인수로 실행하면 동일한 JobInstance의 실행으로 식별)<br>- Job의 오류로 인해 중단된 경우 중간부터 재실행을 지원<br>- 재실행을 지원하지 않거나 Job 처리가 완료된 경우 재실행한다면 중복 수행되지 않도록 JobInstanceAlreadyCompleteException 발생 |
| JobExecution<br>ExecutionContext | - JobExecution은 Job의 물리적 실행<br>- JobExecution은 동일한 Job이 실행되어도 다른 JobExecution으로 지칭되어 여러번 실행(JobInstance:JobExecution는 1:N 관계)<br>- ExecutionContext는 JobExecution에서 프로세스 진행 상황과 같은 메타 데이터를 공유하기 위한 영역<br>- ExecutionContext는 주로 Spring Batch가 프레임워크 상태를 기록할 수 있도록 하는데 사용되지만 애플리케이션에서 ExecutionContext에 액세스하는 수단으로도 제공<br>- Job의 ExecutionContext에 저장된 객체는 java.io.Serialized를 구현하는 클래스 |
| StepExecution<br>ExecutionContext | - StepExecution은 Step의 물리적 실행<br>- JobExecution : StepExecutio는 1:N 관계<br>- JobExecution과 유사하게 ExecutionContext는 Step에서 데이터를 공유하는 영역<br>- 여러 Step에서 공유할 필요가 없는 정보는 Step의 ExecutionContext를 사용<br>- Step의 ExecutionContext에 저장된 객체는 java.io.Serialized를 구현하는 클래스 |
| JobRepository | - JobExecution 또는 StepExecution과 같은 배치 애플리케이션의 실행 결과 및 상태를 관리하기 위한 데이터를 관리하고 유지하는 기능이 제공<br>- 배치 애플리케이션에서 프로세스는 Java 프로세스를 시작하여 시작되고 Java 프로세스도 프로세스 종료와 함께 종료<br>- 따라서, 영속성을 가진 데이터베이스와 같은 저장소에 저장 |

### 기본 배치 애플리케이션

#### Tasklet 구현체 생성

```java
package org.study.spring_batch_study.jobs.task01;

import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.StepContribution;
import org.springframework.batch.core.scope.context.ChunkContext;
import org.springframework.batch.core.step.tasklet.Tasklet;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.beans.factory.InitializingBean;

@Slf4j
public class GreetingTasklet implements Tasklet, InitializingBean {
    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        log.info("---- GreetingTasklet execute ----");
        log.info("GreetingTasklet : {}, {}", contribution, chunkContext);
        return RepeatStatus.FINISHED;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        log.info("---- GreetingTasklet afterPropertiesSet ----");
    }
}
```

- Tasklet, InitializingBean 인터페이스를 구현
- Tasklet.execute
    - RepeatStatus를 반환하며, Tasklet의 반복여부를 결정
        - FINISHED : Tasklet 종료
        - CONTINUABLE : Tasklet 계속 수행
        - continueIf(condition) : 조건에따라 종료할지 계속 수행할지 결정
            - true : 계속 진행
            - false : 종료
- InitializingBean.afterPropertiesSet()
    - Tasklet Bean  초기화시 Property를 설정한 후 실행되는 메소드
    - Bean 초기화시 별도 로직이 필요 없다면 인터페이스를 구현하지 않아도됨

#### @Configuration을 통해서 생성할 배치 빈을 스프링에 등록

- SpringBoot는 @Configuration을 통해 빈 등록설정을 할 수 있도록 애노테이션을 제공

#### Job, Step을 생성하고 빈에 등록

- Job, Step을 생성하고 빈을 등록

```java
package org.study.spring_batch_study.jobs.task01;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.core.step.tasklet.Tasklet;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class BasicTaskJobConfiguration {
    private final PlatformTransactionManager platformTransactionManager;

    @Bean
    public Tasklet greetingTasklet() {
        return new GreetingTasklet();
    }

    @Bean
    public Step myStep(JobRepository jobRepository, PlatformTransactionManager platformTransactionManager) {
        log.info("----- init myStep -----");

        return new StepBuilder("myStep", jobRepository)
                .tasklet(greetingTasklet(), platformTransactionManager)
                .build();
    }

    @Bean
    public Job myJob(Step myStep, JobRepository jobRepository) {
        log.info("----- init myJob -----");

        return new JobBuilder("myJob", jobRepository)
                .incrementer(new RunIdIncrementer())
                .start(myStep)
                .build();
    }
}
```

- Step 등록
    - myStep이라는 step 빈을 등록
    - Spring Batch는 데이터소스와 함께 작업하므로 트렌젝션관리를 위해 PlatformTransactionManager가 필요
    - StepBuilder를 통해 step을 생성
    - 해당 step의 정보는 JobRepository에 등록
    - `greetingTasklet()` 을 통해 step에 tasklet을 주입
    - build를 통해 step을 생성하고 반환하여 빈으로 등록
- Job 등록
    - Job엔 Step과 JobRepository가 필요
    - Job은 JobRepository에 등록
    - JobBuilder를 통해 job 생성
    - incrementer는 job이 지속적으로 실행될때, 유일성을 구분할 수 있는 방법
    - `RunIdIncrementer`는 job의 실행할 때 지속적으로 증가시키면 유일한 job을 실행
    - `start(myStep)`을 통해 job이 시작점을 지정
    - build를 통해 job을 생성하고 반환하여 빈으로 등록

#### 실행결과

![result.png](/assets/images/posts/2025-02-16-02-Spring-Batch-코드-설명-및-아키텍처-알아보기/result.png)

- 실행 순서
    1. Tasklet bean 등록시 afterPropertiesSet 호출
    2. Job 실행
    3. Step 실행
    4. Tasklet 실행
