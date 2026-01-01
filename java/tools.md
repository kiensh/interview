# DevOps & Tools Interview Questions & Answers

---

## Table of Contents

### [Build Tools](#build-tools)
- [Q1: What is the difference between Maven and Gradle?](#q1)
- [Q2: How does Gradle manage dependencies?](#q2)
- [Q3: What are Gradle plugins?](#q3)
- [Q4: How do you configure a Spring Boot project with Gradle?](#q4)

### [Logging Frameworks](#logging-frameworks)
- [Q5: What is the difference between Log4j, SLF4J, and Logback?](#q5)
- [Q6: How do you configure logging in Spring Boot?](#q6)
- [Q7: What are log levels?](#q7)
- [Q8: What is centralized logging?](#q8)

### [Task Scheduling](#task-scheduling)
- [Q9: How do you schedule tasks in Spring?](#q9)
- [Q10: How do you handle cron jobs with multiple service instances?](#q10)

---

## Build Tools

<a id="q1"></a>
### Q1: What is the difference between Maven and Gradle?
**Answer:**

| Feature | Maven | Gradle |
|---------|-------|--------|
| Configuration | XML (pom.xml) | Groovy/Kotlin DSL |
| Build model | Fixed lifecycle | Task-based |
| Performance | Slower | Faster (incremental builds) |
| Flexibility | Convention-based | Highly customizable |
| Learning curve | Easier | Steeper |
| Caching | Local only | Build cache (local/remote) |

**Maven (pom.xml):**
```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>3.2.0</version>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**Gradle (build.gradle):**
```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
}

group = 'com.example'
version = '1.0.0'

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

<a id="q2"></a>
### Q2: How does Gradle manage dependencies?
**Answer:**

**Dependency configurations:**
| Configuration | Purpose |
|--------------|---------|
| `implementation` | Compile + runtime, not exposed to consumers |
| `api` | Compile + runtime, exposed to consumers |
| `compileOnly` | Compile only (e.g., Lombok) |
| `runtimeOnly` | Runtime only (e.g., JDBC drivers) |
| `testImplementation` | Test compile + runtime |
| `annotationProcessor` | Annotation processing |

```groovy
dependencies {
    // Standard dependency
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    // With version
    implementation 'com.google.guava:guava:32.0.0-jre'
    
    // Compile-only (not in runtime)
    compileOnly 'org.projectlombok:lombok:1.18.30'
    annotationProcessor 'org.projectlombok:lombok:1.18.30'
    
    // Runtime only (e.g., database drivers)
    runtimeOnly 'org.postgresql:postgresql:42.6.0'
    
    // Test dependencies
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    
    // Exclude transitive dependency
    implementation('org.springframework.boot:spring-boot-starter-web') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
    }
    
    // Project dependency
    implementation project(':common-module')
    
    // Platform (BOM)
    implementation platform('org.springframework.boot:spring-boot-dependencies:3.2.0')
}
```

**Dependency resolution:**
```groovy
configurations.all {
    // Force version
    resolutionStrategy {
        force 'com.google.guava:guava:32.0.0-jre'
        
        // Fail on version conflict
        failOnVersionConflict()
        
        // Cache dynamic versions
        cacheDynamicVersionsFor 10, 'minutes'
    }
}

// View dependency tree
// ./gradlew dependencies
```

<a id="q3"></a>
### Q3: What are Gradle plugins?
**Answer:**
Plugins add new tasks, configurations, and conventions to your build.

**Types:**
| Type | Description |
|------|-------------|
| Core plugins | Bundled with Gradle (java, application) |
| Community plugins | From Gradle Plugin Portal |
| Custom plugins | Your own plugins |

```groovy
plugins {
    // Core plugins (no version needed)
    id 'java'
    id 'application'
    
    // Community plugins (from plugin portal)
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    
    // Apply without version (defined in settings.gradle)
    id 'com.company.custom-plugin'
}

// Legacy apply syntax
apply plugin: 'java'

// Configure plugin
java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

springBoot {
    buildInfo()
}

application {
    mainClass = 'com.example.Application'
}
```

**Common plugins:**
```groovy
plugins {
    id 'java'                          // Java compilation
    id 'java-library'                  // Library with API/implementation
    id 'application'                   // Executable application
    id 'org.springframework.boot'      // Spring Boot
    id 'io.spring.dependency-management'  // BOM management
    id 'jacoco'                        // Code coverage
    id 'checkstyle'                    // Code style
    id 'com.google.protobuf'           // Protocol Buffers
    id 'org.flywaydb.flyway'           // Database migrations
}
```

<a id="q4"></a>
### Q4: How do you configure a Spring Boot project with Gradle?
**Answer:**

**build.gradle:**
```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot starters
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    
    // Database
    runtimeOnly 'org.postgresql:postgresql'
    
    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    // Testing
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.testcontainers:postgresql'
}

tasks.named('test') {
    useJUnitPlatform()
}

// Custom tasks
tasks.register('printVersion') {
    doLast {
        println "Version: ${project.version}"
    }
}

// Bootjar configuration
bootJar {
    archiveFileName = "${project.name}.jar"
    layered {
        enabled = true
    }
}
```

**settings.gradle:**
```groovy
rootProject.name = 'my-application'

// Multi-module project
include 'api'
include 'core'
include 'common'

// Plugin management
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}
```

**Common commands:**
```bash
# Build
./gradlew build

# Run
./gradlew bootRun

# Test
./gradlew test

# Clean
./gradlew clean

# Build without tests
./gradlew build -x test

# List tasks
./gradlew tasks

# Dependencies
./gradlew dependencies

# Build info
./gradlew bootBuildInfo
```

---

## Logging Frameworks

<a id="q5"></a>
### Q5: What is the difference between Log4j, SLF4J, and Logback?
**Answer:**

| Framework | Type | Description |
|-----------|------|-------------|
| **SLF4J** | Facade/API | Abstraction layer, not implementation |
| **Logback** | Implementation | Default in Spring Boot, implements SLF4J |
| **Log4j2** | Implementation | Apache alternative to Logback |

```
Application Code
      │
      ▼
┌─────────────┐
│   SLF4J     │  ← Logging facade (interface)
│   (API)     │
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  Logback    │  ← Logging implementation
│  (or Log4j2)│
└─────────────┘
```

```java
// Using SLF4J (recommended)
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyService {
    private static final Logger log = LoggerFactory.getLogger(MyService.class);
    
    public void process() {
        log.info("Processing started");
        log.debug("Debug details: {}", data);
        log.error("Error occurred", exception);
    }
}

// Or with Lombok
@Slf4j
public class MyService {
    public void process() {
        log.info("Processing");
    }
}
```

<a id="q6"></a>
### Q6: How do you configure logging in Spring Boot?
**Answer:**

**application.yml:**
```yaml
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.springframework: WARN
    org.hibernate.SQL: DEBUG
  
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  
  file:
    name: logs/application.log
    max-size: 10MB
    max-history: 30
```

**logback-spring.xml (advanced):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <!-- Console appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- Rolling file appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxHistory>30</maxHistory>
            <maxFileSize>10MB</maxFileSize>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- JSON appender for centralized logging -->
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>correlationId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
        </encoder>
    </appender>
    
    <!-- Profile-specific configuration -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
    
    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="JSON"/>
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
    
    <!-- Package-specific levels -->
    <logger name="com.example.service" level="DEBUG"/>
    <logger name="org.springframework" level="WARN"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
</configuration>
```

<a id="q7"></a>
### Q7: What are log levels?
**Answer:**

| Level | Priority | Use Case |
|-------|----------|----------|
| TRACE | Lowest | Fine-grained debugging |
| DEBUG | Low | Development debugging |
| INFO | Medium | General information |
| WARN | High | Potential problems |
| ERROR | Highest | Errors and exceptions |

```java
@Slf4j
public class LoggingExample {
    
    public void demonstrateLevels() {
        log.trace("Trace: Very detailed, rarely enabled");
        log.debug("Debug: Development info, {}", debugData);
        log.info("Info: Important runtime events");
        log.warn("Warn: Potential issues");
        log.error("Error: Actual errors", exception);
    }
    
    public void process(Order order) {
        log.info("Processing order: {}", order.getId());
        
        try {
            // Process
            log.debug("Order details: {}", order);
        } catch (Exception e) {
            log.error("Failed to process order {}: {}", order.getId(), e.getMessage(), e);
        }
    }
}
```

**Best practices:**
```java
// DO: Use parameterized logging (lazy evaluation)
log.debug("User {} logged in from {}", userId, ipAddress);

// DON'T: String concatenation (always evaluated)
log.debug("User " + userId + " logged in from " + ipAddress);

// DO: Include relevant context
log.error("Payment failed for order {}", orderId, exception);

// DO: Use MDC for correlation
MDC.put("correlationId", correlationId);
MDC.put("userId", userId);
try {
    // All logs in this context will include correlationId and userId
    log.info("Processing request");
} finally {
    MDC.clear();
}
```

<a id="q8"></a>
### Q8: What is centralized logging?
**Answer:**
Centralized logging aggregates logs from multiple services into a single location for analysis.

**Common solutions:**
| Solution | Components |
|----------|------------|
| ELK Stack | Elasticsearch, Logstash, Kibana |
| EFK Stack | Elasticsearch, Fluentd, Kibana |
| Graylog | Graylog, Elasticsearch, MongoDB |
| CloudWatch | AWS native logging |
| Splunk | Commercial solution |

**ELK Stack:**
```
┌─────────┐  ┌─────────┐  ┌─────────┐
│Service A│  │Service B│  │Service C│
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  │
                  ▼
           ┌──────────┐
           │ Logstash │  (Collection & Processing)
           └────┬─────┘
                │
                ▼
         ┌─────────────┐
         │Elasticsearch│  (Storage & Search)
         └──────┬──────┘
                │
                ▼
           ┌────────┐
           │ Kibana │  (Visualization)
           └────────┘
```

**JSON logging for centralized logging:**
```xml
<!-- logback-spring.xml -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <customFields>{"service":"order-service"}</customFields>
    </encoder>
</appender>
```

**Output:**
```json
{
  "@timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Order created",
  "service": "order-service",
  "correlationId": "abc-123",
  "orderId": "ORD-456"
}
```

---

## Task Scheduling

<a id="q9"></a>
### Q9: How do you schedule tasks in Spring?
**Answer:**

**Enable scheduling:**
```java
@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**Scheduling methods:**
```java
@Component
public class ScheduledTasks {
    
    // Fixed rate - runs every 5 seconds
    @Scheduled(fixedRate = 5000)
    public void fixedRateTask() {
        log.info("Fixed rate task");
    }
    
    // Fixed delay - 5 seconds after last completion
    @Scheduled(fixedDelay = 5000)
    public void fixedDelayTask() {
        log.info("Fixed delay task");
    }
    
    // Initial delay before first execution
    @Scheduled(fixedRate = 5000, initialDelay = 10000)
    public void withInitialDelay() {
        log.info("Started after 10s, then every 5s");
    }
    
    // Cron expression
    @Scheduled(cron = "0 0 8 * * MON-FRI")  // 8 AM weekdays
    public void cronTask() {
        log.info("Weekday morning task");
    }
    
    // Configurable via properties
    @Scheduled(cron = "${app.scheduler.report.cron}")
    public void configurableTask() {
        log.info("Configurable task");
    }
}
```

**Cron expression format:**
```
┌───────────── second (0-59)
│ ┌───────────── minute (0-59)
│ │ ┌───────────── hour (0-23)
│ │ │ ┌───────────── day of month (1-31)
│ │ │ │ ┌───────────── month (1-12)
│ │ │ │ │ ┌───────────── day of week (0-7, 0 or 7 = Sunday)
│ │ │ │ │ │
* * * * * *

Examples:
"0 0 * * * *"      - Every hour
"0 0 8 * * *"      - Every day at 8 AM
"0 0 8 * * MON-FRI" - Weekdays at 8 AM
"0 */30 * * * *"   - Every 30 minutes
"0 0 0 1 * *"      - First day of month at midnight
```

<a id="q10"></a>
### Q10: How do you handle cron jobs with multiple service instances?
**Answer:**
When running multiple instances, you need to ensure scheduled tasks run only once.

**Solution 1: ShedLock (database-based locking)**
```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class SchedulerConfig {
    
    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime()
                .build()
        );
    }
}

@Component
public class ScheduledTasks {
    
    @Scheduled(cron = "0 0 * * * *")
    @SchedulerLock(name = "hourlyTask", 
                   lockAtLeastFor = "5m",      // Hold lock at least 5 min
                   lockAtMostFor = "10m")       // Release after max 10 min
    public void hourlyTask() {
        // Only one instance executes this
        log.info("Running hourly task");
    }
}
```

**Database table for ShedLock:**
```sql
CREATE TABLE shedlock (
    name VARCHAR(64) NOT NULL,
    lock_until TIMESTAMP NOT NULL,
    locked_at TIMESTAMP NOT NULL,
    locked_by VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

**Solution 2: Leader election (Kubernetes)**
```java
@Component
public class LeaderAwareScheduler {
    
    @Autowired
    private LeaderElection leaderElection;
    
    @Scheduled(fixedRate = 60000)
    public void scheduledTask() {
        if (leaderElection.isLeader()) {
            // Only leader executes
            processTask();
        }
    }
}
```

**Solution 3: Redis-based distributed lock**
```java
@Component
public class RedisScheduledTasks {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Scheduled(fixedRate = 60000)
    public void distributedTask() {
        String lockKey = "scheduler:hourlyTask";
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "locked", Duration.ofMinutes(5));
        
        if (Boolean.TRUE.equals(acquired)) {
            try {
                // Execute task
                processTask();
            } finally {
                redisTemplate.delete(lockKey);
            }
        }
    }
}
```

**Solution 4: External scheduler (dedicated scheduling service)**
- Kubernetes CronJob
- AWS EventBridge + Lambda
- Apache Airflow
- Quartz Scheduler (clustered mode)

```yaml
# Kubernetes CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hourly-report
spec:
  schedule: "0 * * * *"
  concurrencyPolicy: Forbid  # Don't run concurrent jobs
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report-generator
            image: my-app:latest
            command: ["java", "-jar", "app.jar", "--generate-report"]
          restartPolicy: OnFailure
```

---

[← Back to Java Index](README.md)
