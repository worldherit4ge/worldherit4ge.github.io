# Immich + Tailscale 部署說明

本文說明目前這台機器上 Immich（照片伺服器）與 Tailscale（零設定 VPN）的部署方式、檔案結構與操作流程，方便日後維護與移機。

---

## 1. 檔案結構

專案根目錄（例如 `C:\Users\Kris\immich-app`）：

```text
immich-app/
├─ docker-compose.yml        # 主要服務定義
├─ .env                      # 環境變數（file.env）
└─ ts-immich/
   ├─ state/                 # Tailscale 狀態檔（節點登入資訊）
   └─ config/
      └─ immich.json         # Tailscale Serve 反向代理設定
```

---

## 2. .env 內容說明

`.env`（原 file.env）核心參數如下：

```env
# 媒體檔案儲存位置（掛到 /data）
UPLOAD_LOCATION=/d/Home/Pictures/immich-app/library

# Postgres 資料庫檔案位置（掛到 /var/lib/postgresql/data）
DB_DATA_LOCATION=/d/Home/Pictures/immich-app/postgres

# 使用的 Immich 版本（v2 = 最新 v2 穩定版）
IMMICH_VERSION=v2

# Postgres 帳號與密碼
DB_PASSWORD=postgres
DB_USERNAME=postgres
DB_DATABASE_NAME=immich

# Tailscale Auth Key（提供 tailscale-proxy 自動登入 tailnet）
TS_AUTHKEY=tskey-auth-xxxxxxxxxxxxxxxx
```

說明：

- Windows 路徑使用 `/d/...` 形式，以配合 Docker Desktop 掛載磁碟。  
- DB 密碼目前使用預設值，因 Postgres 只在容器網路內暴露，可視需求再改成更隨機的字串。

---

## 3. docker-compose.yml 服務說明

`docker-compose.yml` 以官方 Immich compose 為基礎，外加一個 Tailscale 反向代理服務。

### 3.1 immich-server

```yaml
immich-server:
  container_name: immich_server
  image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
  volumes:
    - ${UPLOAD_LOCATION}:/data
    - /etc/localtime:/etc/localtime:ro
  env_file:
    - .env
  ports:
    - "2283:2283"
  depends_on:
    - redis
    - database
  restart: always
  healthcheck:
    disable: false
```

- 提供 Immich Web UI 與 API。  
- `/data` 掛載到 `UPLOAD_LOCATION`，所有照片與影片存放於該路徑。  
- `ports: 2283:2283` 讓本機可用 `http://localhost:2283` 直接存取。

### 3.2 immich-machine-learning

```yaml
immich-machine-learning:
  container_name: immich_machine_learning
  image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
  volumes:
    - model-cache:/cache
  env_file:
    - .env
  restart: always
  healthcheck:
    disable: false
```

- 執行臉部辨識、物件偵測等 ML 工作。  
- 目前採 CPU 模式，模型快取存於 `model-cache` volume。

### 3.3 redis

```yaml
redis:
  container_name: immich_redis
  image: docker.io/valkey/valkey:9@sha256:fb8d272e529ea567b9bf1302245796f21a2672b8368ca3fcb938ac334e613c8f
  healthcheck:
    test: redis-cli ping || exit 1
  restart: always
```

- Immich 使用的快取 / 佇列服務。

### 3.4 database（Postgres）

```yaml
database:
  container_name: immich_postgres
  image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23
  environment:
    POSTGRES_PASSWORD: ${DB_PASSWORD}
    POSTGRES_USER: ${DB_USERNAME}
    POSTGRES_DB: ${DB_DATABASE_NAME}
    POSTGRES_INITDB_ARGS: '--data-checksums'
    # DB_STORAGE_TYPE: 'HDD' 可在非 SSD 時啟用
  volumes:
    - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
  shm_size: 128mb
  restart: always
```

- Immich 使用的資料庫，資料檔存放於 `DB_DATA_LOCATION`。  
- `--data-checksums` 可在未來偵測資料損毀。

### 3.5 tailscale-proxy（Tailscale 反向代理）

```yaml
tailscale-proxy:
  image: tailscale/tailscale:latest
  container_name: ts-immich-proxy
  hostname: immich
  environment:
    - TS_AUTHKEY=${TS_AUTHKEY}
    - TS_STATE_DIR=/var/lib/tailscale
    - TS_SERVE_CONFIG=/config/immich.json
  volumes:
    - ./ts-immich/state:/var/lib/tailscale
    - ./ts-immich/config:/config
  devices:
    - /dev/net/tun:/dev/net/tun
  cap_add:
    - net_admin
    - sys_module
  network_mode: host
  restart: unless-stopped
  depends_on:
    - immich-server
```

- 讓此機器成為 Tailnet 中的 `immich` 節點。  
- 使用 `TS_AUTHKEY` 自動登入 Tailscale，狀態寫入 `ts-immich/state`。  
- 透過 `TS_SERVE_CONFIG` 載入 `/config/immich.json` 的 Serve 設定。  
- `network_mode: host` 允許 Tailscale 直接連到宿主機上的 `127.0.0.1:2283`。

---

## 4. Tailscale Serve 設定（immich.json）

`ts-immich/config/immich.json`：

```json
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:2283"
        }
      },
      "AllowFunnel": {
        "${TS_CERT_DOMAIN}:443": false
      }
    }
  }
}
```

說明：

- 在節點上開啟 TCP 443，並啟用 HTTPS。  
- `${TS_CERT_DOMAIN}` 會自動替換成類似 `immich.tailb95c5f.ts.net` 的實際網域。  
- 所有到 `https://immich.<tailnet>.ts.net/` 的請求會被 proxy 到 `http://127.0.0.1:2283`（Immich server）。  
- `AllowFunnel=false` → 服務只在 Tailnet 內可見，不會對公網開放。

實際 `tailscale serve status` 顯示（示意）：

```text
https://immich.tailb95c5f.ts.net (tailnet only)
|-- / proxy http://127.0.0.1:2283
```

---

## 5. 啟動、更新與停止

在 `immich-app` 目錄：

### 啟動 / 重啟所有服務

```bash
docker compose up -d
```

### 更新到最新 v2 版本（Immich / ML / Postgres / Tailscale）

```bash
docker compose pull
docker compose up -d
```

### 停止所有服務

```bash
docker compose down
```

### 檢查服務狀態與 log

```bash
docker compose ps
docker compose logs immich-server --tail=100
docker compose logs tailscale-proxy --tail=100
```

---

## 6. 存取方式整理

### 本機（同一台 Windows）

- 瀏覽器：  
  ```text
  http://localhost:2283
  ```

### Tailnet 其他裝置（目前實際使用）

- Immich app / 瀏覽器：  
  ```text
  http://100.92.195.104:2283
  ```
  其中 `100.92.195.104` 為 `immich` 節點在 Tailnet 中的 IPv4 位址。

### Tailnet 其他裝置（預計的 HTTPS 網域）

- 透過 Tailscale Serve：  
  ```text
  https://immich.tailb95c5f.ts.net
  ```
  如遇 iOS 對 HTTPS→HTTP 的警告，可暫時改用上面的 HTTP+IP 方式。

---

## 7. 備份與遷移

要備份或移轉此環境，至少需保留：

- 媒體資料：`UPLOAD_LOCATION` 指向的資料夾  
- 資料庫資料：`DB_DATA_LOCATION` 指向的資料夾  
- Tailscale 狀態與設定：`ts-immich/state/`、`ts-immich/config/immich.json`  
- 設定檔：`docker-compose.yml`、`.env`

在新主機：

1. 安裝 Docker / Docker Compose。  
2. 將上述檔案與資料夾放到新位置（調整 `.env` 中的路徑）。  
3. 於新目錄執行：

   ```bash
   docker compose up -d
   ```

即可還原完整的 Immich + Tailscale 環境。
