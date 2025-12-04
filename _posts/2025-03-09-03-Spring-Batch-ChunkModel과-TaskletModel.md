---
title: 03 Spring Batch ChunkModel과 TaskletModel
tags: SpringBatch
---
# 03 Spring Batch ChunkModel과 TaskletModel

## Chunk Model

![chunkModel.png](/assets/images/posts/2025-03-09-spring-batch/chunkModel.png)

- Chunk Model은 처리할 데이터를 일정(청크)단위로 처리하는 방식
- ChunkOrientedTasklet은 청크 처리를 지원하는 Tasklet의 구체적인 클래스 연할을 수행
- 청크에 포함될 데이터의 최대 레코드(청크 크기)는 commit-interval으로 조정이 가능
    - 청크 크기와 commit-interval은 동일한 것이 아닌 보통 동일하게 설정
    - 단, 아래와 같은 이유로 청크 크기와 commit-interval이 다를 수 있다
        - 청크 크기가 크고 해당 데이터를 전부 DB에 commit을 한다면 DB에 부하가 될수 있다, 따라서 DB에 부하를 줄이기 위해서 commit-interval을 조정
- ItemReader, ItemProcessor, ItemWriter는 청크 단위를 처리하기 위한 인터페이스
- ChunkOrientedTasklet는 청크 당위에 따라 ItemReader, ItemProcessor, ItemWriter 구현체를 각각 반복 호출

### ItemReader

- ItemReader는 청크 크기 만큼 데이터를 읽음
- ItemReader는 구현체를 직접 구현할 수 있지만 Spring Batch에서 이미 구현된 다양한 ItemReader 구현체를 제공

#### 제공하는 ItemReader 구현체

- FlatFileItemReader
    - 구조화 되지 않은 파일을 플랫파일이라고하며 해당 파일을 읽음 (대표적으로 CSV)
    - 읽어드린 데이터를 객체로 매핑하기 위해 delimeter를 기준으로 매핑 룰을 이용하거나 입력에 대한Resource Object를 이용하여 커스텀하게 객체를 맵핑
- StaxEventItemReader
    - XML파일을 StAX(Streaming API for XML))기반으로 읽음
        - **이벤트 기반 처리**: StAX는 XML 문서를 이벤트 중심으로 처리 즉, XML 요소가 시작되거나 끝날 때 이벤트가 발생하여 이를 통해 데이터를 읽거나 쓸 수 있음
        - **풀 기반**: StAX는 두 가지 접근 방식을 제공
            - **Cursor API**: XML 스트림을 순차적으로 탐색할 수 있도록 커서를 제공
            - **Event API**: XML 요소의 시작과 끝, 텍스트 등의 이벤트를 처리하는 방식
        - **메모리 효율성**: StAX는 전체 XML 문서를 메모리에 로드하지 않고, 필요한 부분만을 스트리밍하여 처리하므로 메모리 사용이 효율적
        - **비동기 처리 가능**: StAX는 XML을 비동기적으로 처리할 수 있어, 대량의 데이터를 처리할 때 유용
- JdbcPagingItemReader / JdbcCursorItemReader
    - JDBC를 사용하여 SQL을 실행하고 데이터베이스의 레코드를 읽음
    - DB에서 모든 레코드를 읽어서 처리하면 메모리가 부족으로 오류가 발생할 수 있으므로 데이터를 나누어서 처리하고 처리한 데이터는 메모리에서 제거되도록 필요
        - JdbcPagingItemReader는 JdbcTemplate을 이용하여 각 페이지에 대한 SELECT SQL을 나누어 처리하는 방식으로 구현
        - JdbcCursorItemReader는 JDBC 커서를 이용하여 하나의 SELECT SQL을 발행하여 구현
- MyBatisPagingItemReader / MyBatisCursorItemReader
    - MyBatis를 사용하여 데이터베이스의 레코드를 읽음
    - MyBatis에서 제공하는 MyBatis-Spring 라이브러리에서 제공
    - Paging과 Cursor의 차이점은 MyBatis를 구현방법이 다를뿐이지 Jdbc(Paging / CursorI)ItemReader과 동일
- JmsItemReader / AmqpItemReader
    - 메시지를 JMS 또는 AMQP에서 읽음

### ItemProcessor

- ItemProcessor는 Reader로 읽어들인 청크 데이터들을 처리(데이터 변환, 변경 또는  외부 인터페이스 호출등을 수행)
- ItemProcessor는 chunk model는 없어도 됨

```java
public class MyItemProcessor implements
      ItemProcessor<MyInputObject, MyOutputObject> {  // 입축력 데아토 타입을 제네릭 타입으로 받음
	@Override
	public MyOutputObject process(MyInputObject item) throws Exception {  // 입력 데이터를 arg를 받음

		MyOutputObject processedObject = new MyOutputObject();  // 출력 객체를 생성 (변환된 데이터 등)

        // Coding business logic for item of input data

		return processedObject; // 출력 객체 반환
	}
}
```

- ItemReader와 마찬가지로 Spring Batch에는 다양한 ItemProcessor 구현체도 제공

#### 제공하는 ItemProcessor 구현체

- PassThroughItemProcessor
    - 아무 작업도 수행하지 않음
    - 입력된 데이터의 변경이나 처리가 필요하지 않는경우 사용
    - Step 생성시 ItemProcessor 생략하는 것과 동일
- ValidatingItemProcessor
    - 입력된 데이터를 체크
    - 입력 확인 규칙을 구현하려면 Spring Batch 전용 org.springframework.batch.item.validator.Validator를 구현
    - org.springframework.validation.Validator 어댑터인 SpringValidator와 org.springframework.validation의 규칙을 제공
- CompositeItemProcessor
    - 동일한 입력 데이터에 대해 여러 개의 ItemProcessor를 순차적으로 실행
    - ValidatingItemProcessor를 사용하여 입력 확인을 수행한 후 비즈니스 로직을 실행하려는 경우 활성화

### ItemWriter

- 나 IteamReader에서 읽은 데이터나 ItemProcessor에서 처리한 데이터를 청크 단위가 ItemWriter로 전달되어 데이터를 저장하거나 파일처리를 수행
- ItemWriter 또한 다양한 구현체를 제공

#### 제공하는 ItemWriter 구현체

- FlatFileItemWriter
    - 처리된 Java객체를 CSV 파일과 같은 플랫 파일로 작성
    - 파일 라인에 대한 매핑 규칙은 구분 기호 및 개체에서 사용자 정의로 사용 가능
- StaxEventItemWriter
    - XML파일로 자바 객체를 쓰기
- JdbcBatchItemWriter
    - JDBC를 사용하여 SQL을 수행하고 자바 객체를 DB에 쓰기
    - 내부적으로 JdbcTemplate를 사용
- MyBatisBatchItemWriter
    - Mybatis를 사용하여 자바 객체를 DB에 쓰기
    - MyBatis-Spring는 MyBatis에 의해서 제공되는 라이브러리를 이용
- JmsItemWriter / AmqpItemWriter
    - JMS혹은 AMQP로 자바 객체의 메시지를 전송

## Tasklet Model

- 청크 단위의 체리가 딱 맞지 않을 경우 Takslet Model이 유용
- 예를 들어, 한번에 하나의 레코드만 읽어서 쓰기해야하는 경우 Tasklet Model이 적합
- Tasklet 모델을 사용하면서 Spring Batch에서 제공하는 Tasklet 인터페이스를 구현

### Tasklet 구현 클래스

- SystemCommandTasklet
    - 시스템 명령어를 비동기적으로 실행하는 Tasklet
    - 명령 속성에 수행해야할 명령어를 지정하여 사용 가능
    - 시스템 명령은 호출하는 스레드와 다른 스레드에 의해 실행되므로 프로세스 도중 타임아웃을 설정하고, 시스템 명령의 실행 스레드를 취소 가능
- MethodInvokingTaskletAdapter
    - POJO클래스의 특정 메소드를 실행하기 위한 Tasklet
    - targetObject 속성에 대상 클래스의 빈을 지정하고, targetMethod속성에 실행할 메소드 이름을 지정
    - POJO 클래스는 배치 처리 종료 상태를 메소드의 반환 값으로 반환이 가능하지만, 이경우 사실은 ExitStatus를 반환값으로 설정
    - 다른 타입의 값이 반환될 경우 반환값과 상관없이 "정상 종료(ExitStatus:COMPLETED)" 상태로 간주

    ```java
    @Bean
    public MethodInvokingTaskletAdapter myTasklet() {
    	MethodInvokingTaskletAdapter adapter = new MethodInvokingTaskletAdapter();

    	adapter.setTargetObject(fooDao());
    	adapter.setTargetMethod("updateFoo");

    	return adapter;
    }
    ```
