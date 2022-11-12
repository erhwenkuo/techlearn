# 儀器化 Sprintboot 應用程序

原文:[Spring Boot 使用 Micrometer 集成 Prometheus 监控 Java 应用性能](https://blog.csdn.net/aixiaoyang168/article/details/100866159)

原文:[Monitoring and Profiling Your Spring Boot Application](https://dzone.com/articles/monitoring-and-profiling-spring-boot-application)

![](./assets/java-performance.png)

## Micrometer

Micrometer 為 Java 平台上的性能數據收集提供了一個通用的 API，它提供了多種度量指標類型（Timers、Guauges、Counters等），同時支持接入不同的監控系統，例如 Influxdb、Graphite、Prometheus 等。我們可以通過 Micrometer 收集 Java 性能數據，配合 Prometheus 監控系統實時獲取數據，並最終在 Grafana 上展示出來，從而很容易實現應用的監控。

Micrometer 中有兩個最核心的概念，分別是計量器（Meter）和計量器註冊表（MeterRegistry）。計量器用來收集不同類型的性能指標信息，Micrometer 提供瞭如下幾種不同類型的計量器：

- 計數器 (Counter): 表示收集的數據是按照某個趨勢（增加／減少）一直變化的，也是最常用的一種計量器，例如接口請求總數、請求錯誤總數、隊列數量變化等。
- 計量儀 (Gauge): 表示蒐集的瞬時的數據，可以任意變化的，例如常用的 CPU Load、Mem 使用量、Network 使用量、實時在線人數統計等，
- 計時器 (Timer): 用來記錄事件的持續時間，這個用的比較少。
- 分佈概要 (Distribution summary): 用來記錄事件的分佈情況，表示一段時間範圍內對數據進行採樣，可以用於統計網絡請求平均延遲、請求延遲佔比等。

這裡我重點說一下 DistributionSummary。

### DistributionSummary

DistributionSummary 是用於跟踪事件的分佈情況，有多個指標組成：

- `count`: 事件的個數，聚合指標，如響應的個數
- `sum`: 綜合，聚合指標，如響應大小的綜合
- `histogram`: 分佈，聚合指標，包含 `le` 標籤用於區分 bucket
    - 例如 web.response.size.historgram {le=512} = 99, 表示響應大小不超過 512（Byte) 的響應個數是 99 個。
    - 一般有多個 bucket，如 le=128，le=256，le=512，le=1024, le=+Inf 等。每個 bucket 為一條時間序列

## 引入函式庫

要讓 Micrometer 整合進 Sprintboot，我們只需下列兩個依賴項添加到我們的 pom.xml 中：

```xml title="pom.xml"
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <dependencies>
        ...
        ...
        <!-- actuator for monitoring  -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>

		<!-- micrometer for metrics  -->
		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-registry-prometheus</artifactId>
			<version>1.10.0</version>
		</dependency>
    </dependencies>
    ...
    ...
</project>
```

這裡引入了 io.micrometer 的 `micrometer-registry-prometheus` 依賴以及 `spring-boot-starter-actuator` 依賴，因為該包對 Prometheus 進行了封裝，可以很方便的集成到 Spring Boot 工程中。

## 修改啟動配置

其次在 application.properties 中配置如下：

``` title="application.properties"
# Enable prometheus exporter
management.metrics.export.prometheus.enabled=true

# Enable prometheus end point
management.endpoints.web.exposure.include=prometheus

# enable percentile-based histogram for http requests
management.metrics.distribution.percentiles-histogram.http.server.requests=true

# http SLA histogram buckets
management.metrics.distribution.sla.http.server.requests=100ms,150ms,250ms,500ms,1s

# enable quantile setup
management.metrics.distribution.percentiles.http.server.requests=0.5,0.9,0.95,0.99,0.999

# enable JVM metrics
management.metrics.enable.jvm=true
```

下面說明幾個比較關鍵的設定:

- `management.metrics.export.prometheus.enabled=true`: 添加此行後，Micrometer 將開始累積有關應用程序的數據，並且可以通過訪問 `actuator/prometheus` 端點來查看此數據。
- `management.endpoints.web.exposure.include=prometheus`: 即使啟動了上面的設定，我們也無法瀏覽 Prometheus 端點，因為默認情況下這是禁用的，因此我們需要配置來使用管理端點，將 `prometheus` 包含在列表中。
- `management.metrics.distribution.percentiles-histogram.http.server.requests=true`:  啟動 Distribution summary 的計算。

## 測試專案執行

執行下列命令來運行應用程序：

如果您使用 Gradle，則可以使用 `./gradlew bootRun` 運行應用程序。

如果您使用 Maven，則可以使用 `./mvnw spring-boot:run` 運行應用程序。

使用瀏覽器來連接到 Swagger UI 來進行簡易測試 `http://localhost:8080/swagger`:

![](./assets/swagger-ui.png)


接著切換到 `https://localhost:8080/actuator/prometheus`:

![](./assets/java-sprintboot-metrics.png)


## 容器化 Sprintboot 應用

### 創建 Dockerfile

`docker build` 命令使用 **Dockerfile** 文件來創建容器映像。該文件是一個名為 Dockerfile 的文本文件，沒有擴展名。

在 Sprintboot 專案根目錄中創建一個名為 Dockerfile 的文件，然後在文本編輯器中打開它。

```docker title="Dockerfile"
FROM maven:3-jdk-11 AS build-env
WORKDIR /opt

# Copy everything
COPY . /opt

# Just echo so we can see, if everything is there :)
RUN ls -l

# Run Maven build
RUN mvn clean install

# Build runtime image
FROM adoptopenjdk/openjdk11:alpine-jre
WORKDIR /opt
COPY --from=build-env /opt/target/todoapi-0.0.1-SNAPSHOT.jar /opt/app.jar

# Expose web api port number
EXPOSE 8080

# java -jar /opt/app/app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

從您的終端，運行以下命令：

```bash
docker build -t java-todoapi:metric -f Dockerfile .
```

Docker 將處理 Dockerfile 中的每一行指令。 在 `docker build` 命令中設置鏡像的構建上下文。 `-f` 旗標指向 Dockerfile 的路徑。

此命令構建映像並創建一個名為 `java-todoapi` 的本地存儲庫，該存儲庫指向該映像。

此命令完成後，運行 `docker images` 以查看已安裝的容器鏡像列表：

```bash
$ docker images

REPOSITORY               TAG            IMAGE ID       CREATED         SIZE
java-todoapi             metric         5cdb0dcedfd2   6 minutes ago   196MB
```

### 啟動 Docker 容器

端口映射用於訪問在 Docker 容器內運行的服務。我們打開一個主機端口，讓我們可以訪問 Docker 容器內相應的開放端口。然後所有對主機端口的請求都可以重定向到 Docker 容器中。

```bash
docker run -it --rm -p 8080:8080 java-todoapi:metric
```

### 推送容器鏡像到 Dockerhub

要把容器鏡像推到 Dockerhub 的前題是要先到 Docker Hub 上註冊帳號，本教程假設大家己經都有了帳號。

首先使用 docker login 指令登入到 Docker Hub:

```bash
$ docker login

Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: witlab
Password: 
WARNING! Your password will be stored unencrypted in /home/dxlab/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

要把 Docker Image Push 到 Docker Hub 上，需要把 Docker Image 加上 Dockerhub 的 Username，如下指令:

```bash
$ docker build -t witlab/java-todoapi:metric -f Dockerfile .
```

接著使用下列命令來推送容器鏡像:

```bash
$ docker push witlab/java-todoapi:metric

The push refers to repository [docker.io/witlab/java-todoapi]
6ef9281f1969: Pushed 
0aa20a85da26: Layer already exists 
b61e498b0317: Layer already exists 
63493a9ab2d4: Layer already exists 
metric: digest: sha256:dfb646cdc0dbc36c201c5838ebef773b1690b25f11cd8f36844b25dcb0187ce8 size: 1163
```

成功之後到 Docker Hub 去檢查看看容器鏡像是否己經成功上傳:

![](./assets/docker-hub.png)


點選　"witlab/java-toddapi" 之後再選擇 "Tags" 頁籤:

![](./assets/docker-hub-different-tags.png)

應該看到有一個鏡像的標籤是 metrics。


使用 `docker rm` 指令把 local 的 Image 刪除掉:

```bash
$ docker rmi -f witlab/java-todoapi:metric
```

測試從 Docker Hub pull Docker Image 下來，指令如下:

```bash
$ docker pull witlab/java-todoapi:metric
```

啟動 Docker Container，指令如下:

```bash
docker run -it --rm -p 8080:8080 witlab/java-todoapi:metric
```

啟動完成之後就可以使用 Browser 查看結果，輸入 URL 位址為 http://localhost:8080/swagger 可以看到如下畫面:

![](./assets/swagger-ui.png)
