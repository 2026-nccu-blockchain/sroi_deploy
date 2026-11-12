# SROI

SROI 系統的部署設定，用 Docker Compose 一次啟動 nginx、前端、後端與資料庫。

## 專案組成

| 資料夾 | 說明 |
|---|---|
| `sroi_frontend/` | Vue 3 + Vite 前端，build 後由容器內的 nginx 提供靜態檔（設定在前端 repo 的 `nginx.conf`） |
| `sroi_backend/` | FastAPI 後端 |
| `nginx/` | 對外 nginx 的設定 `nginx.conf.template`（反向代理） |
| `db/` | PostgreSQL 初始化腳本 |

## 架構

```
瀏覽器 → nginx（:80，唯一對外）
          ├─ /       → frontend（nginx :80，前端靜態檔）
          └─ /api/*  → backend（FastAPI :8000）→ db（PostgreSQL :5432）

pgadmin（127.0.0.1:5050，只限伺服器本機）→ db
```

### 網路

| 網路 | 說明 | 使用的服務 |
|---|---|---|
| `public` | 一般 bridge，可以連外、可以發布 port | nginx、backend（呼叫 Brevo 寄信）、pgadmin |
| `internal` | `internal: true`，不能連外，只供容器互連 | nginx、frontend、backend、db、pgadmin |

只有 `nginx` 對外開放 port 80；`frontend`、`backend`、`db` 都沒有發布 port，外部無法直接連線。

## 事前準備

- Docker 與 Docker Compose

## 設定

複製 `.env.example` 為 `.env` 並填入各項設定。

- `SERVER_URL`：網站網域，留空時接受任何網域。
- `BREVO_*`：自動寄信用的 Brevo 設定，`BREVO_API_KEY` 留空時寄信會失敗並回傳錯誤。

## 啟動

```bash
docker compose up -d --build
```

啟動後開啟 http://localhost

## 連線 pgadmin

pgadmin 只綁在伺服器的 `127.0.0.1:5050`，從自己電腦連線時先開 SSH tunnel：

```bash
ssh -L 5050:127.0.0.1:5050 <使用者>@<伺服器>
```

再開啟 http://localhost:5050 ，新增伺服器時主機填 `db`、port 填 `5432`。

## 常用指令

```bash
docker compose ps                       # 查看服務狀態
docker compose logs -f backend          # 查看後端 log
docker compose logs -f nginx            # 查看 nginx log
docker compose up -d --build frontend   # 只重新建置前端
docker compose up -d --build backend    # 只重新建置後端
docker compose up -d --force-recreate nginx   # 修改 SERVER_URL 或 nginx 設定後重新產生設定
docker compose exec nginx cat /etc/nginx/conf.d/default.conf   # 查看實際的 nginx 設定
docker compose down                     # 停止服務（保留資料庫與上傳檔案）
```

> `docker compose down -v` 會一併刪除 volume（資料庫資料、上傳的檔案、pgadmin 設定）。

## 疑難排解

| 狀況 | 可能原因 |
|---|---|
| `backend` 一直重啟 | 資料庫連線失敗，請確認 `DB_USER`、`DB_PASSWORD`、`DB_NAME` |
| 網頁回傳 502 | 後端或前端沒有正常啟動，請執行 `docker compose logs backend frontend` 查看 |
| 重新整理子頁面回傳 404 | 前端映像檔沒有包含 `nginx.conf`，請確認 `sroi_frontend` 已更新並重新建置 |
| 寄信失敗 | `BREVO_API_KEY` 或 `BREVO_WELCOME_TEMPLATE_ID` 不正確，或 backend 沒有接上 `public` 網路 |
| 上傳檔案回傳 413 | 超過 nginx 的 `client_max_body_size` |
