# weakscan：作業系統層弱掃修補（通用 role）

依弱掃報告調整 sshd 的 `Ciphers`、`MACs`、`KexAlgorithms`。清單放在客戶 inventory（例如 `group_vars/all/main.yml`），
清單為空就不改。

- 改 `sshd_config` 前會先用 `sshd -t` 驗證（`validate`），寫錯不會被套用，避免 SSH 失聯。
- 修改前備份，有變動才重啟 sshd。
- 產品自身的弱掃項目放各產品 role 的 `tasks/weakscan.yml`，不放這裡。
