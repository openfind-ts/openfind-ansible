# CVE 定義檔

每個 CVE 一個檔案，`playbooks/cve_check.yml` 讀這個檔案，掃描所有產品主機，**只讀不改**。

## 新增一個 CVE

1. 複製 `CVE-2026-109618.yml`，檔名改成新的 CVE 編號。
2. 改 `component`、`probe`（怎麼找到元件、怎麼讀版本）、`branches`（各分支第一個已修補的版本，EOL 分支寫 `null`）。
3. 先在 lab 跑一次：

```bash
ansible-playbook -i ../customer-inventories/lab playbooks/cve_check.yml -e cve=CVE-xxxx-xxxxx
```

不用改任何 role 或 playbook。元件不是 PHP 時（例如 OpenSSL、Apache），只要 `probe` 寫得出「找哪些檔案、怎麼讀版本」就能沿用。

## 判斷規則

| 結果 | 意思 |
| --- | --- |
| `affected` | 版本落在受影響範圍，或分支已 EOL |
| `fixed` | 版本已大於等於該分支的修補版本 |
| `unknown` | 分支不在定義檔裡，或讀不到版本，要人工確認 |

報告會寫到 control node 的 `~/ansible-reports/`，**不放進任何 repo**（裡面有客戶主機名稱）。
