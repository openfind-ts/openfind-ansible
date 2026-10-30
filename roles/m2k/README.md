# m2k：Mail2000

Mail2000 的產品 role。依規範第 4.1 節，工作拆成 task 檔：install.yml、patch.yml、cert.yml、check.yml。
playbook 用 `include_role` + `tasks_from` 指定要跑哪個檔。

## 目前狀態

- `install.yml`：尚未實作，執行會直接報錯停止。
- `patch.yml`：可用，目前只確認客製化客戶適用。依序安裝 `m2k_patch_dir` 裡比主機目前版本新的 patch，裝到最新；已是最新就跳過。目前版本看各台的安裝紀錄 `m2k_patch_record_file`（m2kpatch.info），多主機共用 `/webmail/etc` 時要把它 link 到各台本機。任一個 patch 失敗就停，服務維持停止，由人確認後再處理。installer 以非同步執行，上限 `m2k_patch_timeout` 秒。
- `cert.yml`：可用，呼叫 `web_cert`。
- `check.yml`：可用，只讀不改。版本從 `m2k_version_file`（`system_version.info` 第一欄）讀取，目前只確認客製化客戶適用。
- `weakscan.yml`：產品層弱掃項目尚未定義，所以還沒建這個檔。

## 標準值

見 `defaults/main.yml`。值為 `"TODO"` 的項目還沒確認，用到時 role 會停下來報錯，不會帶著錯誤的值執行。
