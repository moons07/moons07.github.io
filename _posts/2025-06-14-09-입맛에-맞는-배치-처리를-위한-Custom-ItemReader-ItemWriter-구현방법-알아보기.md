---
title: 09 입맛에 맞는 배치 처리를 위한 Custom ItemReader/ItemWriter 구현방법 알아보기
tags: SpringBatch
---
# 09 입맛에 맞는 배치 처리를 위한 Custom ItemReader/ItemWriter 구현방법 알아보기

## **QuerydslPagingItemReader 개요**

- Querydsl은 SpringBatch의 공식 ItemReader가 아님
- AbstractPagingItemReader를 이용하여 Querydsl 을 활용할 수 있도록 ItemReader를 실습
- Querydsl 기능 활용 : Querydsl의 강력하고 유연한 쿼리 기능을 사용하여 데이터를 효율적으로 읽음
- JPA 엔티티 추상화 : JPA 엔티티에 직접 의존하지 않고 추상화된 쿼리를 작성하여 코드 유지 관리성을 높임
- 동적 쿼리 지원 : 런타임 시 조건에 따라 동적으로 쿼리를 생성

### QuerydslPagingItemReader 생성하기

```java
package org.study.spring_batch_study.reader;

import com.querydsl.jpa.impl.JPAQuery;
import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import org.springframework.batch.item.database.AbstractPagingItemReader;
import org.springframework.util.ClassUtils;
import org.springframework.util.CollectionUtils;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Function;

public class QuerydslPagingItemReader<T> extends AbstractPagingItemReader<T> {
    private EntityManager em;
    private final Boolean alwaysReadFromZero;
    private final Function<JPAQueryFactory, JPAQuery<T>> querySupplier;

    public QuerydslPagingItemReader(EntityManagerFactory entityManagerFactory, Function<JPAQueryFactory, JPAQuery<T>> querySupplier, int chunkSize) {
        this(ClassUtils.getShortName(QuerydslPagingItemReader.class), entityManagerFactory, querySupplier, chunkSize, false);
    }

    public QuerydslPagingItemReader(String name, EntityManagerFactory entityManagerFactory, Function<JPAQueryFactory, JPAQuery<T>> querySupplier, int chunkSize, Boolean alwaysReadFromZero) {
        super.setPageSize(chunkSize);
        setName(name);
        this.querySupplier = querySupplier;
        this.em = entityManagerFactory.createEntityManager();
        this.alwaysReadFromZero = alwaysReadFromZero;
    }

    @Override
    protected void doClose() throws Exception {
        if (em != null) {
            em.close();
        }
        super.doClose();
    }

    @Override
    protected void doReadPage() {
        initQueryResult();

        JPAQueryFactory jpaQueryFactory = new JPAQueryFactory(em);
        long offset = 0;
        if (!alwaysReadFromZero) {
            offset = (long) getPage() * getPageSize();
        }

        JPAQuery<T> query = querySupplier.apply(jpaQueryFactory).offset(offset).limit(getPageSize());

        List<T> queryResult = query.fetch();
        for (T entity: queryResult) {
            em.detach(entity);
            results.add(entity);
        }
    }

    private void initQueryResult() {
        if (CollectionUtils.isEmpty(results)) {
            results = new CopyOnWriteArrayList<>();
        } else {
            results.clear();
        }
    }
}

```

- AbstractPagingItemReader을 상속
- AbstractPagingItemReader는 어댑터 패턴으로 상속받는 클래스에선 doReadPage만 구현

#### 생성자

```java
public QuerydslPagingItemReader(String name, EntityManagerFactory entityManagerFactory, Function<JPAQueryFactory, JPAQuery<T>> querySupplier, int chunkSize, Boolean alwaysReadFromZero) {
        super.setPageSize(chunkSize);
        setName(name);
        this.querySupplier = querySupplier;
        this.em = entityManagerFactory.createEntityManager();
        this.alwaysReadFromZero = alwaysReadFromZero;
    }
```

- name : ItemReader를 구분하기 위한 이름
- entityManagerFactory : JPA를 이용하기 위해서 entityManagerFactory 를 전달
- Function<JPAQueryFactory, JPAQuery> : JPAQuery를 생성하기 위한 Functional Interface
    - 입력 파마미터로 JPAQueryFactory 를 입력으로 전달 받음
    - 반환값은 JPAQuery 형태의 queryDSL 쿼리
    - chunkSize : 한번에 페이징 처리할 페이지 크기
    - alwaysReadFromZero : 항상 0부터 페이징을 읽을지 여부를 지정 (만약 paging 처리된 데이터 자체를 수정하는경우 배치처리 누락이 발생할 수 있으므로 이를 해결하기 위한 방안으로 사용)

#### doClose()

- doClose는 기본적으로 AbstractPagingItemReader를 자체 구현되어 있지만 EntityManager자원을 해제하기 위해서 em.close() 를 수행

#### doReadPage()

```java
@Override
protected void doReadPage() {
    initQueryResult();

    JPAQueryFactory jpaQueryFactory = new JPAQueryFactory(em);
    long offset = 0;
    if (!alwaysReadFromZero) {
        offset = (long) getPage() * getPageSize();
    }

    JPAQuery<T> query = querySupplier.apply(jpaQueryFactory).offset(offset).limit(getPageSize());

    List<T> queryResult = query.fetch();
    for (T entity: queryResult) {
        em.detach(entity);
        results.add(entity);
    }
}
```

- 실제로 구현해야할 추상 메소드
- JPAQueryFactory를 통해서 함수형 인터페이스로 지정된 queryDSL에 적용할 QueryFactory
- alwaysReadFromZero 가 false라면 offset과 limit을 계속 이동하면서 조회하도록 offset을 계산
- querySupplier.apply
    - 제공한 querySupplier에 JPAQueryFactory를 적용하여 JPAQuery를 생성
    - 페이징을 위해서 offset, limit을 계산된 offset과 pageSize (청크크기) 를 지정하여 페이징 처리
- fetch
    - 결과를 패치하여 패치된 내역을 result 할당
    - entityManager에서 detch하여 변경이 실제 DB에 반영되지 않도록 영속성 객체에서 제외
- initQueryResult
    - 매 페이징 결과를 반환할때 페이징 결과만 반환하기 위해서 초기화
    - 만약 결과 객체가 초기화 되어 있지 않다면 CopyOnWriteArrayList 객체를 신규로 생성

### 편의를 위해서 Builder 생성하기

```java
package org.study.spring_batch_study.reader;

import com.querydsl.jpa.impl.JPAQuery;
import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.EntityManagerFactory;
import org.springframework.util.ClassUtils;

import java.util.function.Function;

public class QuerydslPagingItemReaderBuilder<T> {
	private EntityManagerFactory entityManagerFactory;
	private Function<JPAQueryFactory, JPAQuery<T>> querySupplier;

	private int chunkSize = 10;

	private String name;

	private Boolean alwaysReadFromZero;

	public QuerydslPagingItemReaderBuilder<T> entityManagerFactory(EntityManagerFactory entityManagerFactory) {
		this.entityManagerFactory = entityManagerFactory;
		return this;
	}

	public QuerydslPagingItemReaderBuilder<T> querySupplier(Function<JPAQueryFactory, JPAQuery<T>> querySupplier) {
		this.querySupplier = querySupplier;
		return this;
	}

	public QuerydslPagingItemReaderBuilder<T> chunkSize(int chunkSize) {
		this.chunkSize = chunkSize;
		return this;
	}

	public QuerydslPagingItemReaderBuilder<T> name(String name) {
		this.name = name;
		return this;
	}

	public QuerydslPagingItemReaderBuilder<T> alwaysReadFromZero(Boolean alwaysReadFromZero) {
		this.alwaysReadFromZero = alwaysReadFromZero;
		return this;
	}

	public QuerydslPagingItemReader<T> build() {
		if (name == null) {
			this.name = ClassUtils.getShortName(QuerydslPagingItemReader.class);
		}
		if (this.entityManagerFactory == null) {
			throw new IllegalArgumentException("EntityManagerFactory can not be null.!");
		}
		if (this.querySupplier == null) {
			throw new IllegalArgumentException("Function<JPAQueryFactory, JPAQuery<T>> can not be null.!");
		}
		if (this.alwaysReadFromZero == null) {
			alwaysReadFromZero = false;
		}
		return new QuerydslPagingItemReader<>(this.name, entityManagerFactory, querySupplier, chunkSize, alwaysReadFromZero);
	}
}
```

- QuerydslPagingItemReader 객체를 편하게 생성하기 위한 Builder 클래스

## 소스 샘플

```java
package org.study.spring_batch_study.config;

import jakarta.persistence.EntityManagerFactory;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.file.FlatFileItemWriter;
import org.springframework.batch.item.file.builder.FlatFileItemWriterBuilder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;
import org.springframework.transaction.PlatformTransactionManager;
import org.study.spring_batch_study.model.Customer;
import org.study.spring_batch_study.model.QCustomer;
import org.study.spring_batch_study.processor.CustomerItemProcessor;
import org.study.spring_batch_study.reader.QuerydslPagingItemReader;
import org.study.spring_batch_study.reader.QuerydslPagingItemReaderBuilder;

import javax.sql.DataSource;

@Slf4j
@Configuration
public class QueryDSLPagingReaderJobConfig {
	public static final int CHUNK_SIZE = 2;
	public static final String ENCODING = "UTF-8";
	public static final String QUERYDSL_PAGING_CHUNK_JOB = "QUERYDSL_PAGING_CHUNK_JOB";

	@Autowired
	DataSource dataSource;

	@Autowired
	EntityManagerFactory entityManagerFactory;

	@Bean
	public QuerydslPagingItemReader<Customer> customerQuerydslPagingItemReader() {
		return new QuerydslPagingItemReaderBuilder<Customer>()
				.name("customerQuerydslPagingItemReader")
				.entityManagerFactory(entityManagerFactory)
				.chunkSize(2)
				.querySupplier(jpaQueryFactory -> jpaQueryFactory.select(QCustomer.customer).from(QCustomer.customer).where(QCustomer.customer.age.gt(20)))
				.build();
	}

	@Bean
	public FlatFileItemWriter<Customer> customerQuerydslFlatFileItemWriter() {

		return new FlatFileItemWriterBuilder<Customer>()
				.name("customerQuerydslFlatFileItemWriter")
				.resource(new FileSystemResource("./output/customer_new_v2.csv"))
				.encoding(ENCODING)
				.delimited().delimiter("\t")
				.names("Name", "Age", "Gender")
				.build();
	}

	@Bean
	public Step customerQuerydslPagingStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) throws Exception {
		log.info("------------------ Init customerQuerydslPagingStep -----------------");

		return new StepBuilder("customerJpaPagingStep", jobRepository)
				.<Customer, Customer>chunk(CHUNK_SIZE, transactionManager)
				.reader(customerQuerydslPagingItemReader())
				.processor(new CustomerItemProcessor())
				.writer(customerQuerydslFlatFileItemWriter())
				.build();
	}

	@Bean
	public Job customerJpaPagingJob(Step customerJdbcPagingStep, JobRepository jobRepository) {
		log.info("------------------ Init customerJpaPagingJob -----------------");
		return new JobBuilder(QUERYDSL_PAGING_CHUNK_JOB, jobRepository)
				.incrementer(new RunIdIncrementer())
				.start(customerJdbcPagingStep)
				.build();
	}
}
```

- 위에서 생성한 Reader를 통해 queryDSL을 이용하여 배치를 수행 가능

## CustomItemWriter 개요

- CustomItemWriter는 Spring Batch에서 제공하는 기본 ItemWriter 인터페이스를 구현하여 직접 작성한 ItemWriter 클래스
- 기본 ItemWriter 클래스로는 제공되지 않는 특정 기능을 구현할 때 사용

### 구성 요소

- ItemWriter 인터페이스 구현 : write() 메소드를 구현하여 원하는 처리를 수행
- 필요한 라이브러리 및 객체 선언 : 사용할 라이브러리 및 객체를 선언
- 데이터 처리 로직 구현 : write() 메소드에서 데이터 처리 로직을 구현

### 장점

- 유연성 : 기본 ItemWriter 클래스로는 제공되지 않는 특정 기능을 구현
- 확장성 : 다양한 방식으로 데이터 처리를 확장
- 제어 가능성 : 데이터 처리 과정을 완벽하게 제어

### 단점

- 개발 복잡성 : 기본 ItemWriter 클래스보다 개발 과정이 더 복잡
- 테스트 어려움 : 테스트 작성이 더 어려울 수도 있음
- 디버깅 어려움 : 문제 발생 시 디버깅이 더 어려울 수도 있음

### 소스 샘플

#### CustomService

```java
package org.study.spring_batch_study.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.study.spring_batch_study.model.CustomerForMybatis;

import java.util.Map;

@Slf4j
@Service
public class CustomService {
	public Map<String, String> processToOtherService(CustomerForMybatis item) {

		log.info("Call API to OtherService....");

		return Map.of("code", "200", "message", "OK");
	}
}

```

#### **CustomItemWriter**

```java
package org.study.spring_batch_study.writer;

import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.item.Chunk;
import org.springframework.batch.item.ItemWriter;
import org.springframework.stereotype.Component;
import org.study.spring_batch_study.model.CustomerForMybatis;
import org.study.spring_batch_study.service.CustomService;

@Slf4j
@Component
public class CustomItemWriter implements ItemWriter<CustomerForMybatis> {
	private final CustomService customService;

	public CustomItemWriter(CustomService customService) {
		this.customService = customService;
	}

	@Override
	public void write(Chunk<? extends CustomerForMybatis> chunk) throws Exception {
		for (CustomerForMybatis customer: chunk) {
			log.info("Call Porcess in CustomItemWriter...");
			customService.processToOtherService(customer);
		}
	}
}

```

- Chunk는 Customer 객체를 한묶음으로 처리할수 있도록 반복 수행
- 따라서, for문으로 item을 처리
