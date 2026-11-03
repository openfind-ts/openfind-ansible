# haproxy：以 docker 執行的 HAProxy（通用 role）

部分客戶在 Openfind 產品前面放一台以 docker 執行的 HAProxy（SSL 終結、轉送 443／25／993／995 等）。
HAProxy 不是 Openfind 產品，前面可以接任何產品，所以做成通用 role，**不寫死任何客戶路徑**，
差異放客戶 inventory 的 `group_vars/haproxy.yml`。主機放在 inventory 的 `haproxy` 群組。

## 前提

- 主機以 `docker compose` 執行官方 `haproxy` image，`haproxy.cfg` 與憑證目錄以 bind mount 掛進容器，例如：

  ```yaml
  volumes:
    - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    - ./certs:/etc/haproxy/certs:ro
  ```

- `haproxy.cfg` 用單一 pem：`bind *:443 ssl crt /etc/haproxy/certs/server.pem`。
- 主機上有 `docker` 與 `openssl`。

## 工作

| task 檔 | 用途 | playbook |
| --- | --- | --- |
| `cert.yml` | 換憑證 | `playbooks/renew_cert.yml` |
| `check.yml` | 容器狀態、版本、`haproxy -c`、送出憑證的到期日，只讀不改 | `playbooks/healthcheck.yml` |

`cert.yml` 的步驟：

1. 在 control node 確認新憑證未過期、私鑰與憑證成對。
2. 依 HAProxy 的格式組出 pem：伺服器憑證 → 中繼憑證 → 私鑰。
3. 主機上的 pem 內容不同才換：舊檔先備份到 `haproxy_backup_dir`（不放在掛進容器的目錄），再放上新檔。
4. 在容器內跑 `haproxy -c`；不通過就還原舊檔並停止，**不 reload**。
5. 以 handler 對容器送 `HUP`，HAProxy graceful reload（既有連線不中斷，不重建容器）。
6. 從主機的 `127.0.0.1:443` 確認送出的是新憑證；檔案已是新的但送出的仍是舊憑證，也會 reload；reload 後仍不是就停。

## 變數

| 變數 | 說明 |
| --- | --- |
| `haproxy_compose_dir` | 主機上 `docker-compose.yml` 所在目錄（必填） |
| `haproxy_container` | `container_name`，預設 `haproxy` |
| `haproxy_cert_dir` / `haproxy_pem_name` | 掛進容器的憑證目錄與 pem 檔名，預設 `<compose_dir>/certs/server.pem` |
| `haproxy_backup_dir` | 舊 pem 備份目錄，預設 `<compose_dir>/cert-backup` |
| `haproxy_cert_src` / `haproxy_key_src` / `haproxy_chain_src` | control node 上的憑證、私鑰（ansible-vault 加密）、中繼憑證 |
| `haproxy_pem_owner` / `haproxy_pem_group` / `haproxy_pem_mode` | 預設 root:root 0644（官方 image 2.4 起容器內以非 root 執行，要讀得到） |
| `haproxy_reload_signal` | 預設 `HUP` |
| `haproxy_verify_port` / `haproxy_verify_server_name` | 確認送出憑證用的 port 與 SNI；port 設 0 不確認 |

- pem 含私鑰：上傳與還原都關閉 diff、`no_log`，`--diff` 不會印出私鑰。
- 同一張憑證也用在後端產品（例如 M2K）時，`haproxy_cert_src` 等直接指向同一組檔案，換一次兩邊都換。

## 尚未納管（之後再補）

- `docker-compose.yml`、`haproxy.cfg` 本身（目前由現場手動維護，role 只換憑證）。要納管時再加 template，並用 `haproxy -c` 驗證。
- image 版本：客戶目前多用 `haproxy:latest`，建議改成固定版本再納管升級。
