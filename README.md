# edu-logging-java

[Spring Boot Logging](https://docs.spring.io/spring-boot/reference/features/logging.html)
[OpenSearch-ELK alternative](https://opensearch.org)
[Logit-ELK host](https://logit.io/pricing/logs/)
[Logback](https://github.com/qos-ch/logback)
[Logack Tutorial Baeldung](https://www.baeldung.com/logback)
[ELK Tutorial](https://www.elastic.co/blog/getting-started-with-the-elastic-stack-and-docker-compose?utm_source=chatgpt.com)


# Spring Boot Logging & ELK – 1 Week Intern Training Plan

## w1-d1 — Logging basics (Java → SLF4J → Logback)
**Goal:** understand logging fundamentals.

**Tasks**
- Create a tiny Java app using SLF4J API and Logback.
- Practice levels, parameterized messages, MDC.

**Deliverable**
- Short note explaining facade (SLF4J) vs backend (Logback).

---

## w1-d2 — Spring Boot + Gradle setup
**build.gradle**
```gradle
plugins {
  id 'org.springframework.boot' version '3.3.4'
  id 'io.spring.dependency-management' version '1.1.6'
  id 'java'
}
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }
```

Create a REST controller with `INFO`, `DEBUG`, `ERROR` logs.

Run profiles:
```bash
./gradlew bootRun --args='--spring.profiles.active=dev'
SPRING_PROFILES_ACTIVE=prod ./gradlew bootRun
```

---

## w1-d3 — Profiles via properties

**application.properties**
```properties
spring.application.name=demo-logging
logging.level.root=INFO
logging.include-application-name=true
logging.file.name=logs/app.log
logging.logback.rollingpolicy.max-file-size=10MB
logging.logback.rollingpolicy.max-history=7
logging.logback.rollingpolicy.total-size-cap=1GB
```

**application-dev.properties**
```properties
logging.level.com.example=DEBUG
logging.file.name=logs/app-dev.log
```

**application-stage.properties**
```properties
logging.level.com.example=INFO
logging.file.name=logs/app-stage.log
```

**application-prod.properties**
```properties
logging.level.com.example=INFO
logging.file.name=/var/log/myapp/app-prod.log
logging.logback.rollingpolicy.max-file-size=50MB
logging.logback.rollingpolicy.max-history=14
logging.logback.rollingpolicy.total-size-cap=5GB
```

---

## w1-d4 — logback.xml

**src/main/resources/logback.xml**
```xml
<configuration>
  <property name="LOG_FILE" value="${LOG_FILE:-logs/app.log}" />
  <property name="PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n" />

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>${PATTERN}</pattern>
      <charset>UTF-8</charset>
    </encoder>
  </appender>

  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_FILE}</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
      <maxFileSize>${MAX_FILE_SIZE:-10MB}</maxFileSize>
      <maxHistory>${MAX_HISTORY:-7}</maxHistory>
      <totalSizeCap>${TOTAL_SIZE_CAP:-1GB}</totalSizeCap>
    </rollingPolicy>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
  </appender>

  <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>8192</queueSize>
    <neverBlock>true</neverBlock>
    <appender-ref ref="FILE"/>
  </appender>

  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="ASYNC_FILE"/>
  </root>
</configuration>
```

---

## w1-d5 — Structured logs
- Add MDC keys (e.g., requestId, userId).
- Produce `INFO/ERROR` logs and verify JSON output.

**Deliverable:** JSON file with MDC fields.

---

## w1-d6 — ELK in Docker

**docker-compose.yml**
```yaml
version: "3.8"
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.14.3
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ulimits: { memlock: { soft: -1, hard: -1 } }
    ports: ["9200:9200"]
    volumes: ["esdata:/usr/share/elasticsearch/data"]

  kibana:
    image: docker.elastic.co/kibana/kibana:8.14.3
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on: [elasticsearch]
    ports: ["5601:5601"]

  logstash:
    image: docker.elastic.co/logstash/logstash:8.14.3
    depends_on: [elasticsearch]
    ports: ["5044:5044", "8080:8080"]
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logs:/logs:ro

volumes: { esdata: {} }
```

**logstash/pipeline/logstash.conf**
```conf
input {
  file {
    path => "/logs/*.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => "json"
  }
}
filter {
  mutate { add_field => { "env" => "dev" "app" => "demo-logging" } }
}
output {
  elasticsearch {
    hosts  => ["http://elasticsearch:9200"]
    index  => "app-logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => json }
}
```

Run:
```bash
docker compose up -d
sudo sysctl -w vm.max_map_count=262144  # Linux only if ES fails
```

Access Kibana: [http://localhost:5601](http://localhost:5601)

---

## w1-d7 — Kibana queries & dashboards
- In *Discover*: search `level: "ERROR"`.
- Create Lens visualization (errors per 15 min).
- Save a dashboard.
- Explore retention: rollover + delete after N days.

---

## How to Run per Profile
```bash
./gradlew bootRun --args='--spring.profiles.active=dev'
./gradlew bootRun --args='--spring.profiles.active=stage'
./gradlew bootRun --args='--spring.profiles.active=prod'
```

Mount matching log path into Logstash (`./logs`).

---

## Final Outcomes
- Understand SLF4J, Logback, ELK pipeline.
- Configure profile-based logging via `application-*.properties`.
- Output structured JSON logs.
- Run local ELK in Docker.
- Query & visualize logs in Kibana.
