# os_baseline：作業系統基本設定（通用 role）

| 功能 | 狀態 | 開關 |
| --- | --- | --- |
| NTP（chrony） | 可用 | `os_baseline_ntp_servers` 有值才執行 |
| 系統更新（dnf） | 可用 | `os_baseline_update: true` 才執行；預設只裝安全性更新 |
| sshd 設定、管理帳號 | TODO | |

- `chrony.conf` 由 template 整份產生，修改前會備份。
- 系統更新後預設**不會**重開機；kernel 更新要重開時，執行時加 `-e os_baseline_reboot=true`，並搭配 playbook 的 `serial: 1`。
