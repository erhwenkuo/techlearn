# Worker lifecycle

> Worker 如何與 LiveKit 伺服器協調來管理 agent jobs。

## Overview

當您使用 `python agent.py dev` 啟動您的應用程式時，它會將自己註冊為 LiveKit 伺服器的 **worker**。 LiveKit 伺服器透過向可用的 **worker** 發送請求來管理將您的代理程式調度到有使用者的房間。

**LiveKit session** 是指某個 [room](https://docs.livekit.io/home/get-started/api-primitives.md#room) 中的一個或多個參與者。 LiveKit 會話通常簡稱為 `room`。當使用者連接到房間時，**worker** 會滿足請求，派遣代理到房間。


**Worker** 生命週期概述如下：

1. **Worker registration**: 您的代理程式碼將自行註冊為 LiveKit 伺服器的 `worker`，然後等待請求。
2. **Job request**: 當使用者連接到房間時，LiveKit 伺服器會向可用的 **worker** 發送請求。一個 **worker** 接受並開始一個新流程來處理這項工作。這也稱為[agent dispatch](https://docs.livekit.io/agents/worker/agent-dispatch.md)。
3. **Job**: 由你的 `entrypoint` 函數發起的作業。這是您編寫的大部分程式碼和邏輯。要了解更多信息，請參閱[Job lifecycle](https://docs.livekit.io/agents/worker/job.md)。
4. **LiveKit session close**: 預設情況下，當最後一位非代理參與者離開時，房間會自動關閉。任何剩餘的代理均斷開連接。您也可以手動[end the session](https://docs.livekit.io/agents/worker/job.md#ending-the-session)。

下圖展示了 **worker** 的生命週期：

![Diagram describing the functionality of agent workers](./assets/agents-jobs-overview2.svg)

**Worker** 的一些附加功能包括：

- Workers 會自動與 LiveKit 伺服器交換可用性和容量訊息，從而實現傳入請求的負載平衡。
- 每個 worker 可以同時執行多個作業，每個作業在自己的進程中運行以實現隔離。如果一個進程崩潰，它不會影響在同一進程上運行的其他進程。
- 當您部署更新時，worker 會在關閉之前正常地耗盡活動的 LiveKit 會話，確保通話過程中不會中斷任何會話。

## Worker options

您可以透過 [WorkerOptions](https://docs.livekit.io/agents/worker/options.md) 變更權限、調度規則、新增預熱功能等。
