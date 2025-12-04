---
title: 08 CompositeItemProcessor로 여러 단계에 걸쳐 데이터 Transform하기
tags: SpringBatch
---
# 08 CompositeItemProcessor로 여러 단계에 걸쳐 데이터 Transform하기

## **CompositeItemProcessor 개요**

- CompositeItemProcessor는 Spring Batch에서 제공하는 ItemProcessor 인터페이스를 구현하는 클래스
- 여러 개의 ItemProcessor를 하나의 Processor로 연결하여 여러 단계의 처리를 수행

## **CompositeItemProcessor 주요 구성 요소**

- Delegates : 처리를 수행할 ItemProcessor 목록
- TransactionAttribute : 트랜잭션 속성을 설정
    - 확인 결과 해당 설정은 존재하지 않으며 트렌젝션은 step 수준에서 관리되는 것이 아닌가?

## 장점

- 단계별 처리 : 여러 단계로 나누어 처리를 수행하여 코드를 명확하고 이해
- 재사용 가능성 : 각 단계별 Processor를 재사용하여 다른 job에서도 활용 가능
- 유연성 : 다양한 ItemProcessor를 조합하여 원하는 처리 과정을 구현

## 단점

- 설정 복잡성 : 여러 개의 Processor를 설정하고 관리해야 하기 때문에 설정이 복잡해질 수 있음
- 성능 저하 : 여러 단계의 처리 관정을 거치므로 성늘이 저하될 수 있음

## 실습

## **LowerCaseItemProcessor**

```java
@Slf4j
public class LowCaseItemProcessor implements ItemProcessor<CustomerForMybatis, CustomerForMybatis> {
	@Override
	public CustomerForMybatis process(CustomerForMybatis item) {
		log.info("LowCaseItemProcessor - Customer : {}", item);
		item.setName(item.getName().toLowerCase());
		item.setGender(item.getGender().toLowerCase());

		return item;
	}
}
```

- 이름과 소문자로 변경

### **After20YearsItemProcessor**

```java
@Slf4j
public class After20YearsItemProcessor implements ItemProcessor<CustomerForMybatis, CustomerForMybatis> {
	@Override
	public CustomerForMybatis process(CustomerForMybatis item) {
		log.info("After20YearsItemProcessor - Customer : {}", item);
		item.setAge(item.getAge() + 20);

		return item;
	}
}
```

- 나이에 20살을 더함

### **CompositeItemProcess**

```java
@Bean
public CompositeItemProcessor<CustomerForMybatis, CustomerForMybatis> compositeItemProcessor() {
  return new CompositeItemProcessorBuilder<CustomerForMybatis, CustomerForMybatis>()
      .delegates(
          List.of(
              new LowCaseItemProcessor(),
              new After20YearsItemProcessor()
          )
      ).build();
}
```

- `CompositeItemProcessorBuilder`를 사용하여  `CompositeItemProcessor` 를 생성
    - delegates에 ItemProcessor가 수행할 순서대로 List를 만들어 할당

### 결과 로그
![result.png](/assets/images/posts/2025-05-17-spring-batch/result.png)

- List 순서데로 처리하는지 확인을 위해 각 Processor에 Input Customer 객체를 log를 출력
- delegates에 할당한 ItemProcessor 순서데로 수행하는 것을 확인 가능
