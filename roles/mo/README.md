# mo：MailAudit

MailAudit 的產品 role。依規範第 4.1 節，工作拆成 task 檔：install.yml、patch.yml、cert.yml、check.yml。
playbook 用 `include_role` + `tasks_from` 指定要跑哪個檔。

## 目前狀態

- `install.yml`、`patch.yml`：尚未實作，執行會直接報錯停止。
- `cert.yml`：可用，呼叫 `web_cert`。
- `check.yml`：可用，只讀不改。
- `weakscan.yml`：產品層弱掃項目尚未定義，所以還沒建這個檔。

## 標準值

見 `defaults/main.yml`。值為 `"TODO"` 的項目還沒確認，用到時 role 會停下來報錯，不會帶著錯誤的值執行。
