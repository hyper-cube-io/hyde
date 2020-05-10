---
layout: post
title: Prometheus + Grafana로  Java 애플리케이션 모니터링하기
author: kwSeo
tags: [observability, monitoring, prometheus, java, spring, jmx]
---

해당 글은 NHN 기술 블로그 Meetup에도 게시되어 있습니다.
[https://meetup.toast.com/posts/237](https://meetup.toast.com/posts/237)

## 목차

- 요약
- 모니터링 환경 구축
  - Prometheus
  - Grafana
- Java Metrics
  - With Spring Boot Actuator
    - Micrometer
  - With JMX Exporter
- Grafana + Prometheus + Spring Boot 연동
- 부록

## 요약

### Spring Boot Actuator + Micrometer Registry

- spring boot actuator와 micrometer-registry-prometheus 추가

#### pom.xml

```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-actuator</artifactId>  
</dependency>  
<dependency>  
    <groupId>io.micrometer</groupId>  
    <artifactId>micrometer-registry-prometheus</artifactId>  
</dependency>
```

#### application.yml

- spring boot actuator의 prometheus 엔드포인트 활성화
- 서비스를 구분하기 위한 common tag 추가(서비스는 한 개의 이상이 인스턴스로 이루어짐)
  - 일반적으로 application이라는 이름으로  `spring.application.name` 속성이 많이 사용됨
  - 각 인스턴스를 구분하기 위한 instance 태그는 Prometheus에 의해서 추가됨으로 설정 불필요

```yaml
spring:
  application:
    name: my_spring_boot_app
management:
  endpoints:  
    web:
      exposure:
        include: "prometheus"
  metrics:
    tags:
      application: ${spring.application.name}    # 서비스 단위의 식별자. Prometheus label에 추가됨.
```

### Spring Boot가 아닌 경우

- 모니터링할 대상이 Prometheus 모니터링을 지원하는지 공식문서 확인
- 지원하지 않는 다면 JMX Exporter 사용을 고려
  - https://github.com/prometheus/jmx_exporter

### Prometheus

#### Install

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.17.1/prometheus-2.17.1.linux-amd64.tar.gz
tar -xzf prometheus-2.17.1.linux-amd64.tar.gz
cd prometheus-2.17.1.linux-amd64
```

#### Configuration prometheus.yml

- targets에 모니터링할 Spring Boot 애플리케이션 IP:Port  추가
- Service Discovery를 사용한다면 관련 Service Discovery 설정 추가

```yaml
global:
  scrape_interval: "15s"
  evaluation_interval: "15s"
scrape_configs:
- job_name: "springboot"
  metrics_path: "/actuator/prometheus"
  static_configs:
  - targets:
    - "<my_spring_boot_app_ip>:<port>"
- job_name: "prometheus"
  static_configs:
  - targets:
    - "localhost:9090"
```

#### Run

- 기본 포트: 9090
- 기본 설정 파일: prometheus.yml

```bash
./prometheus
```

### Grafana

#### Install and Run

- 기본 포트: 3000
- 기본 계정 ID/PW: admin/admin

```bash
wget https://dl.grafana.com/oss/release/grafana-6.7.2.linux-amd64.tar.gz
tar -zxvf grafana-6.7.2.linux-amd64.tar.gz
cd grafana-6.7.2
./bin/grafana-server
```

#### Add Prometheus datasource

- 실행한 Prometheus를 Grafana의 데이터소스로 추가

**첫 페이지의 경우 아래의 버튼 추가**
![add_datasource_1.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_ds1.png)

**Prometheus 선택**
![add_datasource_2.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_ds2.png)

**설치한 Prometheus 정보 입력**
![add_datasource_3.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_ds3.png)

#### Add Dashboard

**Marketplace**

- Grafana 마켓플레이스에서 원하는 대시보드를 검색 및 import
  - https://grafana.com/grafana/dashboards?dataSource=prometheus&direction=asc&orderBy=name
- 마음에 드는 대시보드가 없다면 export된 다른 사람의 JSON 파일을 다운로드 받거나 직섭 대시보드를 생성

![add_dashboard_1.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_dashboard1.png)

![add_dashboard_2.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_dashboard2.png)

![add_dashboard_3.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_dashboard3.png)

#### 직접 그래프 추가

- 각 대시보드는 변수를 추가해서 사용 가능 
  - `my dashboard -> Setting -> Variables`에서 설정
  - PromQL 내에서 `$variable_name` 같은 형태로 사용 가능
- PromQL(Prometheus Query Language)를 통해서 직접 지표를 가공 및 조회할 수 있음
  - **PromQL 예제**
    - 특정 서비스내 인스턴스의 API별 초 당 요청 수 조회
      - `irate(http_server_requests_seconds_count{application="$application", instance="$instance"}[3m])`
    - 특정 서비스의 초 당 에러 요청 수 합계 조회
      - `sum by (application) (irate(http_server_requests_seconds_count{application="$application", outcome=~"CLIENT_ERROR|SERVER_ERROR"}[3m]))`
  - 조회 결과 시리즈 데이터의 레이블은 범례 이름으로 사용 가능. `{{ label_name }}`와 같은 형태로 레이블 지정

![add_graph_1.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_panel1.png)

![add_panel_2.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_panel2.png)

![add_panel_3.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_panel3.png)

---

## 모니터링 환경 구축

### Prometheus

Prometheus 2012년에 처음으로 모습을 드러낸 오픈소스 모니터링 플랫폼입니다.
기존의 다른 모니터링 플랫폼과는 다르게 Pull 방식을 사용하며, 각 모니터링 대상의 Exporter 또는 Prometheus client를 통해서 지표를 긁어가는(scrape) 방식으로 데이터를 수집합니다.
Prometheus는 다른 모니터링 플랫폼에 비해 사용과 설정이 간단함에도 불구하고 유연하며 매우 좋을 성능을 내어 많은 사랑을 받아왔습니다.

Prometheus는 Google의 내부 모니터링 시스템이던 Borgmon에 영향을 받아 시작된 Go언어 오픈소스 프로젝트입니다.
2012년부터 꾸준히 커뮤니티가 성장해왔고 현재는 매년 PromCon이라는 컨퍼런스를 진행할 정도로 큰 커뮤니티를 가지게 되었습니다.
또한, CNCF(Cloud Native Computing Foundation)의 Graduated 프로젝트가 되어 현재 컨테이너 모니터링의 사실상 표준처럼 사용되고 있습니다.
Prometheus에 대한 보다 자세한 내용은 생략하고 바로 Prometheus를 사용하여 우리의 애플리케이션을 모니터링할 수 있도록 Prometheus를 사용해보는데 중점을 두도록 하겠습니다.

#### Install

Prometheus는 단일 코드로 여러 운영체제의 바이너리를 빌드할 수 있는 Go 언어로 개발되었습니다. 
Go 언어 생태계가 그러하듯 Prometheus도 다양한 운영체제의 바이너리 파일과 컨테이너 이미지를 제공하고 있습니다.
이 문서에서는 컨테이너 환경이 아닌 on-promise 환경을 기준으로 하겠습니다.
하지만 그래도 간단합니다.
우리에게 필요한 운영체제의 바이너리 파일만 다운로드 받아서 실행하면 됩니다.
아래의 페이지를 통해서 다운로드 받을 수 있습니다.

[https://prometheus.io/download/](https://prometheus.io/download/)

리눅스의 주요 명령어를 활용한다면 아래와 같이 사용할 수도 있을 겁니다.
CentOS7을 사용하였습니다.

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.17.1/prometheus-2.17.1.linux-amd64.tar.gz
tar -xzf prometheus-2.17.1.linux-amd64.tar.gz
```

#### Run

Prometheus의 실행은 매우 간단합니다.
위에서 언급했던 Go 언어는 최종적으로 각 운영체제에도 동작할 수 있는 바이너리를 생성합니다.
일반 바이너리 파일처럼 실행하면 됩니다.

```bash
cd prometheus-2.17.1.linux-amd64
./prometheus
```

끝입니다.
모니터링 지표 수집을 위한 기본적인 준비는 완료되었습니다.
기본적으로 설정 파일은  같은 디렉토리 내의 prometheus.yml을 사용하며 포트는 9090사용합니다.
브라우저를 통해서 `localhost:9090`에 접근하면 아래와 같은 Prometheus 웹 화면이 보일 것입니다.

![prom_ui_1.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/prom.png)

이 웹 화면에서 Prometheus의 조회 쿼리를 테스트해볼 수 있고 추가적인 설정을 통해서 지표 기록 및 알림 상태 등을 확인할 수 있습니다.
하지만 저희가 이 Prometheus 웹 화면을 사용할 일은 거의 없습니다.
저희에게는 Grafana라는 훌륭한 오픈소스 대시보드가 있기 때문입니다.

### Grafana

Grafana는 다양한 데이터소스로부터 데이터를 가져와 대시보드를 구성할 수 있도록 돕는 오픈소스 프랫폼입니다.
백엔드 웹서버는 Go 언어로 개발되었으며 프론트엔드는 초기에 Angular를 사용하였으나 현재는 TypeScript + React를 주로 사용하고 있습니다.
일반적으로 Grafana에서 말하는 데이터소스는 실제 시계열 지표 성격의 데이터를 저장하고 조회할 수 있는 플랫폼들을 말합니다.
현재 official로 지원하는 데이터소스는 Graphite, Prometheus, InfluxDB, Elasticsearch, AWS CloudWatch, OpenTSDB... 등으로 사실상 많이 사용되고 있는 대부분의 스토리지는 지원한다고 볼 수 있습니다.
official 데이터소스가 아니더라도 다른 사용자가 제작한 데이터소스 플러그인을 사용할 수도 있기에 지원범위가 매우 넓습니다.
Prometheus 데이터소스는 위에서 언급한대로 Grafana의 official 데이터소스 중 하나이기 때문에 Grafana에 기본적으로 built-in되어있어 저희가 별도로 설치해야할 것은 없습니다.

#### Install

아래의 페이지에서 Grafana 바이너리와 각 운영체제 및 설치 방법에 따른 가이드가 제공되고 있습니다.
역시 단일 바이너리 형태로 제공되기 때문에 기능 테스트 용도라면 그저 바이너리 파일을 다운로드받아서 간단하게 실행해볼 수 있습니다.

[https://grafana.com/grafana/download](https://grafana.com/grafana/download)

```bash
wget https://dl.grafana.com/oss/release/grafana-6.7.2.linux-amd64.tar.gz
tar -zxvf grafana-6.7.2.linux-amd64.tar.gz
```

#### Run

standalone으로 실행한다면 별다른 설정이 필요하지 않습니다.
물론, 고가용성 또는 보안(인증, 권한 등)이 필요하거나 많은 트레픽을 처리해야하는 경우 별도의 설정이 필요할 것입니다.

```bash
cd grafana-6.7.2
./bin/grafana-server
```

Grafana 웹서버의 기본 포트는 3000입니다.
브라우저를 통해서 `http://localhost:3000`으로 접속하면 Grafana 로그인 페이지가 보일 것 입니다.

![grafana_login_1.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/grafana_login.png)

기본 관리자 계정은 `ID: admin`, `PW: admin`입니다.
로그인에 성공하면 Grafana에서 다음 단계를 위한 가이드를 보여줄 것 입니다.

#### Add Prometheus Data Source

상단의 `Add data source` 버튼을 클릭하거나 왼쪽 사이드메뉴의 `Configuration > Data sources > Add data source` 버튼을 클릭하여 데이터소스를 추가할 수 있습니다.

![add_datasource_1.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_ds1.png)

클릭하면 아래와 같은 화면이 나타납니다.
`Time series databases` 탭에서 `Prometheus` 선택합니다.

![add_datasource_2.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_ds2.png)

원하는 이름으로 `Name` 필드를 작성하고 아래 HTTP 탭의 `URL` 필드에서는 조금 전에 설치하여 실행한 Prometheus의 URL을 입력합니다.
Grafana와 Prometheus를 같은 호스트에 설치하였다면 `http://lcoalhost:9090`을 입력하면 될 겁니다.
값을 입력하고 아래의 `Save & Test` 버튼을 클릭하여 데이터소스를 저장합니다.
입력한 설정 값에 문제가 없다면 Success라는 녹색창의 메시지가 노출됩니다.

![add_datasource_3.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_ds3.png)

이렇게 간단하게 Prometheus와 Grafana를 연동할 수 있습니다.
제대로 동작하는지 확인을 위해서 새로 대시보드를 생성하여 차트를 만들어볼 수 있습니다.
하지만 현재 모니터링하고 있는 애플리케이션이 없기 때문에 애초에 차트로 만들 것이 없습니다.
(기본 설정상 Prometheus는 자기 자신도 모니터링하기 때문에 수집되는 지표가 존재하기는 하겠지만 여기서는 Java 애플리케이션이 목적이기 때문에 넘어가도록 하겠습니다.)

## Java Metrics

### With Spring Boot Actuator

**Spring Boot 2 이상 버전을 기준으로 합니다.**
Spring Boot Actuator는 Spring Boot 애플리케이션을 관리하기 위한 도구를 JMX 또는 HTTP 엔드포인트 형태로 제공합니다.
info, logger, env, health 등 많은 기능을 제공하지만 여기서는 prometheus 엔드포인트를 사용할 것 입니다.
이름 그대로 Prometheus 서버가 지표를 수집할 수 있도록 애플리케이션 지표를 Prometheus 포맷으로 expose합니다.
이를 이용해서 큰 고생없이 Spring Boot 앱의 지표를 Prometheus로 수집할 수 있도록 준비할 수 있습니다.
이 엔드포인트는 평범한 HTTP API이기 때문에 저희가 직접 요청해서 데이터를 확인해볼 수도 있습니다.
(NSight의 Spring Boot 모니터링도 prometheus 엔드포인트를 사용하여 지표를 수집하고 있습니다)

#### 필요한 의존성

Spring Boot Actuator는 물론 필요하며 추가적으로  **Micrometer Registry**가 필요합니다.
Micrometer는 범용 지표 측정 라이브러리로, Spring Boot 2 버전부터 Spring Boot Actuator의 근간을 이루는 핵심 라이브러리입니다. 
때문에 Micrometer Core는 이미 Spring Boot Actuator에 포함되어 있습니다.
Micrometer의 또다른 목적은 지표 수집의 추상화입니다. 
Micrometer는 다음과 같이 자신을 소개하고 있습니다.

> Think SLF4J, but for metrics.

SLF4J로 로그를 남기듯 동일한 인터페이스로 지표를 측정하고, 서로 다른 모니터링 플랫폼에서 수집이 가능하도록 도와줍니다.
이때 Micrometer로 측정된 지표를 실제 백엔드에 저장하는 기능을 구현한 것이  **Micrometer Registry**입니다.
Spring Boot Actuator는 의존성에 어떤 Micrometer Registry가 포함되었는지 확인하고, 감지된 Registry에 맞춰서 AutoConfiguration을 수행합니다.
현재 Prometheus, Elastic, InfluxDB, Dynatrace 등 많은 registry를 제공하고 있으며 전체 목록은 아래의 페이지에서 확인할 수 있습니다.

[https://micrometer.io/docs](https://micrometer.io/docs)

우리는 그저 Spring Boot Actuator와 Micrometer Registry Prometheus를 의존성에 추가하고 prometheus 엔드포인트를 expose하면 됩니다.
또한, 필수는 아니지만 각 애플리케이션 서비스를 구분하기 위한 태그(Micrometer의 tag)를 추가하는 것을 권장합니다.
여기에서 `애플리케이션 서비스`는 전체 구조를 봤을 때 각 서비스 컴포넌트를 의미합니다.
즉, 하나의 서비스는 한 개 이상의 인스턴스로 구성되어있다고 생각하시면 편할 것 같습니다.
일반적으로 태그 이름은 `application`으로, 태그 값은 Spring Boot의 `spring.application.name` 속성을 많이 사용합니다.
`management.metrics.tags.<key>: <value>`에 입력된 태그는 모든 지표에 추가될 것이며, Prometheus 포맷으로 변환시 모든 지표의 레이블에 추가됩니다.
각 애플리케이션 인스턴스를 구분하기 위한 식별자는 저희가 추가하지 않아도 됩니다. 
Prometheus가 수집하면서 자동으로 `IP:Port`로 구성된 `instance` 레이블을 추가합니다.

정리하자면 아래의 코드가 필요합니다.
아래의 설정이 정상적으로 적용되면 `http://{host}:{port}/actuator/prometheus` URL을 통해서 Prometheus에서 수집할 지표를 확인할 수 있습니다.

**pom.xml**

```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-actuator</artifactId>  
</dependency>  
<dependency>  
    <groupId>io.micrometer</groupId>  
    <artifactId>micrometer-registry-prometheus</artifactId>  
</dependency>
```

**application.yml**

```yaml
spring:
  application:
    name: my_spring_boot_app
management:
  endpoints:  
    web:
      exposure:
        include: "prometheus"
  metrics:
    tags:
      application: ${spring.application.name}	# 서비스 단위의 식별자. Prometheus label에 추가됨.
```

**정상 동작 확인**

```bash
curl "http://localhost:8080/actuator/prometheus"
```

### With JMX Exporter

Spring Boot의 경우 자체적으로 Prometheus를 지원하여 편리하게 사용할 수 있었습니다.
마찬가지로 많은 플랫폼, 프레임워크, 라이브러리가 Prometheus를 지원하지만 그렇지 않은 경우도 많이 존재합니다.
그런 경우, JMX Exporter를 사용할 수 있습니다.

[https://github.com/prometheus/jmx_exporter](https://github.com/prometheus/jmx_exporter)

이름 그대로 JMX Exporter는 Java 애플리케이션의 JMX에서 제공하는 지표를 Prometheus 포맷으로 변경하여 Prometheus가 수집할 수 있도록 도와줍니다.
어떤 JMX Bean을 지표로 사용할 것인지, 어떤식으로 값을 추출하고 어떤 이름으로 사용할 것인지 등 JMX를 Prometheus 포맷으로 가공하는데에는 정규표현식을 사용합니다.
때문에 정규표현식에 익숙치 않은 분은 설정에 어려움이 있을 수 있습니다.
아니면 미리 제공하고 있는 설정 예제를 활용하는 방법도 있습니다.
유명한 Java 프로젝트는 설정을 미리 프리셋으로 만들어서 제공하고 있습니다.

[https://github.com/prometheus/jmx_exporter/tree/master/example_configs](https://github.com/prometheus/jmx_exporter/tree/master/example_configs)

JMX Exporter는 일반적으로 JVM  agent 방식으로 사용합니다.
아래와 같은 JVM 옵션을 실행할 애플리케이션에 추가해야합니다.
아래와 같이 설정하면 `config.yml`파일을 참고하여 모니터링할 JMX를 Prometheus 포맷으로 변환하고 HTTP `/metrics`  엔드포인트를 8080 포트로 등록합니다.

```bash
java -javaagent:./jmx_prometheus_javaagent-0.12.0.jar=8080:config.yaml -jar my_app.jar
```

JMX를 사용하기에 실질적으로 거의 모든 Java 애플리케이션을 모니터링할 수 있습니다.
저희 팀 내에서 사용하고 있는 Apache Cassandra도 JMX Exporter를 통해서 모니터링하고 있습니다.

## Grafana + Prometheus + Spring Boot 연동

### Prometheus에 Spring Boot 엔드포인트 등록하기

이제 지표를 수집하기 위해서 다시 Prometheus를 살펴보겠습니다.
앞에서 언급했듯 Prometheus는 Pull 방식의 수집 방법을 사용하고 있어 직접 각 모니터링 대상에게 HTTP 요청을 하여 지표 데이터를 가져갑니다.
그래서 Promeheus에 모니터링 대상의 주소를 등록할 필요가 있습니다.
만약 Service Discovery와 연동이 되어있다면 자동으로 새로운 대상을 발견하고 모니터링을 시작하겠지만 저희가 준비한 최소설정에는 그런 설정이 없으므로 직접 손으로 입력이 필요합니다.
`prometheus.yml`을 열어서 `static_configs`의 `targets`에 앞에서 만든 애플리케이션의 주소를 입력합니다.
그리고 어떤 URL 경로(path)를 통해서 지표를 수집할지를 `metrics_path` 속성으로 입력합니다.

```yaml
global:
  scrape_interval: "15s"
  evaluation_interval: "15s"
scrape_configs:
- job_name: "springboot"
  metrics_path: "/actuator/prometheus"          # prometheus 엔드포인트 경로
  static_configs:
  - targets:
    - "<my_spring_boot_app_ip>:<port>"         # 모니터링 대상을 리스트로 지정
- job_name: "prometheus"
  static_configs:
  - targets:
    - "localhost:9090"
```

### PromQL로 지표 조회하기

Prometheus는 지표를 조회하기 위한 PromQL(Prometheus Query Language)이라는 쿼리 언어를 제공하고 있습니다.
SQL처럼 수집된 지표를 필터링 및 집계할 수 있으며, 또한, 가공하고 알림을 발생시키는데 사용할 수 있습니다.
그리고 중요한 점은 Prometheus의 모니터링 대상이 expose하는 지표 데이터는 이전 수집 시기로부터의 변화량이 아니라 앱 실행시부터 누적된 값입니다(gauge 타입 제외).
지표의 변화량은 서버에서 조회시 PromQL을 통해서 계산할 수 있으며, `rate`, `irate` 같은 함수가 그 역할을 수행합니다.
자세한 사항은 이 글의 범위는 넘어서므로 일단 PromQL 공식문서를 참고하거나 이미 PromQL이 작성되어있는 대시보드를 Grafana Marketplace에서 받아서 살펴보는 것을 권합니다.

#### PromQL 예제

- 특정 서비스내 인스턴스의 JVM Heap Eden 영역 조회

```
jvm_memory_used_bytes{instance="123.0.0.1:8080", application="my_api", id="G1 Eden Space"}
```

- 특정 서비스내 인스턴스의 API(Method + URL Path + Status Code)별 초 당 요청 수 조회
  - `count` 지표는 앱 시작시부터 계속 누적되는 값이기에 `irate` 함수를 사용하고 있습니다.

```
irate(http_server_requests_seconds_count{application="my_api", instance="123.0.0.1:8080"}[3m])
```

- 특정 서비스의 초 당 에러 요청 수 총 합계 조회

```
sum by (application) (irate(http_server_requests_seconds_count{application="my_api", outcome=~"CLIENT_ERROR|SERVER_ERROR"}[3m]))`
```

### Grafana에 대시보드와 차트 생성하기

Marketplace 원하는 대시보드가 없거나 import 받은 대시보드에 원하는 그래프가 없다면 직접 그래프를 생성할 수 있습니다.
이때 역시 PromQL이 사용됩니다.

참고로 각 대시보드에는 변수를 추가해서 사용할 수 있습니다.
상수로 지정할 수도 있지만 label_values라는 함수를 사용하여 Prometheus로부터 특정 레이블 목록을 가져올 수도 있습니다.
설정한 변수는 각 PromQL에서 `$variable_name` 같은 형태로 사용 가능합니다.
Grafana 대시보드 변수 매우 편한 기능이지만 변수 기능을 사용하면 Grafana Alert을 등록할 수 없다는 문제도 존재합니다.
이때는 Alert 전용 대시보드를 별도로 생성해서 사용하는 것을 권장합니다.

**패널 추가 버튼 클릭**
![add_graph_1.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_panel1.png)

**쿼리 추가 버튼 클릭**
![add_panel_2.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_panel2.png)

**PromQL 입력**

- 두 개 이상의 PromQL을 작성하여 그래프를 만들 수 있습니다.
- PromQL을 포함한 대부분의 필드 내에서  `$variable_name` 형태로 대시보드 변수를 사용할 수 있습니다.
- 범례명으로 PromQL 결과 데이터의 레이블을 사용할 수 있습니다. `{{ label_name }}`과 같은 형태로 사용합니다.
![add_panel_3.png]({{site.url}}/public/posts_images/monitoring_java_with_prometheus/add_panel3.png)

## 부록

1. **이미 Scouter 또는 Pinpoint를 사용하고 있는데, 그럼 필요없나요?**
그렇지 않습니다.
엄밀히 따지면 Prometheus와 Scouter(Pinpoint)는 역할이 다릅니다.
따라서 Java 애플리케이션을 모니터링하기 위해서는 Scouter와 Prometheus 둘 다 필요하다고 볼 수 있습니다.
Scouter가 Java 애플리케이션 특히 Tomcat 같은 웹 베이스 애플리케이션에 특화되어 있으며 각 요청에 대한 메소드 호출을 추적(tracing)할 수 있다면, Prometheus는 애플리케이션의 상태를 나타내는 각 지표들(metric)을 살펴보고 조회하는데 더 특화되어 있습니다.
JVM과 관련된(heap, GC 등) 지표들은 겹칠지 모르나 그 외에 사용하는 라이브러리의 지표나(Hystrix, Ribbon 등), 내부 구성 또는 로직과 관련된 지표(Memtable 상태, redis 사용률, 초당 Email 발송량 및 지연시간 등), 여러 목적으로 나누어져 관리되는 Thread/Connection Pool 등 보다 세부적으로 수집가능하며 사용자의 선택에 따라 더 추가/삭제될 수 있습니다.
원한다면 Micrometer 같은 도구를 사용하여 직접 지표를 생성하기에도 매우 간편합니다.
그리고 이러한 지표들을 PromQL을 통해서 입맛대로 조회 및 가공할 수 있습니다.
숲을 보는데에는 Prometheus가 좋고, 나무를 보는데에는(그리고 추적하는데) Scouter가 좋은 것 같습니다.

2. **엔드포인트 보안**
Prometheus는 수집 엔드포인트의 보안과 관련해서는 관여하지 않습니다.
이는 어디까지나 클라이언트의 책임으로 보고 있습니다.
사실 엔드포인트도 결국 HTTP API이기에 어떻게 보안을 할지는 사용자가 더 잘 알고 있을 것이라고 판단한 것 같습니다.
Spring Boot의 경우 Spring Boot Security를 사용할 수 있을 것이며, 보다 편리한 설정을 위해서 Spring Boot Actuator는 각 엔드포인트에 대한 RequestMatcher를 제공하고 있습니다.

3. **DisableExplicitGC**
만약 JMX을 사용하여 모니터링을 수행하고 내부적으로 `System.gc()`가 중요한 역할을 하지 않는다면 `-XDisableExplicitGC` JVM 옵션을 추가하는 것을 권합니다.
JMX에 접근하기 위해서 RMI포트를 사용한다면 JVM은 RMI에 사용된 메모리를 정리하기 위해서 주기적으로 강제적인 FullGC를 수행합니다.
기본 주기는 1시간이기 때문에 만약 이 옵션을 추가하지 않았다면 JVM의 힙메모리가 넉넉함에도 불구하고 1시간 단위로 주기적인 FullGC가 발생하는 모습을 보게될 것입니다(어떤 GC를 사용하느냐에 따라 다른 모습을 보일 수도 있습니다).

4. **배치 같이 짧게 실행되는 애플리케이션은 어떻게 해야하나요?**
짧은 배치성 작업을 모니터링하기 위해서 Prometheus는 Push Gateway를 제공합니다.
배 작업을 진행하면서 측정된 지표를 Push Gateway로 전송하여 모니터링할 수 있습니다.
단, Push Gateway를 배치성 작업 외에 다른 모든 지표를 Push 수집하는 용도로 사용해서는 안됩니다. 
Prometheus는 어디까지나 Pull 기반 플랫폼이며, Push Gateway는 대용량 트레픽 및 지속적인 시계열 데이터를 받도록 설계되지 않았습니다.

5. **클러스터링 및 고가용성**
안타깝게도 Prometheus는 따로 클러스터링을 구성할 수 없으며 고가용성도 보장하지 않습니다.
Federation이나 remote write/read 등의 기능을 통해서 보완을 할 수 있으나 여러 제한이 있습니다.
이와 관련해서 Prometheus 커뮤니티에서도 프로젝트를 진행하고 있습니다.
Service as a Prometheus를 목표로하는 두 가지 프로젝트가 진행되고 있습니다.
두 프로젝트는 목적은 같지만 방법은 많이 다릅니다.

    - Cortex: https://cortexmetrics.io/
    - Thanos: https://thanos.io/
