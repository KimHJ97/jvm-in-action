# MyBatis 테스트 코드

MyBatis를 이용한 애플리케이션에서 테스트 코드를 작성하기 위해서는 JPA와 다르게 번거로운 작업을 해야한다. JPA의 경우 테이블과 매칭되는 엔티티 클래스를 작성하면서 DDL-AUTO 옵션으로 테이블을 자동으로 생성할 수 있으며, 기본적으로 CRUD 기능을 제공하여 각 테스트마다 격리된 환경을 위해 테스트 실행시 초기 데이터를 등록(INSERT) 하거나, 테스트가 종료된 후에 정리(DELETE)를 쉽게 할 수 있다.  
MyBatis는 JPA와 다르게 CRUD SQL을 직접 정의해주어야 하며, 자동으로 테이블을 생성해주는 DDL-AUTO 옵션이 존재하지 않는다. 때문에, 별도의 DDL SQL Script를 만들어서 테스트 실행시에 테이블을 초기화하도록 직접 정의해주어야 한다. 또, 초기 데이터 생성과 정리를 위해 필요한 CRUD SQL을 정의해주어야 한다. 이때, 정리의 경우 Spring 테스트 환경 특성을 알고 있다면 @Transactional 어노테이션을 이용할 수도 있다.  

## 테스트 코드 @Transactional

Controller나 Service 레이어에서 동작 테스트를 위해 DB 데이터를 설정하게 된다. 이렇게 테스트에 사용된 데이터들이 기본적으로 실제로 동작하여 DB에 남게 된다. 이러한 문제를 해결하기 위해 Spring 테스트 코드 환경에서 @Transactional 어노테이션을 정의하면 별도의 코드 없이 테스트 케이스 종료 후 트랜잭션을 롤백하게 되어 편리하게 테스트 환경을 격리할 수 있다. 하지만, 몇 가지 예외 상황이 있을 수 있다.  
 - __@Transactional 주의사항__
    - 참고 블로그: https://jojoldu.tistory.com/761
 - __롤백이 되지 않는 상황__
    - @Transactional(propagation = Propagation.REQUIRES_NEW) 옵션이 적용된 메서드는 별도의 트랜잭션에서 작업된다. 때문에, 해당 트랜잭션은 종료 후에 이미 커밋이 되기 때문에 롤백되지 않는다.
    - 비동기 메서드에서의 DB 작업도 롤백되지 않는다.

## MyBatis 테스트 코드 - 준비

 - `test/java/resources/schema.sql`
    - 테이블을 생성하기 위한 DDL SQL을 정의한다.
    - 실제로 사용하는 DB에 없는 함수를 정의한다. H2 DB는 Java로 만들어진 인메모리 DB로 Java 문법으로 함수를 정의할 수 있다.
    - 필요한 경우 초기 데이터를 설정한다. 공통 코드나 초기 설정이 필요한 데이터를 정의해준다. 다만, 테스트 환경이 격리될 수 있도록 유의해야 한다.
```sql
-- DDL 정의
CREATE TABLE IF NOT EXISTS member (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200),
    email VARCHAR(200),
    birthday TIMESTAMP()
    createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 함수 정의
CREATE ALIAS IF NOT EXISTS ADD_NUMBERS AS $$
    int addNumbers(int a, int b) {
        return a + b;
    }
$$;

CREATE ALIAS IF NOT EXISTS DATE_FORMAT AS $$
    String dateFormat(String dateTime, String format) {
        if (dateTime == null || format == null) {
            return null;
        }
        java.time.LocalDateTime localDateTime = java.time.LocalDateTime.parse(dateTime.replace(" ", "T"));
        String javaFormat = format.replace("%Y", "yyyy")
                                  .replace("%m", "MM")
                                  .replace("%d", "dd")
                                  .replace("%H", "HH")
                                  .replace("%i", "mm")
                                  .replace("%s", "ss");
        java.time.format.DateTimeFormatter formatter = java.time.format.DateTimeFormatter.ofPattern(javaFormat);
        return localDateTime.format(formatter);
    }
$$;

-- 초기 데이터 정의
MERGE INTO commonGroupCode (groupCode, name)
KEY (groupCode, name)
VALUES
    ('MEMBER_ROLE', '회원 역할');

MERGE INTO commonCode (groupCode, code, message)
KEY (groupCode, code, message)
VALUES
    ('MEMBER_ROLE', 'ADMIN', '관리자'),
    ('MEMBER_ROLE', 'USER', '사용자');
```

 - `test/java/resources/application.yml`
    - 테스트 코드 실행시 'test/java/resources/' 폴더에 application.yml이 존재하면 해당 파일을 사용한다. 테스트 코드에 필요한 설정 값을 지정하여 사용한다.
    - MyBatis를 사용하기 위한 H2 DB를 설정한다.
        - 'spring.jpa.hibernate.ddl-auto' 옵션을 사용할 수 없다.
```yml
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;INIT=RUNSCRIPT from 'classpath:schema.sql'
    driverClassName: org.h2.Driver
    username: sa
    password:
```

## MyBatis 테스트 코드 - 작성

### 통합 테스트
