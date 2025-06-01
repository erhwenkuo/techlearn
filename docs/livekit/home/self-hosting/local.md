# Running LiveKit locally

> 這將在本地的伺服器上啟動並運行 LiveKit 實例，準備接收來自參與者的音訊和視訊串流。

### Install LiveKit Server

=== "Linux"

    ```text
    curl -sSL https://get.livekit.io | bash

    ```

=== "macOS"

    ```text
    brew update && brew install livekit

    ```

=== "Windows"

    從[此處](https://github.com/livekit/livekit/releases/latest)下載最新版本。

### Start the server in `dev` mode

您可以透過執行以下命令以開發模式啟動 LiveKit：

```text
livekit-server --dev

```

這將使用以下 API 金鑰/密鑰對啟動一個實例：

```text
API key: devkey
API secret: secret

```

若要自訂生產設置，請參閱我們的 [deployment guides](https://docs.livekit.io/home/self-hosting/deployment/)。

!!! tip
    預設情況下，LiveKit 的訊號伺服器綁定到 `127.0.0.1:7880`。如果您想從網路上的其他裝置存取它，請傳入 `--bind 0.0.0.0`

### livekit-server command

`livekit-server` 是在 terminal 啟動 LiveKit 實例的命令。

用法如下:

```bash
livekit-server [global options] command [command options]
```

**global options**

| Option | Description | Example |
|--------|-------------|---------|
|`--bind value` | LiveKit 實例的 IP 位址，可使用 flags 來指定多個位址 | `0.0.0.0:7880` |
|`--config value` |LiveKit 設定檔的路徑, 範例設定檔可參考: [config-sample.yaml](https://github.com/livekit/livekit/blob/master/config-sample.yaml) | `--config config.yaml`|
|`--config-body value` | YAML 格式的 LiveKit 配置，通常會作為容器中的環境變數傳入 [$LIVEKIT_CONFIG]| |
|`--key-file value` | 包含LiveKit API 金鑰/機密的檔案路徑 | `--key-file keys.yaml` |
|`--keys value` | API 金鑰（key：secret\n）[$LIVEKIT_KEYS] | `--keys devkey:secret` |
|`--region value` | 當前節點的區域。由區域感知節點選擇器 [$LIVEKIT_REGION] 使用 | `--region us-west-2` |
|`--node-ip value` | 目前節點的IP位址，用於向客戶端通告。預設自動確定 [$NODE_IP] | `--node-ip 10.25.44.88`|
|`--udp-port value` | 用於 WebRTC 流量的 UDP 連接埠 [$UDP_PORT] | |
|`--redis-host value` | 主機（包括連接埠）到 redis 伺服器 [$REDIS_HOST] | `--redis-host 10.25.44.78:6379` |
|`--redis-password value` | redis 密碼 [$REDIS_PASSWORD] | `--redis-password 5488` |
| `--turn-cert value` | TURN 伺服器的 tls 憑證檔案 [$LIVEKIT_TURN_CERT] | |
|`--turn-key value` | TURN 伺服器的 tls 金鑰檔案 [$LIVEKIT_TURN_KEY] | |
|`--memprofile file` | 將記憶體設定檔寫入文件 | |
|`--dev` | 將日誌等級設定為偵錯、控制台格式化程式和 `/debug/pprof`。對於生產來說不安全（預設值：false）, 但方便開發時使用。| |
|`--help, -h` | 顯示幫助資訊 | |
|`--version, -v` | 列印版本 | |


**command**

| Command | Description |
|--------|-------------|
| `generate-keys` | 產生 API 金鑰和秘密對 |
| `ports` | 列印伺服器配置使用的連接埠 |
| `list-nodes` | 列出 LiveKit 實例節點清單 |
| `help-verbose` | 列印 LiveKit 所有的幫助資訊，包括所有生成的配置 flags |
| `help` | 顯示命令清單或某個命令的幫助 |


**配置檔範例**

```yaml title="config-sample.yaml"
# Copyright 2024 LiveKit, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# RoomService 和 RTC 端點的主要 TCP 連接埠
# 對於生產設置，此連接埠應放置在具有 TLS 的負載平衡器後面

port: 7880

# 當設定了 redis 時，LiveKit 將自動以完全分散的方式運行
# 客戶端可以連接到任何節點並被路由到同一個房間

redis:
  address: redis.host:6379
  # db: 0
  # username: myuser
  # password: mypassword
  # To use sentinel remove the address key above and add the following
  # sentinel_master_name: livekit
  # sentinel_addresses:
  # - livekit-redis-node-0.livekit-redis-headless:26379
  # - livekit-redis-node-1.livekit-redis-headless:26379
  # If you use a different set of credentials for sentinel add
  # sentinel_username: user
  # sentinel_password: pass
  #
  # To use TLS with redis
  # tls:
  #   enabled: true
  #   # when set to true, LiveKit will not verify the server's certificate, defaults to true
  #   insecure: false
  #   server_name: myserver.com
  #   # file containing trusted root certificates for verification
  #   ca_cert_file: /path/to/ca.crt
  #   client_cert_file: /path/to/client.crt
  #   client_key_file: /path/to/client.key
  #
  # To use cluster remove the address key above and add the following
  # cluster_addresses:
  # - livekit-redis-node-0.livekit-redis-headless:6379
  # - livekit-redis-node-1.livekit-redis-headless:6380
  # And it will use the password key above as cluster password
  # And the db key will not be used due to cluster mode not support it.

# WebRTC 配置

rtc:
  # UDP ports to use for client traffic.
  # this port range should be open for inbound traffic on the firewall
  port_range_start: 50000
  port_range_end: 60000
  # when set, LiveKit enable WebRTC ICE over TCP when UDP isn't available
  # this port *cannot* be behind load balancer or TLS, and must be exposed on the node
  # WebRTC transports are encrypted and do not require additional encryption
  # only 80/443 on public IP are allowed if less than 1024
  tcp_port: 7881
  # when set to true, attempts to discover the host's public IP via STUN
  # this is useful for cloud environments such as AWS & Google where hosts have an internal IP
  # that maps to an external one
  use_external_ip: true
  # # when set, LiveKit will attempt to use a UDP mux so all UDP traffic goes through
  # # listed port(s). To maximize system performance, we recommend using a range of ports
  # # greater or equal to the number of vCPUs on the machine.
  # # port_range_start & end must not be set for this config to take effect
  # udp_port: 7882-7892
  # # when set to true, server will use a lite ice agent, that will speed up ice connection, but
  # # might cause connect issue if server running behind NAT.
  # use_ice_lite: true
  # # optional STUN servers for LiveKit clients to use. Clients will be configured to use these STUN servers automatically.
  # # by default LiveKit clients use Google's public STUN servers
  # stun_servers:
  #   - server1
  # # optional TURN servers for clients. This isn't necessary if using embedded TURN server (see below).
  # turn_servers:
  #   - host: myhost.com
  #     port: 443
  #     # tls, tcp, or udp
  #     protocol: tls
  #     username: ""
  #     credential: ""
  # # allows LiveKit to monitor congestion when sending streams and automatically
  # # manage bandwidth utilization to avoid congestion/loss. Enabled by default
  # congestion_control:
  #   enabled: true
  #   # in the unlikely event of highly congested networks, SFU may choose to pause some tracks
  #   # in order to allow others to stream smoothly. You can disable this behavior here
  #   allow_pause: true
  # # allows automatic connection fallback to TCP and TURN/TLS (if configured) when UDP has been unstable, default true
  # allow_tcp_fallback: true
  # # number of packets to buffer in the SFU for video, defaults to 500
  # packet_buffer_size_video: 500
  # # number of packets to buffer in the SFU for audio, defaults to 200
  # packet_buffer_size_audio: 200
  # # minimum amount of time between pli/fir rtcp packets being sent to an individual
  # # producer. Increasing these times can lead to longer black screens when new participants join,
  # # while reducing them can lead to higher stream bitrate.
  # pli_throttle:
  #   low_quality: 500ms
  #   mid_quality: 1s
  #   high_quality: 1s
  # # when set, Livekit will collect loopback candidates, it is useful for some VM have public address mapped to its loopback interface.
  # enable_loopback_candidate: true
  # # network interface filter. If the machine has more than one network interface and you'd like it to use or skip specific interfaces
  # # both inclusion and exclusion filters can be used together. If neither is defined (default), all interfaces on the machine will be used.
  # # If both of them are set, then only include takes effect.
  # interfaces:
  #   includes:
  #     - en0
  #   excludes:
  #     - docker0
  # # ip address filter. If the machine has more than one ip address and you'd like it to use or skip specific ips,
  # # both inclusion and exclusion CIDR filters can be used together. If neither is defined (default), all ip on the machine will be used.
  # # If both of them are set, then only include takes effect.
  # ips:
  #   includes:
  #     - 10.0.0.0/16
  #   excludes:
  #     - 192.168.1.0/24
  # # Set to true to enable mDNS name candidate. This should be left disabled for most users.
  # # when enabled, it will impact performance since each PeerConnection will process the same mDNS message independently
  # use_mdns: true
  # # Set to false to disable strict ACKs for peer connections where LiveKit is the dialing side,
  # # ie. subscriber peer connections. Disabling strict ACKs will prevent clients that do not ACK
  # # peer connections from getting kicked out of rooms by the monitor. Note that if strict ACKs
  # # are disabled and clients don't ACK opened peer connections, only reliable, ordered delivery
  # # will be available.
  # strict_acks: true
  # # enable batch write to merge network write system calls to reduce cpu usage. Outgoing packets
  # # will be queued until length of queue equal to `batch_size` or time elapsed since last write exceeds `max_flush_interval`.
  # batch_io:
  #    batch_size: 128
  #    max_flush_interval: 2ms
  # # max number of bytes to buffer for data channel. 0 means unlimited.
  # # when this limit is breached, data messages will be dropped till the buffered amount drops below this limit.
  # data_channel_max_buffered_amount: 0

# 啟用後，LiveKit 將在 :6789/metrics 上公開 prometheus 指標
# prometheus_port: 6789

# API 金鑰/秘密對。
# 金鑰用於 JWT 身份驗證，伺服器 API 需要金鑰對才能產生存取權杖
# 並呼叫伺服器
keys:
  key1: secret1
  key2: secret2

# 日誌配置

# logging:
#   # log level, valid values: debug, info, warn, error
#   level: info
#   # log level for pion, default error
#   pion_level: error
#   # when set to true, emit json fields
#   json: false
#   # for production setups, enables sampling algorithm
#   # https://github.com/uber-go/zap/blob/master/FAQ.md#why-sample-application-logs
#   sample: false

# 預設房間配置
# 每個建立的房間都會繼承這些設定。如果使用 CreateRoom 明確建立房間，它們將優先於預設值

# room:
#   # allow rooms to be automatically created when participants join, defaults to true
#   # auto_create: false
#   # number of seconds to keep the room open if no one joins
#   empty_timeout: 300
#   # number of seconds to keep the room open after everyone leaves
#   departure_timeout: 20
#   # limit number of participants that can be in a room, 0 for no limit
#   max_participants: 0
#   # only accept specific codecs for clients publishing to this room
#   # this is useful to standardize codecs across clients
#   # other supported codecs are video/h264, video/vp9, video/av1, audio/red
#   enabled_codecs:
#     - mime: audio/opus
#     - mime: video/vp8
#   # allow tracks to be unmuted remotely, defaults to false
#   # tracks can always be muted from the Room Service APIs
#   enable_remote_unmute: true
#   # control playout delay in ms of video track (and associated audio track)
#   playout_delay:
#     enabled: true
#     min: 100
#     max: 2000
#   # improves A/V sync when playout_delay set to a value larger than 200ms. It will disables transceiver re-use
#   # so not recommended for rooms with frequent subscription changes
#   sync_streams: true

# Webhooks
# 配置完成後，LiveKit 會透過房間事件通知您的 URL 處理程序

# webhook:
#   # the API key to use in order to sign the message
#   # this must match one of the keys LiveKit is configured with
#   api_key: <api_key>
#   # list of URLs to be notified of room events
#   urls:
#     - https://your-host.com/handler

# 訊號中繼
# 從 v1.4.0 開始，可以使用更可靠的、基於 psrpc 的訊號中繼
# 這使我們能夠在訊號伺服器和 RTC 節點之間可靠地代理訊息

# signal_relay:
#   # amount of time a message delivery is tried before giving up
#   retry_timeout: 30s
#   # minimum amount of time to wait for RTC node to ack,
#   # retries use exponentially increasing wait on every subsequent try
#   # with an upper bound of max_retry_interval
#   min_retry_interval: 500ms
#   # maximum amount of time to wait for RTC node to ack
#   max_retry_interval: 5s
#   # number of messages to buffer before dropping
#   stream_buffer_size: 1000

# PSRPC
# 自 v1.5.1 以來，更可靠、基於 psrpc 的內部 rpc

# psrpc:
#   # maximum number of rpc attempts
#   max_attempts: 3
#   # initial time to wait for calls to complete
#   timeout: 500ms
#   # amount of time added to the timeout after each failure
#   backoff: 500ms
#   # number of messages to buffer before dropping
#   buffer_size: 1000


# 自訂音訊等級靈敏度

# audio:
#   # minimum level to be considered active, 0-127, where 0 is loudest
#   # defaults to 30
#   active_level: 30
#   # percentile to measure, a participant is considered active if it has exceeded the
#   # ActiveLevel more than MinPercentile% of the time
#   # defaults to 40
#   min_percentile: 40
#   # frequency in ms to notify changes to clients, defaults to 500
#   update_interval: 500
#   # to prevent speaker updates from too jumpy, smooth out values over N samples
#   smooth_intervals: 4
#   # enable red encoding downtrack for opus only audio up track
#   active_red_encoding: true

# turn 伺服器

# turn:
#   # Uses TLS. Requires cert and key pem files by either:
#   # - using turn.secretName if deploying with our helm chart, or
#   # - setting LIVEKIT_TURN_CERT and LIVEKIT_TURN_KEY env vars with file locations, or
#   # - using cert_file and key_file below
#   # defaults to false
#   enabled: false
#   # defaults to 3478 - recommended to 443 if not running HTTP3/QUIC server
#   # only 53/80/443 are allowed if less than 1024
#   udp_port: 3478
#   # defaults to 5349 - if not using a load balancer, this must be set to 443
#   tls_port: 5349
#   # set UDP port range for TURN relay to connect to LiveKit SFU, by default it uses a any available port
#   relay_range_start: 1024
#   relay_range_end: 30000
#   # set external_tls to true if using a L4 load balancer to terminate TLS. when enabled,
#   # LiveKit expects unencrypted traffic on tls_port, and still advertise tls_port as a TURN/TLS candidate.
#   external_tls: true
#   # needs to match tls cert domain
#   domain: turn.myhost.com
#   # optional (set only if not using external TLS termination)
#   # cert_file: /path/to/cert.pem
#   # key_file: /path/to/key.pem

# ingress 伺服器

# ingress:
#   # Prefix used to generate RTMP URLs for RTMP ingress.
#   rtmp_base_url: "rtmp://my.domain.com/live"
#   # Prefix used to generate WHIP URLs for WHIP ingress.
#   whip_base_url: "http://my.domain.com/whip"

# 目前節點的區域。如果使用區域感知節點選擇器則為必需

# region: us-west-2


# 節點選擇器

# node_selector:
#   # default: any. valid values: any, sysload, cpuload, regionaware
#   kind: sysload
#   # priority used for selection of node when multiple are available
#   # default: random. valid values: random, sysload, cpuload, rooms, clients, tracks, bytespersec
#   sort_by: sysload
#   # used in sysload and regionaware
#   # do not assign room to node if load per CPU exceeds sysload_limit
#   sysload_limit: 0.7
#   # used in regionaware
#   # list of regions and their lat/lon coordinates
#   regions:
#     - name: us-west-2
#       lat: 44.19434095976287
#       lon: -123.0674908379146

# # 節點限制
# # 設定為 -1 以停用限制

# limit:
#   # defaults to 400 tracks in & out per CPU, up to 8000
#   num_tracks: -1
#   # defaults to 1 GB/s, or just under 10 Gbps
#   bytes_per_sec: 1_000_000_000
#   # how many tracks (audio / video) that a single participant can subscribe at same time.
#   # if the limit is exceeded, subscriptions will be pending until any subscribed track has been unsubscribed.
#   # value less or equal than 0 means no limit.
#   subscription_limit_video: 0
#   subscription_limit_audio: 0
#   # limit size of room and participant's metadata, 0 for no limit
#   max_metadata_size: 0
#   # limit size of participant attributes, 0 for no limit
#   max_attributes_size: 0
#   # limit length of room names
#   max_room_name_length: 0
#   # limit length of participant identity
#   max_participant_identity_length: 0
```