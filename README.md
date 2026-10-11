# SROI

SROI 系統的部署設定，用 Docker Compose 一次啟動前端、後端與資料庫通道。

## 專案組成

| 資料夾 | 說明 |
|---|---|
| `sroi_frontend/` | Vue 3 + Vite 前端，由 nginx 提供靜態檔並反向代理 API |
| `sroi_backend/` | FastAPI 後端 |
| `sroi_deploy` | 部屬相關 |

## 架構

```
瀏覽器 → web（nginx :80）
          ├─ /       → 前端靜態檔
          └─ /api/*  → api（FastAPI :8000）→ db-tunnel（cloudflared）→ 遠端 PostgreSQL
```

只有 `web` 對外開放 port 80，其他服務只能在 Docker 內部網路互相連線。

## 事前準備

- Docker 與 Docker Compose
- 在 Cloudflare Zero Trust 建立 **Service Token**，並在資料庫的 Access Policy 加入一條 `Service Auth` 規則，允許這組 token

## 設定

1. 在此資料夾建立 `.env`：

   ```
   DB_TUNNEL_HOSTNAME=
   DB_USER=
   DB_PASSWORD=
   DB_NAME=
   ```

2. 設定後端的 `sroi_backend/.env`（請參考 `sroi_backend/.env.example`）。
   其中的 `DATABASE_URL` 會被 compose 覆蓋，不用修改。

## 啟動

```bash
docker compose up -d --build
```

啟動後開啟 http://localhost

## 常用指令

```bash
docker compose ps                  # 查看服務狀態
docker compose logs -f api         # 查看後
docker compose logs -f db-tunnel   # 查看資料庫通道 log
docker compose up -d --build web   # 只重新建置前端
docker compose down                # 停止服務（保留上傳檔案）
```

>`docker compose down -v` 會一併刪除 vol。

## 疑難排解

| 狀況 | 可能原因 |
|---|---|
| `db-tunnel` 出現 403 或驗證錯誤 | Service 尚未允許這組 token |
| `api` 一直重啟 | 資料庫連線失敗，請先確認通道與帳號密碼 |
| 網頁回傳 502 | 後端沒有正常啟動，請查看 `l
| 上傳檔案回傳 413 | 超過 nginx 的 `client_max_body_size` |