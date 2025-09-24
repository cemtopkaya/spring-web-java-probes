# HEALTH PROBE DEMO

Spring Boot uygulamalarının **Liveness**, **Readiness**, ve **Startup** probları, özellikle Kubernetes gibi container orchestration ortamlarında uygulamanın sağlık durumunu doğru bir şekilde yönetmek için oldukça önemlidir. Docker Compose ile bu problemleri kullanarak bir uygulama ayağa kaldırarak, benzer bir yapıyı daha basit bir ortamda simüle edebiliriz.

İstediğiniz senaryo için tam bir örnek proje hazırladım. Bu proje aşağıdaki bileşenleri içerecektir:

1.  **Spring Boot Uygulaması**: `management.endpoint.health.probes.enabled=true` yapılandırmasını kullanarak prob uç noktalarını etkinleştireceğiz.
2.  **docker-compose.yml**: Docker Compose ile uygulamayı ayağa kaldırırken, **Liveness**, **Readiness** ve **Startup** problarını nasıl belirtebileceğimizi gösterecek.

---

Oluşturduğunuz `application.properties` dosyasına göre, Spring Boot uygulamanızda aşağıdaki Actuator uç noktaları oluşur:

### Web Uç Noktaları

`management.endpoints.web.exposure.include=*` ayarı sayesinde, tüm Actuator uç noktaları web üzerinden erişilebilir hale gelir. Genel Actuator uç noktalarına **`/actuator`** path'i üzerinden erişebilirsiniz.

Bu, aşağıdaki ana grupların hepsini içerir:

- `/actuator`: Tüm Actuator uç noktalarının listesini gösteren ana sayfa.
- `/actuator/health`: Uygulamanın genel sağlık durumunu gösterir.
- `/actuator/info`: Uygulama hakkındaki özel bilgileri (versiyon, ad vb.) gösterir.
- `/actuator/beans`: Spring IoC konteynerindeki tüm bean'lerin listesini gösterir.
- `/actuator/env`: Uygulama ortamı özelliklerini gösterir.
- `/actuator/metrics`: Uygulama ve sistem metriklerini (CPU kullanımı, bellek vb.) gösterir.
- `/actuator/shutdown`: Uygulamayı güvenli bir şekilde kapatır (varsayılan olarak devre dışıdır).

---

### Sağlık Probu Uç Noktaları

`management.endpoint.health.probes.enabled=true` yapılandırması, prob uç noktalarını etkinleştirir. Bu ayar ve tanımladığınız sağlık grupları (`liveness` ve `readiness`) ile birlikte aşağıdaki özel prob uç noktaları aktif olur:

- `/actuator/health/liveness`: **Liveness probu** için özel bir uç noktadır. Bu, uygulamanın çalışır durumda olup olmadığını ve kendi kendini yeniden başlatmaya gerek olup olmadığını belirler. Yanıt olarak **"UP"** veya **"DOWN"** durumlarından birini döner.
- `/actuator/health/readiness`: **Readiness probu** için özel bir uç noktadır. Uygulamanın gelen trafiği işlemeye hazır olup olmadığını belirler. Yanıt olarak **"UP"** veya **"DOWN"** durumlarından birini döner.
- `/actuator/health/probes`: Tanımlanan tüm prob gruplarını (`liveness`, `readiness`) listeler ve bunların genel durumunu gösterir.
- Özellikle container orchestration ortamlarında (Docker, Kubernetes) kullanılan sağlık probları ise **`/actuator/health/liveness`** ve **`/actuator/health/readiness`** uç noktalarıdır.

---

### Hızlı Başlangıç

```sh
mvn clean package
java -jar target/health-probe-demo-0.0.1-SNAPSHOT.jar
```

```sh
curl -s -o - http://localhost:8080/actuator | jq
```

```shell
curl -s -o - http://localhost:8080/actuator | jq
{
  "_links": {
    "self": {
      "href": "http://localhost:8080/actuator",
      "templated": false
    },
    "beans": {
      "href": "http://localhost:8080/actuator/beans",
      "templated": false
    },
    "caches-cache": {
      "href": "http://localhost:8080/actuator/caches/{cache}",
      "templated": true
    },
    "caches": {
      "href": "http://localhost:8080/actuator/caches",
      "templated": false
    },
    "health": {
      "href": "http://localhost:8080/actuator/health",
      "templated": false
    },
    "health-path": {
      "href": "http://localhost:8080/actuator/health/{*path}",
      "templated": true
    },
    "info": {
      "href": "http://localhost:8080/actuator/info",
      "templated": false
    },
    "conditions": {
      "href": "http://localhost:8080/actuator/conditions",
      "templated": false
    },
    "configprops": {
      "href": "http://localhost:8080/actuator/configprops",
      "templated": false
    },
    "configprops-prefix": {
      "href": "http://localhost:8080/actuator/configprops/{prefix}",
      "templated": true
    },
    "env": {
      "href": "http://localhost:8080/actuator/env",
      "templated": false
    },
    "env-toMatch": {
      "href": "http://localhost:8080/actuator/env/{toMatch}",
      "templated": true
    },
    "loggers": {
      "href": "http://localhost:8080/actuator/loggers",
      "templated": false
    },
    "loggers-name": {
      "href": "http://localhost:8080/actuator/loggers/{name}",
      "templated": true
    },
    "heapdump": {
      "href": "http://localhost:8080/actuator/heapdump",
      "templated": false
    },
    "threaddump": {
      "href": "http://localhost:8080/actuator/threaddump",
      "templated": false
    },
    "metrics-requiredMetricName": {
      "href": "http://localhost:8080/actuator/metrics/{requiredMetricName}",
      "templated": true
    },
    "metrics": {
      "href": "http://localhost:8080/actuator/metrics",
      "templated": false
    },
    "scheduledtasks": {
      "href": "http://localhost:8080/actuator/scheduledtasks",
      "templated": false
    },
    "mappings": {
      "href": "http://localhost:8080/actuator/mappings",
      "templated": false
    }
  }
}
```

```sh
curl -s -o - http://localhost:8080/actuator/health | jq
```

```shell
$ curl -s -o - http://localhost:8080/actuator/health | jq
{
  "status": "UP",
  "groups": [
    "liveness",
    "readiness"
  ]
}
```

---

### 1. Spring Boot Uygulaması

Öncelikle, `Liveness` ve `Readiness` problarının doğru çalışması için Spring Boot uygulamanızı **Spring Boot Actuator** ile yapılandırmanız gerekir.

**`pom.xml`**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.1.4</version>
        <relativePath/> </parent>
    <groupId>com.example</groupId>
    <artifactId>health-probe-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>health-probe-demo</name>
    <description>Demo project for Spring Boot Health Probes</description>
    <properties>
        <java.version>17</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
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

**`src/main/resources/application.properties`**:

Bu dosya, prob uç noktalarını etkinleştirmek ve erişilebilir hale getirmek için ana yapılandırmayı içerir.

```properties
# Web sunucusu portu
server.port=8080

# Actuator uç noktalarını etkinleştirme
management.endpoints.web.exposure.include=*
#management.endpoints.web.exposure.include=health,info,probes
management.endpoint.health.probes.enabled=true

# Health grup tanımlamaları
# Liveness probu için gerekli yapılandırma
management.endpoint.health.group.liveness.include=livenessState

# Readiness probu için gerekli yapılandırma
management.endpoint.health.group.readiness.include=readinessState
```

**`src/main/java/com/example/healthprobedemo/HealthProbeDemoApplication.java`**:

Basit bir ana sınıf ve bir örnek REST denetleyicisi.

```java
package com.example.healthprobedemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class HealthProbeDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(HealthProbeDemoApplication.class, args);
    }

    @GetMapping("/")
    public String home() {
        return "Merhaba, Docker Compose ve Health Probes!";
    }
}
```

---

### 2. Docker Compose Dosyası (`docker-compose.yml`)

Bu dosya, Docker konteynerinizi ayağa kaldırmak ve **Liveness**, **Readiness** ve **Startup** problarını tanımlamak için en kritik bileşendir. `healthcheck` bölümü bu probları yönetmek için kullanılır.

```yaml
name: probes-java

services:
  spring-app:
    image: openjdk:17-jdk-slim
    command: ['java', '-jar', '/app/health-probe-demo-0.0.1-SNAPSHOT.jar']
    volumes:
      - ${WORKSPACE_PATH_ON_HOST}/target:/app
    ports:
      - '8080:8080'
    healthcheck:
      test: ['CMD', 'curl', '--fail', 'http://localhost:8080/actuator/health/liveness']
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    restart: unless-stopped
```

Yukarıdaki `docker-compose.yml` dosyası, `healthcheck` mekanizmasını kullanarak bir prob örneği gösteriyor. Bu örnekte, `healthcheck` **Liveness** probunu kullanarak konteynerin durumunu izler.

- `test: ["CMD", "curl", "--fail", "http://localhost:8080/actuator/health/liveness"]`: Bu komut, konteyner içinden `/actuator/health/liveness` uç noktasına bir `curl` isteği gönderir. Eğer istek başarılı olursa (`200 OK` dönüyorsa), prob başarılı kabul edilir.
- `interval: 10s`: Her 10 saniyede bir prob kontrolü yapılır.
- `timeout: 5s`: Prob komutunun tamamlanması için maksimum bekleme süresi.
- `retries: 3`: Bir probun başarısız olduğu durumda, konteynerin `unhealthy` olarak işaretlenmeden önce kaç kez daha deneme yapılacağı.
- `start_period: 30s`: Uygulamanın tam olarak başlaması için verilen süre. Bu süre boyunca yapılan prob başarısızlıkları `retries` sayısına dahil edilmez. Bu, özellikle uygulamanızın ilk başlatma anında yavaş çalışması durumunda, **Startup** probu gibi davranır.

**Projenin Dizin Yapısı:**

```
.
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           └── healthprobedemo/
│   │   │               └── HealthProbeDemoApplication.java
│   │   └── resources/
│   │       └── application.properties
├── pom.xml
└── docker-compose.yml
```

### Projeyi Çalıştırma Adımları

1.  **Spring Boot Uygulamasını Derleyin**: Proje dizininde `mvn package` komutunu çalıştırarak JAR dosyasını oluşturun.
2.  **Docker Compose ile Uygulamayı Başlatın**: Aynı dizinde `docker-compose up` komutunu çalıştırın.

Bu adımları tamamladığınızda, Docker Compose konteyneri ayağa kaldıracak ve `docker ps` komutu ile konteynerin durumunu kontrol ettiğinizde `healthy` olarak çalıştığını göreceksiniz. `docker logs` ile konteynerin loglarını inceleyerek de uygulamanın başlatma ve prob kontrolleri hakkında bilgi edinebilirsiniz.
