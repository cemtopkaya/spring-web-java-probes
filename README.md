# HEALTH PROBE DEMO

## Konteynerleştirmede Health Probe'lara Giriş

Spring Boot uygulamalarının **Liveness**, **Readiness**, ve **Startup** probları, özellikle Kubernetes gibi container orchestration ortamlarında uygulamanın sağlık durumunu doğru bir şekilde yönetmek için oldukça önemlidir.

Docker'da kullandığımız `HEALTHCHECK`, temel bir sağlık kontrolü sağlar ancak Kubernetes'in `startupProbe`, `livenessProbe` ve `readinessProbe` gibi ayrı ve daha gelişmiş proplarına sahip değildir. `Dockerfile` ve `docker-compose.yml` içinde `HEALTHCHECK` özelliğini kullanarak konteynerinizin startup ve liveness kontrollerini yönetebilirsiniz. Ancak, Kubernetes'teki gibi startup, liveness ve readiness proplarını ayrı ayrı tanımlama imkanı yoktur. Docker Compose, bu kontrolleri tek bir `healthcheck` bloğu altında birleştirir.

Kubernetes, bu propları kullanarak uygulamanın yaşam döngüsünü daha hassas bir şekilde yönetir:

- `startupProbe`: Yalnızca başlangıç aşamasını yönetir. Uygulamanın ilk başlatma anında hazır olup olmadığını kontrol etmektir. Uygulamanızın tüm bağımlılıklarını (veritabanı bağlantısı, önbellek vb.) yüklemesi ve servise hazır hale gelmesi zaman alabilir. startupProbe, bu yavaş başlangıç süresine tolerans gösterir. Eğer `startupProbe` tanımlı olmasaydı, `livenessProbe` çok daha kısa bir süre sonra devreye girecek ve uygulamanızın yavaş başlaması nedeniyle onu `unhealthy` olarak işaretleyip yeniden başlatacaktı. Bu da uygulamanın sürekli olarak döngüsel bir şekilde yeniden başlamasına (_crashloop_) neden olacaktı.

- `livenessProbe`: Uygulama çalıştığı sürece sürekli olarak canlılık durumunu kontrol eder. Uygulamanızın çalışır durumdayken beklenmedik bir şekilde kilitlenip kilitlenmediğini kontrol etmektir. Uygulama bir hata nedeniyle yanıt veremez hale gelirse, livenessProbe bunu tespit eder.

- `readinessProbe`: Uygulamanın dışarıdan gelen trafiği kabul etmeye hazır olup olmadığını belirler. Yani: "İstek alabilir miyim?" sorusuna evet veya hayır cevabı verir.Dışarıdan gelen trafiği kabul etmek için uygulamanın kullandığı 3. parti bağımlılıkların da (rabbitmq, memcached, db vs.) ayakta ve erişilebilir olması gerekebilir.

Bu üç prob, özellikle mikroservis mimarilerinde veya karmaşık uygulamalarda çok daha detaylı ve güvenilir bir kontrol sağlar.

Aşağıda kullanılacak bayrakların açıklamaları (ortak oldukları için burada topladım):

- `test`: Bu kısım, sağlık kontrolünün yapılacağı komutu tanımlar. Sizin `CMD` (`curl`) komutunuzu burada kullanabilirsiniz. `curl --fail` komutu, 200 olmayan HTTP durum kodlarında (örneğin 404, 500) başarısız olacaktır.

- `--interval=10s`: Her 10 saniyede bir sağlık kontrolü yapar.

- `--timeout=5s`: Komutun tamamlanması için maksimum bekleme süresi 5 saniyedir. Eğer curl komutu 5 saniye içinde bir yanıt alamazsa, o deneme başarısız sayılır.

- `--start-period=30s`: Bu parametre, Kubernetes'teki `startupProbe`'a en çok benzeyen kısımdır. Konteyner başladıktan sonra `start-period` ile belirtilen ilk 30 saniye içinde her `interval` süresinde `test` çalıştırılır ve yapılan kontrollerin (`--fail`) başarısız olması durumunda, Docker bunu hemen "başarısız" olarak işaretlemez. Bu süre, uygulamanın tamamen başlaması için tanınan bir "lütuf süresidir".

- `--retries=3`: start-period bittikten sonra, eğer `HEALTHCHECK` komutu üç kez üst üste başarısız olursa, konteyner `unhealthy` olarak işaretlenir.

**Hesaplama**: `start_period` + (`retries` \* `interval`) = 30s + (3 \* 10s) = 60 saniye

_Dockerfile için:_

```dockerfile
HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=3 \
  CMD curl --fail http://localhost:8080/actuator/health/liveness || exit 1
```

_Docker Compose için:_

```yaml
services:
  spring-app:
    image: spring-app-image:latest
    ports:
      - '8080:8080'
    healthcheck:
      test: ['CMD', 'curl', '--fail', 'http://localhost:8080/actuator/health/liveness']
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
```

_Kubernetes için:_

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: spring-app-startup-probe-example
  labels:
    app: spring-app
spec:
  containers:
    - name: my-spring-app
      image: spring-app-image:latest
      ports:
        - containerPort: 8080
      startupProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        failureThreshold: 3
        periodSeconds: 10
        timeoutSeconds: 5
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        failureThreshold: 3
        periodSeconds: 10
        timeoutSeconds: 5
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        failureThreshold: 3
        periodSeconds: 10
        timeoutSeconds: 5
```

**Hesaplama**: `failureThreshold` \* `periodSeconds` = 3 \* 10 = 100 saniye

- **`startupProbe`**: Uygulamanın tam olarak başlatılması için gereken süreyi kapsayan ilk kontrol probu.

  - **`httpGet`**: `liveness` endpoint'ine HTTP GET isteği gönderir.
  - **`failureThreshold: 3`**: Uygulama, 3 kez üst üste başarısız olursa (yani `200 OK` dışında bir durum kodu dönerse veya zaman aşımı olursa), Kubernetes Pod'u yeniden başlatır. Bu değer, uygulamanın yavaş başlamasına karşı tolerans sağlar.
  - **`periodSeconds: 10`**: Her 10 saniyede bir kontrol yapar. Bu, toplamda `3 x 10 = 30 saniye` bekleme süresi tanır.
  - **`timeoutSeconds: 5`**: İstek 5 saniyeden uzun sürerse, başarısız olarak kabul edilir.

- **`livenessProbe`**: `startupProbe` başarılı olduktan sonra devreye girer. Uygulamanın çalışır durumda olduğunu ve servise devam ettiğini kontrol eder.

  - Bu prob da aynı `liveness` endpoint'ini kullanır. Eğer uygulama, bir hata veya kilitlenme nedeniyle bu endpoint'e cevap veremez hale gelirse, `failureThreshold: 3` deneme sonrasında konteyner yeniden başlatılır.

- **`readinessProbe`**: Uygulamanın dışarıdan gelen trafiği kabul etmeye hazır olduğunu kontrol eder.

  - Genellikle bağımlılıkların (veritabanı, önbellek vb.) hazır olup olmadığını kontrol eden farklı bir endpoint (`/actuator/health/readiness`) kullanılır. Bu sayede, uygulama çalışır durumda olsa bile (liveness), bağımlılıkları hazır olana kadar trafiği reddedebilir.

## Spring Boot İçinde Probe Uygulaması

`management.endpoints.web.exposure.include=health` ifadesi http://localhost:8080/actuator/health adresine erişim sağlar ve `{"status":"UP"}` JSON cevabı döner. Genel bir sağlık durumu döner ama liveness ve readiness probes için ayrı ayrı durumlarını görebileceğimiz uç noktalarını sunmaz!

`management.endpoints.web.exposure.include=health,probes` ifadesi `Spring Boot Actuator` yapılandırmasında uç noktlarının hangilerinin HTTP üzerinden dış dünyaya (web üzerinden) açılacağını belirtir.

- `health`: /actuator/health endpoint’ini açar.
- `probes`: Kubernetes için özel olarak tasarlanmış liveness ve readiness probe endpoint’lerini açar (`/actuator/health/liveness` ve `/actuator/health/readiness` gibi). Ancak `../health/liveness` ve `../health/readiness` uç noktalarını görmek için ek bir ayar gerekir (`management.endpoint.health.probes.enabled=true`). Şimdilik `../health` uç noktasının cevabı `{"status":"UP"}` olacaktır.

> probes eklemek, health endpoint’inin detaylı alt endpoint’lerini (özellikle liveness/readiness) aktif hale getirir.

- `health`: Uygulamanın sağlığına ilişkin temel bilgileri sağlar. Sistem bileşenlerinin durumu, veritabanı bağlantıları gibi kriterler bu endpoint üzerinden izlenebilir. Genellikle uygulamanın çalışıp çalışmadığını anlamak için kullanılır.
- `probes`: Kubernetes gibi konteyner orkestrasyon sistemlerinde sağlık kontrolleri için kullanılan özel yaşam ve hazır olma kontrollerini ([`liveness`](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.application-availability.liveness)/[`readiness`](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.application-availability.readiness) probes) sunar.

Hepsi uygulamanın çalışma durumu ve bilgisi ile ilgili olsalar da farklı amaçlara hizmet ederler. `management.endpoints.web.exposure.include=health,info,probes` ifadesi bu üç endpoint'in HTTP üzerinden erişilebilir olduğunu belirtir, hepsi ayrı ayrı da kullanılabilir ve hepsi uygulama yönetimi için ortak ama farklı işlevler sunar. Kullanıcı gereksinimine göre içlerinden biri veya birkaçı web üzerinden sağlanabilir.
- `health` genel sağlık durumu için,
- `info` uygulama bilgileri için
- ve `probes` Kubernetes ortamındaki container sağlık kontrolleri içindir.

Bu üç endpoint birbirinden bağımsızdır ve ayrı ayrı da kullanılabilir. Ancak genellikle birlikte yapılandırılırlar çünkü her biri farklı bir yönetim ve izleme amacına hizmet eder. Yönetim uç noktalarının hangilerinin web üzerinden erişilebilir olacağını belirtmek için aynı property altında virgülle ayrılarak birden fazla endpoint belirtilebilir (`management.endpoints.web.exposure.include=health,probes,info` gibi).


Tüm probe senaryolarını öğrenmek ve denemek için aşağıdaki 2 maddeyi takip edebiliriz:

1.  **Spring Boot Uygulaması**: `management.endpoint.health.probes.enabled=true` yapılandırmasını kullanarak prob uç noktalarını etkinleştireceğiz. Bu aşamada ornek1-mustakil modülünü koşturacağız. Uygulama ayağa kalkınca http://localhost:8080/actuator/\* adreslerinden liveness ve readiness problarını kontrol edeceğiz.

2.  **docker-compose-test.yml**: Docker Compose ile uygulamayı ayağa kaldırırken, **Liveness**, **Readiness** problarını nasıl belirtebileceğimizi göstereceğiz ancak bu kez uygulamanın RabbitMQ ve DB bağımlılıkları bu durumları etkileyecek.

Oluşturduğunuz Spring Boot uygulamasının `application.properties` dosyasına göre, Spring Boot uygulamanızda aşağıdaki Actuator uç noktaları oluşur:

### Web Uç Noktaları

`management.endpoints.web.exposure.include=*` ayarı sayesinde, tüm Actuator uç noktaları web üzerinden erişilebilir hale gelir. Genel Actuator uç noktalarına **`/actuator`** path'i üzerinden erişebilirsiniz.

Bu ayar (`management.endpoints.web.exposure.include=health,info,beans,env,metrics,shutdown`), aşağıdaki ana grupların hepsini içerir:

- `/actuator`: Tüm Actuator uç noktalarının listesini gösteren ana sayfa.
- `/actuator/health`: Uygulamanın genel sağlık durumunu gösterir.
- `/actuator/info`: Uygulama hakkındaki özel bilgileri (versiyon, ad vb.) gösterir.
- `/actuator/beans`: Spring IoC konteynerindeki tüm bean'lerin listesini gösterir.
- `/actuator/env`: Uygulama ortamı özelliklerini gösterir.
- `/actuator/metrics`: Uygulama ve sistem metriklerini (CPU kullanımı, bellek vb.) gösterir.
- `/actuator/shutdown`: Uygulamayı güvenli bir şekilde kapatır (varsayılan olarak devre dışıdır).

### Sağlık Probu Uç Noktaları

`management.endpoints.web.exposure.include=health,probes` + `management.endpoint.health.probes.enabled=true` yapılandırması, prob uç noktalarını etkinleştirir. Bu ayarlar ve tanımladığınız sağlık grupları (`liveness` ve `readiness`) ile birlikte aşağıdaki özel prob uç noktaları aktif olur:

- `/actuator/health/liveness`: **Liveness probu** için özel bir uç noktadır. Bu, uygulamanın çalışır durumda olup olmadığını ve kendi kendini yeniden başlatmaya gerek olup olmadığını belirler. Yanıt olarak **"UP"** veya **"DOWN"** durumlarından birini döner.
- `/actuator/health/readiness`: **Readiness probu** için özel bir uç noktadır. Uygulamanın gelen trafiği işlemeye hazır olup olmadığını belirler. Yanıt olarak **"UP"** veya **"DOWN"** durumlarından birini döner (`{"status":"UP"}` JSON cevabını döner).
- `/actuator/health`: Tanımlanan tüm prob gruplarını (`liveness`, `readiness`) listeler ve bunların genel durumunu gösterir.
  En basit haliyle `{ "status": "UP", "groups": [ "liveness", "readiness" ] }`cevabını döndürür.
- Özellikle container orchestration ortamlarında (Docker, Kubernetes) kullanılan sağlık probları ise **`/actuator/health/liveness`** ve **`/actuator/health/readiness`** uç noktalarıdır.

> `*.properties` dosyasında bu değerler verildiği gibi bir `*.yaml` dosyasında da verilebilir.

Genelde `livenessState` için özel indicator eklenmez, varsayılan `livenessState` yeterlidir.
Ancak `readinessState` 3. parti bağımlılıkların gruplanmasıyla da belirlenebilir. Spring Boot’un sağladığı bazı health indicator isimleri: ` db`, `rabbit`, `kafka`, `diskSpace`, `mongo`, `redis`. Bununla beraber kendi özel sağlık durumu göstergelerimizi de yazabiliriz. Örneğim memcached için aşağıdaki sınıfı oluşturalım:

```java
@Component("memoryCache")
public class MemcachedHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            // Memcached bağlantısını test et
            if (memcachedClient.isConnected()) {
                return Health.up().withDetail("server", "connected").build();
            } else {
                return Health.down().withDetail("reason", "not connected").build();
            }
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

```yaml
management:
  endpoint:
    health:
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, rabbit, kafka, diskSpace, memoryCache
```

**Kaynaklar:**

- https://spring.io/blog/2020/03/25/liveness-and-readiness-probes-with-spring-boot
- https://docs.spring.io/spring-boot/api/rest/actuator/index.html

---

### Hızlı Başlangıç

```sh
mvn clean package
java -jar target/ornek1-mustakil-0.0.1-SNAPSHOT.jar
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

Bağımlılıklarla probe kontrolü için:

```sh
docker compose -f ./docker-compose-test.yml up
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

---

### 2. Docker Compose Dosyası (`docker-compose-test.yml`)

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
      - '8090:8080'
    healthcheck:
      test: ['CMD', 'curl', '--fail', 'http://localhost:8080/actuator/health/liveness']
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    restart: unless-stopped

  rabbitmq:
    image: rabbitmq:3.8-management
    ports:
      - '5672:5672'
      - '15672:15672'
    healthcheck:
      test: ['CMD', 'rabbitmq-diagnostics', 'check_port_connectivity']
      interval: 5s
      timeout: 10s
      retries: 5
```

Yukarıdaki `docker-compose-test.yml` dosyası, `healthcheck` mekanizmasını kullanarak bir prob örneği gösteriyor. Bu örnekte, `healthcheck` **Liveness** probunu kullanarak konteynerin durumunu izler.

- `test: ["CMD", "curl", "--fail", "http://localhost:8090/actuator/health/liveness"]`: Bu komut, konteyner içinden `/actuator/health/liveness` uç noktasına bir `curl` isteği gönderir. Eğer istek başarılı olursa (`200 OK` dönüyorsa), prob başarılı kabul edilir.
- `interval: 10s`: Her 10 saniyede bir prob kontrolü yapılır.
- `timeout: 5s`: Prob komutunun tamamlanması için maksimum bekleme süresi.
- `retries: 3`: Bir probun başarısız olduğu durumda, konteynerin `unhealthy` olarak işaretlenmeden önce kaç kez daha deneme yapılacağı.
- `start_period: 30s`: Uygulamanın tam olarak başlaması için verilen süre. Bu süre boyunca yapılan prob başarısızlıkları `retries` sayısına dahil edilmez. Bu, özellikle uygulamanızın ilk başlatma anında yavaş çalışması durumunda, **Startup** probu gibi davranır.

```sh
docker compose -f ./docker-compose-test.yml up
```

### Projeyi Çalıştırma Adımları

1.  **Spring Boot Uygulamasını Derleyin**: Proje dizininde `mvn package` komutunu çalıştırarak JAR dosyasını oluşturun.
2.  **Docker Compose ile Uygulamayı Başlatın**: Aynı dizinde `docker-compose up` komutunu çalıştırın.

Bu adımları tamamladığınızda, Docker Compose konteyneri ayağa kaldıracak ve `docker ps` komutu ile konteynerin durumunu kontrol ettiğinizde `healthy` olarak çalıştığını göreceksiniz. `docker logs` ile konteynerin loglarını inceleyerek de uygulamanın başlatma ve prob kontrolleri hakkında bilgi edinebilirsiniz.
