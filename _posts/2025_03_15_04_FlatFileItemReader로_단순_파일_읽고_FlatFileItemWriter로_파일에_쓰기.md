---
title: 04 FlatFileItemReader로 단순 파일 읽고, FlatFileItemWriter로 파일에 쓰기
tags: SpringBatch
---
# 04 FlatFileItemReader로 단순 파일 읽고, FlatFileItemWriter로 파일에 쓰기

## FlatFileItemReader 개요

- FlatFileItemReader는 Spring Batch에서 제공하는 기본적인 ItemReader로, 텍스트 파일로부터 데이터를 읽음
- 고정 길이, 구분자 기반, 멀티라인 등 다양한 형식의 텍스트 파일을 지원

### FlatFileItemRader 장점, 단점

- 장점
    - 간단하고 효율적인 구현
        - 설정 및 사용이 간편하며, 대규모 데이터 처리에도 효율적
    - 다양한 텍스트 파일 형식 지원
        - 고정 길이, 구분자 기반, 멀티라인 등 다양한 형식의 텍스트 파일을 읽음
    - 확장 가능성
        - 토크나이저, 필터 등을 통해 기능을 확장
    - 사용처
        - 고정 길이, 구분자 기반, 멀티라인 등 다양한 형식의 텍스트 파일 데이터 처리
- 단점
    - 복잡한 데이터 구조 처리에는 적합하지 않음

## **FlatFileItemReader 주요 구성 요소**

- Resource : 읽을 텍스트 파일을 지정
- LineMapper : 텍스트 파일의 각 라인을 Item으로 변환하는 역할
- LineTokenizer : 텍스트 파일의 각 라인을 토큰으로 분리하는 역할
- FieldSetMapper : 토큰을 Item의 속성에 매핑하는 역할
- SkippableLineMapper : 오류 발생 시 해당 라인을 건너뜀
- LineCallbackHandler : 라인별로 처리를 수행
- ReadListener : 읽기 시작, 종료, 오류 발생 등의 이벤트를 처리

## **FlatFileItemReader 샘플코드**

### **Customer 모델 생성**

```java
public class Customer {
  private String name;
  private int age;
  private String gender;
}
```

- 읽어들인 정보를 Customer 객체에 매핑할 수 있도록 객체를 정의

### **FlatFileItemReader bean 생성**

```java
@Bean
public FlatFileItemReader<Customer> flatFileItemReader() {
	return new FlatFileItemReaderBuilder<Customer>()
		.name("FlatFileItemReader")
    .resource(new ClassPathResource("./customer.csv"))
    .encoding("UTF-8")
    .delimited().delimiter(",")
    .names("name", "age", "gender")
    .targetType(Customer.class)
    .build();
}
```

- FlatFileItemReader를 생성하고, Customer 객체에 등록하여 반환
    - name
        - Reader의 이름을 지정
    - resource
        - 클래스 패스 내부에 존재하는 csv 파일을 읽음
    - encoding
        - 파일 데이터의 인코딩을 추가
    - delimited
        - 구분자로 설정되어 있음을 의미
    - delimiter
        - 구분자를 무엇으로할지 지정
    - names
        - 구분자로 구분된 데이터의 이름을 지정한
    - targetType
        - 구분된 데이터를 어느 모델에 넣을지 클래스 타입을 지정

## **FlatFileItemWriter 개요**

- FlatFileItemWriter는 Spring Batch에서 제공하는 ItemWriter 인터페이스를 구현하는 클래스
- 데이터를 텍스트 파일로 출력하는 데 사용

## **FlatFileItemWriter 구성요소**

- Resource : 출력 파일 경로를 지정
- LineAggregator : Item을 문자열로 변환하는 역할
- HeaderCallback : 출력 파일 헤더를 작성하는 역할
- FooterCallback : 출력 파일 푸터를 작성하는 역할
- Delimiter : 항목 사이 구분자를 지정
- AppendMode : 기존 파일에 추가할지 여부를 지정

## **FlatFileItemWriter 장단점**

- 장점
    - 간편성 : 텍스트 파일로 데이터를 출력하는 간편한 방법을 제공
    - 유연성 : 다양한 설정을 통해 원하는 형식으로 출력 파일을 만듬
    - 성능 : 대량의 데이터를 빠르게 출력
- 단점
    - 형식 제약 : 텍스트 파일 형식만 지원
    - 복잡한 구조 : 복잡한 구조의 데이터를 출력할 경우 설정이 복잡
    - 오류 가능성 : 설정 오류 시 출력 파일이 손상

## **FlatFileItemWriter 샘플코드**

### **FlatFileItemWriter bean 생성**

```java
@Bean
public FlatFileItemWriter<Customer> flatFileItemWriter() {
	return new FlatFileItemWriterBuilder<Customer>()
	  .name("flatFileItemWriter")
	  .resource(new FileSystemResource("./output/customer_new.csv"))
	  .encoding("UTF-8")
	  .delimited().delimiter("\t")
	  .names("Name", "Age", "Gender")
	  .append(false)
	  .lineAggregator(new CustomerLineAggregator())
	  .headerCallback(new CustomerHeader())
	  .footerCallback(new CustomerFooter(aggregateInfos))
	  .build();
}
```

- name
    - Writer의 이름을 지정
- resource
    - 저장할 최종 파일 이름
- encoding
    - 저장할 파일의 인코딩 타입
- delimited().delimiter
    - 각 필드를 구분할 딜리미터를 지정
- append
    - true인 경우 기존 파일에 첨부
    - false인 경우 새로운 파일 생성
- lineAggregator
    - 라인 구분자를 지정
- headerCallback
    - 출력 파일의 헤더를 지정
- footerCallback
    - 출력 파일의 푸터를 지정

### CustomerLineAggregator

```java
package org.study.spring_batch_study.chater4.model;

import org.springframework.batch.item.file.transform.LineAggregator;

public class CustomerLineAggregator implements LineAggregator<Customer> {
    @Override
    public String aggregate(Customer item) {
        return item.getName() + "," + item.getAge();
    }
}
```

- LineAggregator는 FlatFile에 저장할 아이템들을 String으로 변환하는 방법을 지정

### CustomerHeader

```java
package org.study.spring_batch_study.chater4.model;

import org.springframework.batch.item.file.FlatFileHeaderCallback;

import java.io.IOException;
import java.io.Writer;

public class CustomerHeader implements FlatFileHeaderCallback {
    @Override
    public void writeHeader(Writer writer) throws IOException {
        writer.write("ID,AGE");
    }
}
```

- FlatFileHeaderCallback은 writeHeader를 구현
- 출력될 파일의 header를 달아주는 역할

### CustomerFooter

```java
package org.study.spring_batch_study.chater4.model;

import org.springframework.batch.item.file.FlatFileFooterCallback;

import java.io.IOException;
import java.io.Writer;
import java.util.concurrent.ConcurrentHashMap;

public class CustomerFooter implements FlatFileFooterCallback {
    ConcurrentHashMap<String, Integer> aggregateCustomers;

    public CustomerFooter(ConcurrentHashMap<String, Integer> aggregateCustomers) {
        this.aggregateCustomers = aggregateCustomers;
    }

    @Override
    public void writeFooter(Writer writer) throws IOException {
        writer.write("총 고객 수: " + aggregateCustomers.get("TOTAL_CUSTOMERS"));
        writer.write(System.lineSeparator());
        writer.write("총 나이: " + aggregateCustomers.get("TOTAL_AGES"));
    }
}
```

- FlatFileFooterCallback은 writeFooter를 구현
- 출력될 파일의 footer를 달아주는 역할
- 결과를 집계하여 총 고객 수, 나이를 출력
- 이 때 전달되는 HashMap을 전달하여, 결과를 출력

### **AggregateCustomerProcessor**

```java
package org.study.spring_batch_study.chater4.processor;

import org.springframework.batch.item.ItemProcessor;
import org.study.spring_batch_study.chater4.model.Customer;

import java.util.concurrent.ConcurrentHashMap;

public class AggregateCustomerProcessor implements ItemProcessor<Customer, Customer> {
    private static final String CUSTOMERS_KEY = "TOTAL_CUSTOMERS";
    private static final String AGES_KEY = "TOTAL_AGES";

    ConcurrentHashMap<String, Integer> aggregateCustomers;

    public AggregateCustomerProcessor(ConcurrentHashMap<String, Integer> aggregateCustomers) {
        this.aggregateCustomers = aggregateCustomers;
    }

    @Override
    public Customer process(Customer item) throws Exception {
        aggregateCustomers.putIfAbsent(CUSTOMERS_KEY, 0);
        aggregateCustomers.putIfAbsent(AGES_KEY, 0);

        aggregateCustomers.put(CUSTOMERS_KEY, aggregateCustomers.get(CUSTOMERS_KEY) + 1);
        aggregateCustomers.put(AGES_KEY, aggregateCustomers.get(AGES_KEY) + item.getAge());

        return item;
    }
}

```

- ItemProcessor의 process 메소드 구현
    - 각 item(고객, 나이)를 집계
