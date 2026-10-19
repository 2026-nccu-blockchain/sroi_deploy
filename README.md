# SROI

SROI 系統的部署設定，用 Docker Compose 一次啟動前端、後端與資料庫通道。

## 專案組成

| 資料夾 | 說明 |
|---|---|
| `sroi_frontend/` | Vue 3 + Vite 前端，由 nginx 提供靜態檔並反向代理 API |
| `sroi_backend/` | FastAPI 後端 |
| `sroi_deploy` | 部署相關 |

## 架構

```
瀏覽器 → frontend（nginx :80）
          ├─ /       → 前端靜態檔
          └─ /api/*  → backend（FastAPI :8000）→ db-tunnel（cloudflared）→ 遠端 PostgreSQL
```

只有 `frontend` 對外開放 port 8080，其他服務只能在 Docker 內部網路互相連線。

## 事前準備

- Docker 與 Docker Compose

## 設定

複製 `.env.example` 為 `.env` 並填入各項設定。
   `SERVER_URL` 為網站網域，留空時接受任何網域。


## 啟動

```bash
docker compose up -d --build
```

啟動後開啟 http://localhost:8080

## 常用指令

```bash
docker compose ps                   # 查看服務狀態
docker compose logs -f backend      # 查看後端 log
docker compose logs -f db-tunnel    # 查看資料庫通道 log
docker compose up -d --build frontend   # 只重新建置前端
docker compose up -d frontend           # 修改 SERVER_URL 後重新產生 nginx 設定
docker compose exec frontend cat /etc/nginx/conf.d/default.conf   # 查看實際的 nginx 設定
docker compose down                 # 停止服務（保留上傳檔案）
```

> `docker compose down -v` 會一併刪除 volume（上傳的檔案）。

## 疑難排解

| 狀況 | 可能原因 |
|---|---|
| `db-tunnel` 連不上 | `DB_TUNNEL_HOSTNAME` 不正確，或 Cloudflare 通道沒有啟動 |
| `backend` 一直重啟 | 資料庫連線失敗，請先確認通道與帳號密碼 |
| 網頁回傳 502 | 後端沒有正常啟動，請執行 `docker compose logs backend` 查看 |
| 上傳檔案回傳 413 | 超過 nginx 的 `client_max_body_size` |