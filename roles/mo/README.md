# mo：MailAudit

MailAudit 的產品 role。依規範第 4.1 節，工作拆成 task 檔：install.yml、patch.yml、cert.yml、check.yml。
playbook 用 `include_role` + `tasks_from` 指定要跑哪個檔。

## 目前狀態

- `install.yml`：尚未實作，執行會直接報錯停止。
- `patch.yml`：可用，依 `mo_patch_all.pl` 的流程，把 `mo_patch_dir` 底下的 patch 依序號裝到最新，標準版與客製版都適用。
  - 版本看 `/mailgates/mg/etc/version` 第一行：標準版例如 `mo-6.1.10.051-x86_64-linux`，客製版多一段客戶代號，例如 `mo-CustA_6.2.11.005-x86_64-linux`。主機的客戶代號（標準版沒有）和主版本要跟 patch 一致，版本也要有序號，否則停下不裝。
  - patch.pl 會自己停、啟它需要的服務（例如 syncd）。如果 patch.pl 會停止或重啟所有服務、crond，就停下由人確認。舊腳本的 ChgProcess 改的是 `update/process.pl`，但 patch.pl 執行的是內嵌的版本，那樣改沒有作用。
  - `--check` 只顯示安裝計畫。
- `cert.yml`：可用，呼叫 `web_cert`。
- `check.yml`：可用，只讀不改。
- `weakscan.yml`：產品層弱掃項目尚未定義，所以還沒建這個檔。

## 標準值

見 `defaults/main.yml`。值為 `"TODO"` 的項目還沒確認，用到時 role 會停下來報錯，不會帶著錯誤的值執行。
