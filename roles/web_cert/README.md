# web_cert：Apache 換憑證（通用 role）

上傳憑證與私鑰、修改 Apache 的 SSL 設定，有變動才重啟。所有產品共用，**不寫死任何產品的路徑**，
由產品 role 的 `tasks/cert.yml` 帶入參數，例如 `roles/mg/tasks/cert.yml`。

| 變數 | 說明 |
| --- | --- |
| `web_cert_crt_src` / `web_cert_key_src` | control node 上的憑證、私鑰（私鑰以 ansible-vault 加密） |
| `web_cert_conf_dir` | 主機上放憑證的目錄，也是 `httpd.conf` 所在目錄 |
| `web_cert_ssl_conf` | 含 `SSLCertificateFile` 的設定檔 |
| `web_cert_include` | 要在 `httpd.conf` 啟用的 Include；空字串表示不需要 |
| `web_cert_owner` / `web_cert_group` | 檔案擁有者 |
| `web_cert_restart_cmd` | 重啟指令（由 handler 執行，只在有變動時跑） |

- 任何參數是空值或 `TODO` 時，第一個 task 就會停下來，不會帶著錯誤的值修改主機。
- 私鑰權限 0600，且上傳私鑰的 task 關閉 diff，`--diff` 不會印出私鑰內容。
- 修改設定檔前都會備份（`backup: true`）。
