---
title: 05 JdbcPagingItemReader로 DB내용을 읽고, JdbcBatchItemWriter로 DB에 쓰기
tags: SpringBatch
---
# 05 JdbcPagingItemReader로 DB내용을 읽고, JdbcBatchItemWriter로 DB에 쓰기

## JdbcPagingItemReader 개요

- JdbcPagingItemReader는 Spring Batch에서 제공하는 ItemReader로, DB로부터 데이터를 페이지 단위로 읽음
- 메모리 사용량을 최소화하고 커밋 간격을 설정하여 대규모 데이터를 효율적으로 처리 (대규모 데이터 처리 효율성)
- SQL 쿼리를 직접 작성하여 최적화된 데이터 읽기 가능 (쿼리 최적화)
- 데이터베이스 커서를 사용하여 데이터 순회를 제어 (커서 제어)

## **JdbcPagingItemReader 주요 구성 요소**

- DataSource : 데이터베이스 연결 정보를 설정
- SqlQuery : 데이터를 읽을 SQL 쿼리를 설정
- RowMapper : SQL 쿼리 결과를 Item으로 변환하는 역할
- PageSize : 페이지 크기를 설정
- SkippableItemReader : 오류 발생 시 해당 Item을 건너뛰도록 함
- ReadListener : 읽기 시작, 종료, 오류 발생 등의 이벤트를 처리
- SaveStateCallback : job 중단 시 현재 상태를 저장하여 재시작 시 이어서 처리

## JdbcPagingItemReader 실습

### Customer 클래스 생성

```java
@Getter
@Setter
public class Customer {
    private String name;
    private int age;
    private String gender;
}
```

### 쿼리 Provider 작성

```java
@Bean
public PagingQueryProvider queryProvider() throws Exception {
    SqlPagingQueryProviderFactoryBean queryProvider = new SqlPagingQueryProviderFactoryBean();
    queryProvider.setDataSource(dataSource);  // DB 에 맞는 PagingQueryProvider 를 선택하기 위함
    queryProvider.setSelectClause("id, name, age, gender");
    queryProvider.setFromClause("from customer");
    queryProvider.setWhereClause("where age >= :age");

    Map<String, Order> sortKeys = new HashMap<>(1);
    sortKeys.put("id", Order.DESCENDING);

    queryProvider.setSortKeys(sortKeys);

    return queryProvider.getObject();
}
```

- 쿼리 Provider는 실제 배치를 위해 데이터를 읽어올 쿼리를 작성
- SqlPagingQueryProviderFactoryBean : query provider factory
- setDataSource : 데이터소스를 설정
- setSelectClause : select에서 조회할 필드
- setFromClause : 조회할 테이블
- setWhereClause : 조건절
- setSortKeys : 정렬 key를 지정

### **JdbcPagingItemReader 작성**

```java
@Bean
public JdbcPagingItemReader<Customer> jdbcPagingItemReader() throws Exception {

    Map<String, Object> parameterValue = new HashMap<>();
    parameterValue.put("age", 20);

    return new JdbcPagingItemReaderBuilder<Customer>()
            .name("jdbcPagingItemReader")
            .fetchSize(CHUNK_SIZE)
            .dataSource(dataSource)
            .rowMapper(new BeanPropertyRowMapper<>(Customer.class))
            .queryProvider(queryProvider())
            .parameterValues(parameterValue)
            .build();
}
```

- fetchSize : query 실행으로 조회된 결과를 메모리에 가져올 row 수 (pageSize와 다를 수 있음)
- dataSource : 조회할 DB의 dataSource
- rowMapper : 조회한 결과를 객체에 매핑
- queryProvider : 실행할 query 지정
- parameterValues : query에 실행시 전달될 파라미터

## **JdbcBatchItemWriter 개요**

- JdbcBatchItemWriter Spring Batch에서 제공하는 ItemWriter 인터페이스를 구현한 클래스
- 데이터를 JDBC를 통해 DB에 저장하는데 사용

# **JdbcBatchItemWriter 구성 요소**

- DataSource : 데이터베이스 연결 정보를 지정
- SqlStatementCreator : 동적으로 query를 생성하는 역할

    ```java
    import org.springframework.batch.item.database.SqlParameterSourceProvider;
    import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
    import org.springframework.jdbc.core.namedparam.SqlParameterSource;

    public class CustomSqlStatementCreator implements SqlParameterSourceProvider<Customer> {

        @Override
        public SqlParameterSource createSqlSource(Customer customer) {
            MapSqlParameterSource parameterSource = new MapSqlParameterSource();
            parameterSource.addValue("id", customer.getId());
            parameterSource.addValue("name", customer.getName());
            parameterSource.addValue("age", customer.getAge());

            if (customer.isNew()) {
                // 새로운 고객이라면 INSERT SQL
                parameterSource.addValue("sql", "INSERT INTO customer (id, name, age) VALUES (:id, :name, :age)");
            } else {
                // 기존 고객이라면 UPDATE SQL
                parameterSource.addValue("sql", "UPDATE customer SET name = :name, age = :age WHERE id = :id");
            }

            return parameterSource;
        }
    }
    ```

    ```java
    @Bean
    public JdbcBatchItemWriter<Customer> customerItemWriter(DataSource dataSource) {
        NamedParameterJdbcTemplate namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(dataSource);

        return new JdbcBatchItemWriterBuilder<Customer>()
                .itemSqlParameterSourceProvider(new CustomSqlStatementCreator())
                .sql("PLACEHOLDER") // 실제 SQL은 CustomSqlStatementCreator에서 제공되므로 placeholder 사용
                .namedParametersJdbcTemplate(namedParameterJdbcTemplate)
                .build();
    }
    ```

- ItemPreparedStatementSetter : INSERT 쿼리의 파라미터 값을 설정하는 역할

    ```java
    public class CustomerPreparedStatementSetter implements ItemPreparedStatementSetter<Customer> {

        @Override
        public void setValues(Customer customer, PreparedStatement ps) throws SQLException {
            ps.setLong(1, customer.getId());
            ps.setString(2, customer.getName());
            ps.setInt(3, customer.getAge());
        }
    }
    ```

    ```java
    @Bean
    public JdbcBatchItemWriter<Customer> customerItemWriter(DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<Customer>()
                .dataSource(dataSource)
                .sql("INSERT INTO customer (id, name, age) VALUES (?, ?, ?)")
                .itemPreparedStatementSetter(new CustomerPreparedStatementSetter())
                .build();
    }
    ```

- ItemSqlParameterSourceProvider: Item 객체를 기반으로 PreparedStatementSetter에 전달할 파라미터 값을 생성하는 역할

### 장점

- 데이터베이스 연동 : JDBC를 통해 다양한 DB에 데이터를 저장
- 성능 : 대량의 데이터를 빠르게 저장
- 유연성 : 다양한 설정을 통해 원하는 방식으로 데이터를 저장

### 단점

- 설정 복잡성 : JDBC 설정 및 쿼리 작성이 복잡할 수 있음
- DB 종속 : 특정 DB에 종속적
- 오류 가능성 : 설정 오류 시 데이터 손상 가능성 있음

## JdbcBatchItemWriter 샘플 코드

### application.properties

```java
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.url=jdbc:h2:file:~/testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=test
spring.datasource.password=test
```

### 테이블 생성

```sql
create table CUSTOMER2
(
    ID     INTEGER auto_increment,
    NAME   CHARACTER VARYING(100),
    AGE    INTEGER,
    GENDER CHARACTER VARYING(10)
);
```

### **JdbcBatchItemWriter 작성**

```java
@Bean
public JdbcBatchItemWriter<Customer> flatFileItemWriter() {
    return new JdbcBatchItemWriterBuilder<Customer>()
            .dataSource(dataSource)
            .sql("INSERT INTO customer2 (name, age, gender) VALUES (:name, :age, :gender)")
            .itemSqlParameterSourceProvider(new CustomerItemSqlParameterSourceProvider())
            .build();
}
```

### **SqlPatameterSourceProvider 작성**

```java
public class CustomerItemSqlParameterSourceProvider implements ItemSqlParameterSourceProvider<Customer> {
    @Override
    public SqlParameterSource createSqlParameterSource(Customer item) {
        return new BeanPropertySqlParameterSource(item);
    }
}
```
