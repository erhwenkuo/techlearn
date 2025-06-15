# Worker options

> 了解建立 worker 可用的選項。

## WorkerOptions parameters

建立 worker 的介面是透過 `WorkerOptions` 類別實現的。以下僅包含部分可用參數。完整清單請參閱 [WorkerOptions 參考](https://docs.livekit.io/reference/python/v1/livekit/agents/index.html.md#livekit.agents.WorkerOptions)。

**Python**:

```python
opts = WorkerOptions(
    # 當作業被指派給此 Worker 時，會呼叫 entrypoint 函數
    # 這是 WorkerOptions 中唯一必要的參數
    # https://docs.livekit.io/agents/worker/job/#entrypoint
    entrypoint_fnc,

    # 檢查請求並決定目前 worker 是否應該處理它。
    request_fnc,

    # 一個在新進程啟動時需要被執行的必要初始化的函數。
    prewarm_fnc,

    # 一個收集目前系統負載（CPU 或 RAM 等）的 callback 函數。
    load_fnc,

    # load_fnc 的最大值，超過該值將不會產生新的 job 流程
    load_threshold,

    # 代理是否可以訂閱軌道、發布資料、更新元資料等。
    permissions,

    # 收到 SIGTERM 或 SIGINT 時等待現有作業完成的時間
    drain_timeout,

    # 要建立的工作者類型，JT_ROOM 或 JT_PUBLISHER
    worker_type=WorkerType.ROOM,
)

# start the worker
cli.run_app(opts)

```

!!! warn
    🔥 **Caution**

    出於安全目的，將 LiveKit API 金鑰和機密設定為環境變量，而不是 `WorkerOptions` 參數。

### Entrypoint

`entrypoint_fnc` 是每個新作業呼叫的主函數，也是代理應用的核心。要了解更多信息，請參閱作業生命週期文章中的 [entrypoint 文件](https://docs.livekit.io/agents/worker/job.md#entrypoint)。


**Python**:

```python
async def entrypoint(ctx: JobContext):
    # connect to the room
    await ctx.connect()
    # handle the session
    ...

```

### Request handler

每當 LiveKit 伺服器有代理程式的作業時，都會執行 `request_fnc` 函數。框架要求 worker 明確接受或拒絕每個 job request。如果接受請求，則會呼叫 `entrypoint_fnc` 函數。如果請求被拒絕，則會將其傳送到下一個可用的 worker。

{==預設情況下，如果留空，則行為是自動接受所有傳送給工作器的請求。==}

**Python**:

```python
async def request_fnc(req: JobRequest):
    # accept the job request
    await req.accept(
        # the agent's name (Participant.name), defaults to ""
        name="agent",
        # the agent's identity (Participant.identity), defaults to "agent-<jobid>"
        identity="identity",
        # attributes to set on the agent participant upon join
        attributes={"myagent": "rocks"},
    )

    # or reject it
    # await req.reject()

opts = WorkerOptions(entrypoint_fnc=entrypoint, request_fnc=request_fnc)

```

### Prewarm function

出於隔離和效能方面的考慮，agent fraemwork 會在每個代理作業的獨立進程中運行它們。代理通常需要存取需要較長時間載入的模型檔案。為了解決這個問題，您可以使用 `prewarm` 函數在向其分配任何作業之前預熱該進程。您可以使用 `num_idle_processes` 參數控制需要預熱的進程數量。

**Python**:

```python
def prewarm_fnc(proc: JobProcess):
    # 載入 silero 權重並儲存以處理使用者數據
    proc.userdata["vad"] = silero.VAD.load()


async def entrypoint(ctx: JobContext):
    # 存取已載入的 silero 實例
    vad: silero.VAD = ctx.proc.userdata["vad"]

opts = WorkerOptions(entrypoint_fnc=entrypoint, prewarm_fnc=prewarm_fnc)

```

### Load function

`load_fnc` 和 `load_threshold` 控制 worker 何時停止接受新 jobs。

- `load_fnc`: 以 0 到 1.0 之間的浮點數傳回 worker 目前負載的函數。
- `load_threshold`: Worker 仍能接受新工作的最大負載值。


如果未提供 load_fnc ，則預設使用 worker 在 5 秒視窗內的平均 CPU 使用率。

以下範例展示如何定義一個自訂負載函數，將工作器限制為 9 個併發作業，且不受 CPU 使用率的影響：


```python
from livekit.agents import Worker, WorkerOptions

def compute_load(worker: Worker) -> float:
    return min(len(worker.active_jobs) / 10, 1.0)

opts = WorkerOptions(
    load_fnc=compute_load,
    load_threshold=0.9,
)

```

### Drain timeout

由於 agent sessions 是有狀態的，因此在進程關閉時不應突然終止它們。代理框架支援優雅終止：當收到 `SIGTERM` 或 `SIGINT` 訊號時，worker 將進入 `draining` 狀態。在此狀態下，它會停止接受新任務，但允許現有任務完成，最長可達配置的逾時時間。

`drain_timeout` 參數設定等待活動作業完成的最長時間。預設值為 30 分鐘。

### Permissions

預設情況下，代理可以向同一房間內的其他參與者發布訊息，也可以向他們訂閱訊息。不過，您可以透過設定 `WorkerOptions` 中的 `permissions` 參數來自訂這些權限。要查看完整的參數列表，請參閱 [WorkerPermissions 參考](https://docs.livekit.io/reference/python/v1/livekit/agents/index.html.md#livekit.agents.WorkerPermissions)。

**Python**:

```python
opts = WorkerOptions(
    ...
    permissions=WorkerPermissions(
        can_publish=True,
        can_subscribe=True,
        can_publish_data=True,
        # 設定為 true 時，房間中的其他人將看不到該代理。
        # 隱藏時，它也無法將軌跡發佈到房間，因為它不可見。
        hidden=False,
    ),
)

```

### Worker type

您可以選擇為每個房間或房間內的每個發布者啟動一個新的 agent 實例。您可以在註冊 worker 時進行設定：

**Python**:

```python
opts = WorkerOptions(
    ...
    # when omitted, the default is WorkerType.ROOM
    worker_type=WorkerType.ROOM,
)

```

---

**Node.js**:

```ts
const opts = new WorkerOptions({
  // when omitted, the default is JobType.JT_ROOM
  workerType: JobType.JT_ROOM,
});

```

`WorkerType` 枚舉有兩個選項：
- `ROOM`: 為每個房間建立一個新的代理實例。
- `PUBLISHER`: 為房間中的每個發布者建立一個新的代理實例。

如果代理程式在可能包含多個發布者的房間中執行資源密集型操作（例如，處理來自一組安全攝影機的傳入視訊），則可以將 `worker_type` 設定為 `JT_PUBLISHER`，以確保每個發布者都有自己的代理實例。

對於 `PUBLISHER` jobs，為房間中的每個發布者呼叫一次 `entrypoint` 函數。 `JobContext.publisher` 物件包含一個代表該發布者的 `RemoteParticipant`。

## Starting the worker

若要使用 `WorkerOptions` 定義的設定啟動 worker，請呼叫 CLI：

**Python**:

```python
if __name__ == "__main__":
    cli.run_app(opts)

```

---

**Node.js**:

```ts
cli.runApp(opts);

```

Agents 工作器 CLI 提供了兩個子指令: `start` 和 `dev`。前者將原始 JSON 資料輸出到標準輸出，建議用於生產環境。而 `dev` 則建議用於開發環境，因為它輸出人性化的彩色日誌，並且支援 Python 的熱重載。