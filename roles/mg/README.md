# mg：MailGates

MailGates 的產品 role。依規範第 4.1 節，工作拆成 task 檔：install.yml、patch.yml、cert.yml、check.yml、shutdown.yml。
playbook 用 `include_role` + `tasks_from` 指定要跑哪個檔。

## 目前狀態

- `install.yml`、`patch.yml`：尚未實作，執行會直接報錯停止。
- `cert.yml`：可用，呼叫 `web_cert`。照 MG 管理介面（`mg_adm`）的做法：憑證、私鑰、中繼憑證分別放 `conf/ssl.crt/server.crt.<後綴>`、`conf/ssl.key/server.key.<後綴>`、`conf/server-ca.crt.<後綴>`（後綴為生效時間＋指紋，同一張憑證重跑不會再產生新檔），權限 0600、擁有者 webmail；直接改掉 `extra/mg_ssl.conf` 原本的三行，舊檔留著。
  - **換完會開啟 HTTPS**：取消 `httpd.conf` 裡 `Include conf/extra/mg_ssl.conf` 的註解（新安裝預設關閉），同管理介面選「HTTPS」。管理介面的「HTTPS 轉址」「HTTPS only」不動。
  - 重啟前先跑 `apachectl -t`，語法錯就停；以 webmail 執行 `mg_ctl.pl -g restart httpd`（graceful），不可用 root 執行 apachectl，否則 httpd 會改以 root 執行。
  - 換完確認 443 送出的是新憑證，不是就重啟、重啟後仍不是就停。
- `check.yml`：可用，只讀不改。顯示 HTTPS 是否開啟，以及 `mg_ssl.conf` 實際生效的憑證的主旨與到期日。
- `shutdown.yml`：可用。以 webmail 執行 `mg_ctl.pl stop all`（不含 httpd，由關機時的系統停掉） 停服務，再呼叫 `os_baseline` 的 `shutdown` 關機；由 `playbooks/shutdown.yml` 執行。
- `weakscan.yml`：產品層弱掃項目尚未定義，所以還沒建這個檔。

## 標準值

見 `defaults/main.yml`。值為 `"TODO"` 的項目還沒確認，用到時 role 會停下來報錯，不會帶著錯誤的值執行。
