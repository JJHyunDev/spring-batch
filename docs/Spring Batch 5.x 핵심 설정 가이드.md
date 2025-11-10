# Spring Batch 5.x (Spring Boot 3.x) 핵심 설정 가이드

Spring Boot 3.x와 함께 Spring Batch 5.x 버전을 사용하면서 변경된 주요 설정 방식과 주의사항을 정리합니다. 이 가이드를 통해 더 명확하고 안정적인 배치 애플리케이션을 구성할 수 있습니다.

---

## 1. 새로운 Job/Step 설정 방식 (Factory 클래스 대체)

Spring Batch 5.x부터 `JobBuilderFactory`와 `StepBuilderFactory`는 더 이상 권장되지 않으며(**Deprecated**), `JobRepository`와 `PlatformTransactionManager`를 직접 주입받아 `JobBuilder`와 `StepBuilder`를 생성하는 방식이 표준이 되었습니다.

### 왜 변경되었나요?

기존 Factory 방식은 편리했지만, 내부 동작이 감춰져 있어 명시적이지 않았습니다. 새로운 방식은 Job과 Step 구성에 필요한 의존성을 개발자가 직접 명시적으로 주입하여 코드의 명확성을 높이고, 내부 동작을 이해하기 쉬우며, 단위 테스트에 더 용이한 구조를 제공합니다.

### 설정 방법

설정 클래스(`@Configuration`)에서 생성자를 통해 `JobRepository`와 `PlatformTransactionManager`를 주입받습니다.

```java
package com.example.springbatch;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Slf4j
@RequiredArgsConstructor
@Configuration
public class HelloJobConfiguration {

    // 1. 생성자를 통해 핵심 컴포넌트를 직접 주입받습니다.
    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;

    @Bean
    public Job myJob() {
        // 2. JobBuilder를 'new' 키워드로 직접 생성하고, JobRepository를 전달합니다.
        return new JobBuilder("myJob", jobRepository)
                .start(helloStep1())
                .next(helloStep2())
                .build();
    }

    @Bean
    public Step helloStep1() {
        // 3. StepBuilder도 직접 생성하며, JobRepository와 TransactionManager를 명시적으로 전달합니다.
        return new StepBuilder("helloStep1", jobRepository)
                .tasklet((contribution, chunkContext) -> {
                    log.info(" >> Hello, Spring Batch 1!");
                    return RepeatStatus.FINISHED;
                }, transactionManager)
                .build();
    }

    @Bean
    public Step helloStep2() {
        return new StepBuilder("helloStep2", jobRepository)
                .tasklet((contribution, chunkContext) -> {
                    log.info(" >> Hello, Spring Batch 2!");
                    return RepeatStatus.FINISHED;
                }, transactionManager)
                .build();
    }
}
```

---

## 2. `@EnableBatchProcessing` 사용 중단 및 Spring Boot 자동 설정

Spring Boot 2.x 버전 이후부터, `@EnableBatchProcessing` 어노테이션은 **사용하지 않는 것이 좋습니다.**

### 왜 사용하지 않아야 하나요?

`build.gradle`에 `spring-boot-starter-batch` 의존성이 포함되어 있으면, Spring Boot의 **자동 설정(Auto-configuration)** 기능이 `@EnableBatchProcessing`이 하던 모든 역할을 대신 수행합니다.

만약 `@EnableBatchProcessing`을 함께 사용하면, **배치 설정이 중복으로 적용**되어 다음과 같은 문제가 발생할 수 있습니다.

*   **빈(Bean) 충돌**: `JobLauncher`, `JobRepository` 등 핵심 빈이 중복으로 생성되어 설정이 꼬이거나 예기치 않은 오류가 발생할 수 있습니다.
*   **자동 실행 메커니즘 방해**: Spring Boot가 제공하는 Job 자동 실행 기능(`JobLauncherApplicationRunner`)이 정상적으로 동작하지 않을 수 있습니다. (이것이 바로 Job 로그가 출력되지 않았던 원인입니다.)

### 올바른 설정 방법

`@SpringBootApplication` 어노테이션만 남겨두고 `@EnableBatchProcessing`은 삭제합니다. Spring Boot가 모든 것을 알아서 처리해 줄 것입니다.

#### 잘못된 예 (Before)

```java
package com.example.springbatch;

import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableBatchProcessing // <-- 이 어노테이션이 문제를 일으킬 수 있습니다.
public class SpringBatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBatchApplication.class, args);
    }
}
```

#### 올바른 예 (After)

```java
package com.example.springbatch;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// @EnableBatchProcessing를 삭제하여 Spring Boot 자동 설정만 사용합니다.
@SpringBootApplication
public class SpringBatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBatchApplication.class, args);
    }
}
```

---

## 요약 및 권장 사항

1.  **Job/Step 설정**: `JobBuilderFactory` 대신 `JobRepository`를 직접 주입받아 `new JobBuilder(...)` 방식으로 구성합니다.
2.  **`@EnableBatchProcessing`**: Spring Boot 환경에서는 **반드시 삭제**하고 Spring Boot의 자동 설정을 활용합니다.
3.  **Job 재실행**: 개발 중 동일한 Job을 반복 테스트하려면, 매번 다른 `JobParameters`를 전달하여 새로운 `JobInstance`를 생성해야 합니다. `System.currentTimeMillis()`를 파라미터로 활용하는 것이 좋은 방법입니다.