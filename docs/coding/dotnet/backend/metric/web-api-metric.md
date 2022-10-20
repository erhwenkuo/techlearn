# 儀器化 ASP.NET Core 應用程序

原文: https://blog.csdn.net/sD7O95O/article/details/124009720

## 引入函式庫

[prometheus-net](https://github.com/prometheus-net/prometheus-net#aspnet-core-http-request-metrics) 是一個 .NET 庫，用於檢測應用程序並將指標暴露出來給 Prometheus 捉取。

該函式庫允許使用自定義指標來檢測代碼，並為 ASP.NET Core 提供一些內置的指標集合集成。

```bash
dotnet add package prometheus-net.AspNetCore
```

修改

```csharp title="Program.cs" hl_lines="3 35 36"
using Microsoft.EntityFrameworkCore;
using TodoApi.Models;
using Prometheus;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.

builder.Services.AddControllers();
builder.Services.AddDbContext<TodoContext>(opt =>
    opt.UseInMemoryDatabase("TodoList"));

// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseSwagger();
app.UseSwaggerUI();

// remark below line so dotnet api will allow using http instead of https
//app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

app.MapMetrics();
app.UseHttpMetrics();

app.Run();
```

執行下列命令來運行應用程序：

```bash
dotnet run
```

啟動成功後在瀏覽器中，導航到 https://localhost:8080/swagger 。

![](./assets/swagger-ui.png)

接著切換到 https://localhost:8080/metrics 。

![](./assets/metrics-endpoint.png)

## 容器化 ASP.NET Application


### 創建 Dockerfile

從您的終端，運行以下命令：

```
docker build -t witlab/dotnet-todoapi:metric -f Dockerfile .
```

Docker 將處理 Dockerfile 中的每一行指令。 在 `docker build` 命令中設置鏡像的構建上下文。 `-f` 旗標指向 Dockerfile 的路徑。此命令構建映像並創建一個名為 donet-todoapi 的本地存儲庫，該存儲庫指向該映像。此命令完成後，運行 `docker images` 以查看已安裝的容器鏡像列表：

```bash
$ docker images

REPOSITORY                        TAG       IMAGE ID       CREATED         SIZE
witlab/dotnet-todoapi             metric    43e06ecd29a9   5 seconds ago   275MB
```

### 推送容器鏡像到 Dockerhub

要把容器鏡像推到 Dockerhub 的前題是要先到 Docker Hub 上註冊帳號，本教程假設大家己經都有了帳號。

首先使用 `docker login` 指令登入到 Docker Hub:

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

使用下列命令來推送容器鏡像:

```bash
$ docker push witlab/dotnet-todoapi:metric

The push refers to repository [docker.io/witlab/dotnet-todoapi]
6007b133d0ec: Pushed 
16ab2fb3fd71: Pushed 
efccb7d95dee: Pushed 
64d665f70cc1: Pushed 
e487c4dad54c: Pushed 
5ec686cbc3c7: Pushed 
b45078e74ec9: Pushed 
base: digest: sha256:54322ab39ce67b3d7a728f66935c3c4797b00c9c21bc9e71e457f94465c5aaf5 size: 1789
```

成功之後到 Docker Hub 去檢查看看容器鏡像是否己經成功上傳:

![](./assets/upload_to_dockerhub.png)

點選　"witlab/dotnet-toddapi" 之後再選擇 "Tags" 頁籤:

![](./assets/upload_to_dockerhub2.png)

應該看到有一個鏡像的標籤是 `metrics`。

使用 `docker rm` 指令把 local 的 Image 刪除掉:

```bash
$ docker rmi -f witlab/dotnet-todoapi:metric
```

測試從 Docker Hub pull Docker Image 下來，指令如下:

```bash
$ docker pull witlab/dotnet-todoapi:metric
```

啟動 Docker Container，指令如下:

```bash
docker run -it --rm -p 8080:8080 witlab/dotnet-todoapi:metric
```

啟動完成之後就可以使用 Browser 查看結果，輸入 URL 位址為 http://localhost:8080/swagger 可以看到如下畫面:

![](./assets/swagger-ui3.png)
