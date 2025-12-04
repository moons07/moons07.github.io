---
title: 07 MyBatisPagingItemReader로 DB내용을 읽고, MyBatisItemWriter로 DB에 쓰기
tags: SpringBatch
---
# 07 MyBatisPagingItemReader로 DB내용을 읽고, MyBatisItemWriter로 DB에 쓰기

## **MyBatisItemReader**

- MyBatisPagingItemReader Spring Mybatis에서 제공하는 ItemReader 인터페이스를 구현하는 클래스

### 장단점

#### 장점

- 간편한 설정 : MyBatis 쿼리 매퍼를 직접 활용하여 데이터를 읽을 수 있어 설정이 간편
- 쿼리 최적화 : MyBatis의 다양한 기능을 활용하여 최적화된 쿼리를 작성할
- 동적 쿼리 지원 : 런타임 시 조건에 따라 동적으로 쿼리를 생성

#### 단점

- MyBatis 의존성 : MyBatis 라이브러리에 의존
- 커스터마이징 복잡 : Chunk-oriented Processing 방식과 비교했을 때 커스터마이징이 더 복잡

### 주요 구성 요소

- SqlSessionFactory : MyBatis 설정 정보 및 SQL 쿼리 매퍼 정보를 담고 있는 객체
- QueryId : 데이터를 읽을 MyBatis 쿼리 ID
- PageSize : 페이징 쿼리를 위한 페이지 크기를 지정

#### **SqlSessionFactory**

- MyBatisPagingItemReader SqlSessionFactory 객체를 통해 MyBatis와 연동
- SqlSessionFactory는 다음과 같은 방법으로 설정
- @Bean 어노테이션을 사용하여 직접 생성
- Spring Batch XML 설정 파일에서 설정
- Java 코드에서 직접 설정

#### **QueryId**

- MyBatisPagingItemReader setQueryId() 메소드를 통해 데이터를 읽을 MyBatis 쿼리 ID를 설정
- 쿼리 ID는 com.example.mapper.CustomerMapper.selectCustomers 와 같은 형식으로 지정

#### **PageSize**

- MyBatisItemReader는 pageSize를 이용하여 offset, limit 을 이용하는 기준을 설정

#### **추가 구성 요소**

- SkippableItemReader : 오류 발생 시 해당 Item을 건너뛸 수 있도록 함
- ReadListener : 읽기 시작, 종료, 오류 발생 등의 이벤트를 처리
- SaveStateCallback : job의 중단 시 현재 상태를 저장하여 재시작 시 이어서 처리

## **MyBatisItemWriter**

- MyBatisBatchItemWriter Spring Batch에서 제공하는 ItemWriter 인터페이스를 구현하는 클래스
- 데이터를 MyBatis를 통해 데이터베이스에 저장하는데 사용

### 구성요소

- SqlSessionTemplate : MyBatis SqlSession 생성 및 관리를 위한 템플릿 객체
- SqlSessionFactory : SqlSessionTemplate 생성을 위한 팩토리 객체
- StatementId : 실행할 MyBatis SQL 맵퍼의 스테이tement ID
- ItemToParameterConverter : 객체를 ParameterMap으로 변경

### 장단점

#### 장점

- ORM 연동 : MyBatis를 통해 다양한 데이터베이스에 데이터를 저장
- SQL 쿼리 분리 : SQL 쿼리를 Java 코드로부터 분리하여 관리 및 유지 보수가 용이
- 유연성 : 다양한 설정을 통해 원하는 방식으로 데이터를 저장

#### 단점

- 설정 복잡성 : MyBatis 설정 및 SQL 맵퍼 작성이 복잡
- 데이터베이스 종속 : 특정 데이터베이스에 종속적
- 오류 가능성 : 설정 오류 시 데이터 손상 가능성
