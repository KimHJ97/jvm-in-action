# 어노테이션 프로세서

annotationProcessor는 컴파일 시점에 특정 어노테이션을 처리하여 새로운 코드를 자동으로 생성하는 기능을 제공하는 Gradle의 설정입니다.
이는 Java의 어노테이션 프로세싱(annotation processing) 기능을 활용하여 동작하며, annotationProcessor로 등록된 프로세서는 소스 코드(Java 파일)를 분석하고 새로운 Java 코드를 생성할 수 있습니다.

```groovy
dependencies {
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'

    implementation 'com.google.dagger:dagger:2.44'
    annotationProcessor 'com.google.dagger:dagger-compiler:2.44'

    implementation 'org.projectlombok:lombok:1.18.30'
    annotationProcessor 'org.projectlombok:lombok-processor:1.18.30'

    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jakarta"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
}
```

## annotationProcessor의 동작 방식

annotationProcessor는 컴파일러의 어노테이션 프로세서를 실행하여 Java 소스 코드(Java 파일)를 기반으로 새로운 코드를 생성합니다.

annotationProcessor는 Java 파일(.java)을 분석하여 새로운 .java 파일을 생성하고, 이후에 일반적인 컴파일 과정을 거쳐 .class 파일이 생성됩니다.

즉, Java 파일을 기반으로 어노테이션 프로세싱이 수행되고, 그 결과로 생성된 파일이 다시 컴파일되는 방식입니다.

 - Java 파일(.java)을 컴파일러가 읽음
 - 등록된 어노테이션 프로세서(annotation processor)가 어노테이션을 분석
 - 새로운 Java 파일(.java)을 자동 생성 (Generated 등의 폴더에 저장됨)
 - 새로 생성된 Java 파일을 포함하여 컴파일 진행
 - 최종적으로 .class 파일이 생성됨


## annotationProcessor 사용 시 주의할 점

 - Java 파일이 있어야 동작함
    - 기존 .class 파일만 있는 경우 어노테이션 프로세서는 동작하지 않음
 - 새로운 Java 파일을 생성할 수 있음
    - 생성된 코드를 프로젝트에 포함시키려면 build/generated 디렉토리를 확인해야 함

## QueryDSL 모듈

만약 멀티 모듈 프로젝트에서 Entity만 존재하는 모듈이 있고, Persistence 모듈에서 DB 접근 기술 구현체로 QueryDSL 모듈이 있다고 가정한다.

QueryDSL을 이용하려면 엔티티 클래스를 기반으로 AnnotationProcessor가 동작하여 Q Class를 만들어야 한다. AnnotationProcessor는 Java 파일의 어노테이션을 읽어 동작한다. 때문에, Entity 모듈을 의존성으로 받았을 때 사용되는 class 파일 정보로는 Q Class를 만들 수 없다.

즉, Entity 모듈에 QueryDSL 의존성을 추가하여 Q Class를 생성하도록 한다. QueryDSL 모듈에서 Entity 모듈에서 generate된 Q Class를 sourceSets으로 사용할 수도 있고, Entity 모듈에서 Q Class를 포함하여 빌드할 수도 있다.

 - Entity 모듈에서 Q Class를 생성하고, Q Class를 sourceSets 옵션으로 추가하여 빌드한다.
 - QueryDSL 모듈에서 Q Class와 Entity 클래스가 포함된 모듈을 사용한다.
```groovy
plugins {
    id 'java-library'
}

dependencies {
    implementation project(':commons:commons-domain')

    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jakarta"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
}

sourceSets {
    main {
        java {
            srcDirs("$buildDir/generated/sources/annotationProcessor/java/main") // QClass 경로 추가
        }
    }
}
```
