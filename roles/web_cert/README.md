# web_cert：Apache 換憑證（通用 role）

上傳憑證與私鑰、修改 Apache 的 SSL 設定，有變動才重啟。所有產品共用，**不寫死任何產品的路徑**，
由產品 role 的 `tasks/cert.yml` 帶入參數，例如 `roles/mg/tasks/cert.yml`。

| 變數 | 說明 |
| --- | --- |
| `web_cert_crt_src` / `web_cert_key_src` | control node 上的憑證、私鑰（私鑰以 ansible-vault 加密） |
| `web_cert_chain_src` | control node 上的中繼憑證；空字串表示沒有 |
| `web_cert_conf_dir` | `httpd.conf` 所在目錄 |
| `web_cert_cert_dir` | 主機上放憑證的目錄，預設同 `web_cert_conf_dir` |
| `web_cert_key_dir` / `web_cert_chain_dir` | 私鑰、中繼憑證放的目錄，預設同 `web_cert_cert_dir`（MG 分放 `ssl.key/`、`conf/`） |
| `web_cert_dir_mode` | 建立憑證目錄的權限，預設 0700 |
| `web_cert_ssl_conf` | 含 `SSLCertificateFile` 的設定檔 |
| `web_cert_sites` | 要換同一張憑證的站（`ssl_conf`、`cert_dir`，選填 `key_dir`、`chain_dir`、`server_name`），預設只有上面這一站 |
| `web_cert_include` | 要在 `httpd.conf` 啟用的 Include；空字串表示不需要 |
| `web_cert_owner` / `web_cert_group` / `web_cert_crt_mode` | 檔案擁有者、憑證檔權限 |
| `web_cert_versioned` | `false`：沿用來源檔名覆蓋。`true`：檔名 `<prefix>.crt.<生效時間>.<指紋>`，舊檔保留 |
| `web_cert_keep_old_lines` | `true`：舊設定行註解保留、新行放最上面（M2K）。`false`：直接改掉原本那一行（MG）。預設同 `web_cert_versioned` |
| `web_cert_restart_cmd` | 重啟指令（由 handler 執行） |
| `web_cert_configtest_cmd` | 重啟前檢查設定語法的指令；不通過就停、不重啟 |
| `web_cert_verify_port` | 大於 0 時確認主機這個 port 送出的是新憑證，不是就重啟、重啟後仍不是就停 |

- 任何參數是空值或 `TODO` 時，第一個 task 就會停下來，不會帶著錯誤的值修改主機。
- 私鑰權限 0600，且上傳私鑰的 task 關閉 diff，`--diff` 不會印出私鑰內容。
- 修改設定檔前都會備份（`backup: true`）。
- 新的設定行都會加在同一組註解範例旁邊，留在原本的 VirtualHost 裡，不會跑到檔尾。
- 多台主機共用同一份設定（例如 NAS）時，第二台檔案已是新的、不會有變動；設 `web_cert_verify_port` 才會依「實際送出的憑證」決定重啟。
- 新憑證的檔名後綴在 control node 用 `openssl` 讀取，control node 需要有 openssl。
