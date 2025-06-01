# Agent dispatch

> 指定如何以及何時將您的代理分配到 LiveKit 房間。

## Dispatching agents

調度是將代理分配到房間的過程。LiveKit 伺服器將此流程作為 [worker 生命週期](https://docs.livekit.io/agents/worker.md) 的一部分進行管理。 LiveKit 針對高並發性和低延遲性進行了調度最佳化，通常支援每秒數十萬個新連接，最大調度時間低於 150 毫秒。


## Automatic agent dispatch

預設情況下，會自動派遣一名代理到每個新房間。如果您想為所有新參與者指派同一個代理，自動排程是最佳選擇。

## Explicit agent dispatch

可以透過明確的調度來更好地控制代理何時以及如何加入房間。這種方法利用相同的工作系統，讓您以相同的方式運行 agent workers。

若要使用顯式調度，請在 `WorkerOptions` 中設定 `agent_name` 欄位：

=== "Python"

    ```python
    opts = WorkerOptions(
        ...
        agent_name="test-agent",
    )

    ```

=== "Node.js"

    ```ts
    const opts = new WorkerOptions({
    ...
    agentName: "test-agent",
    });

    ```

!!! tip
    ❗ **Important**

    如果設定了 `agent_name` 屬性，則自動調度將被停用。

### Dispatch via API

可以透過 `AgentDispatchService` 將設定了 `agent_name` 的代理工作者明確地調度到某個房間。

=== "Python"

    ```python
    import asyncio
    from livekit import api

    room_name = "my-room"
    agent_name = "test-agent"

    async def create_explicit_dispatch():
        lkapi = api.LiveKitAPI()
        dispatch = await lkapi.agent_dispatch.create_dispatch(
            api.CreateAgentDispatchRequest(
                agent_name=agent_name, room=room_name, metadata='{"user_id": "12345"}'
            )
        )
        print("created dispatch", dispatch)

        dispatches = await lkapi.agent_dispatch.list_dispatch(room_name=room_name)
        print(f"there are {len(dispatches)} dispatches in {room_name}")
        await lkapi.aclose()

    asyncio.run(create_explicit_dispatch())

    ```

=== "Node.js"

    ```ts
    import { AgentDispatchClient } from 'livekit-server-sdk';

    const roomName = 'my-room';
    const agentName = 'test-agent';

    async function createExplicitDispatch() {
    const agentDispatchClient = new AgentDispatchClient(process.env.LIVEKIT_URL, process.env.LIVEKIT_API_KEY, process.env.LIVEKIT_API_SECRET);

    // create a dispatch request for an agent named "test-agent" to join "my-room"
    const dispatch = await agentDispatchClient.createDispatch(roomName, agentName, {
        metadata: '{"user_id": "12345"}',
    });
    console.log('created dispatch', dispatch);

    const dispatches = await agentDispatchClient.listDispatch(roomName);
    console.log(`there are ${dispatches.length} dispatches in ${roomName}`);
    }

    ```

=== "LiveKit CLI"

    ```shell
    lk dispatch create \
    --agent-name test-agent \
    --room my-room \
    --metadata '{"user_id": "12345"}'

    ```

=== "Go"

    ```go
    func createAgentDispatch() {
        req := &livekit.CreateAgentDispatchRequest{
            Room:      "my-room",
            AgentName: "test-agent",
            Metadata:  "{\"user_id\": \"12345\"}",
        }
        dispatch, err := dispatchClient.CreateDispatch(context.Background(), req)
        if err != nil {
            panic(err)
        }
        fmt.Printf("Dispatch created: %v\n", dispatch)
    }

    ```

如果房間 `my-room` 尚不存在，則在調度過程中會自動建立房間，並且 worker 會將 `test-agent` 指派給該房間。

#### Job metadata

明確調度可讓您將元資料傳遞給代理，可在 `JobContext` 中使用。這對於包含用戶 ID、姓名或電話號碼等詳細資訊很有用。

元資料(metadata)欄位是一個字串。 LiveKit 建議使用 JSON 傳遞結構化資料。

上一節的 [examples](#via-api) 示範如何在排程期間傳遞作業元資料(job metadata)。

有關在代理程式中使用作業元資料(job metadata)的信息，請參閱以下指南：

- **[Job metadata](https://docs.livekit.io/agents/worker/job.md#metadata)**: 了解如何在代理程式中使用作業元資料。

### Dispatch from inbound SIP calls

可以明確調度代理來接聽入站 SIP 呼叫(inbound SIP calls)。 [SIP dispatch rules](https://docs.livekit.io/sip/dispatch-rule.md) 可以使用 `room_config.agents` 欄位定義一個或多個代理程式。

LiveKit 建議對 SIP 入站呼叫進行明確代理調度，而不是自動代理調度，因為它允許單一專案內有多個代理。

### Dispatch on participant connection

您可以配置參與者的 token 以在連接時立即調度一個或多個代理。

若要調度多個代理，請在 `RoomConfiguration` 中包含多個 `RoomAgentDispatch` 條目。

以下範例建立一個令牌，當參與者連線時，該令牌將 `test-agent` 代理程式調度到 `my-room` 房間：

=== "Python"

    ```python
    from livekit.api import (
    AccessToken,
    RoomAgentDispatch,
    RoomConfiguration,
    VideoGrants,
    )

    room_name = "my-room"
    agent_name = "test-agent"

    def create_token_with_agent_dispatch() -> str:
        token = (
            AccessToken()
            .with_identity("my_participant")
            .with_grants(VideoGrants(room_join=True, room=room_name))
            .with_room_config(
                RoomConfiguration(
                    agents=[
                        RoomAgentDispatch(agent_name="test-agent", metadata='{"user_id": "12345"}')
                    ],
                ),
            )
            .to_jwt()
        )
        return token

    ```

=== "Node.js"

    ```ts
    import { RoomAgentDispatch, RoomConfiguration } from '@livekit/protocol';
    import { AccessToken } from 'livekit-server-sdk';

    const roomName = 'my-room';
    const agentName = 'test-agent';

    async function createTokenWithAgentDispatch(): Promise<string> {
    const at = new AccessToken();
    at.identity = 'my-participant';
    at.addGrant({ roomJoin: true, room: roomName });
    at.roomConfig = new RoomConfiguration({
        agents: [
        new RoomAgentDispatch({
            agentName: agentName,
            metadata: '{"user_id": "12345"}',
        }),
        ],
    });
    return await at.toJwt();
    }

    ```

=== "Go"

    ```go
    func createTokenWithAgentDispatch() (string, error) {
        at := auth.NewAccessToken(
            os.Getenv("LIVEKIT_API_KEY"),
            os.Getenv("LIVEKIT_API_SECRET"),
        ).
            SetIdentity("my-participant").
            SetName("Participant Name").
            SetVideoGrant(&auth.VideoGrant{
                Room:     "my-room",
                RoomJoin: true,
            }).
            SetRoomConfig(&livekit.RoomConfiguration{
                Agents: []*livekit.RoomAgentDispatch{
                    {
                        AgentName: "test-agent",
                        Metadata:  "{\"user_id\": \"12345\"}",
                    },
                },
            })

        return at.ToJWT()
    }

    ```
