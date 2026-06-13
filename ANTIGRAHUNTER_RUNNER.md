---
name: AntigraHunter-runner
description: AntigraHunter runner — Auto-load trigger. Khi user gõ 'bugfinding ...', AI đọc BUGFINDING.md và thực thi pipeline đầy đủ.
---

# AntigraHunter Auto-Runner

Khi user nhập lệnh bắt đầu bằng `bugfinding`, thực hiện ngay:

1. Đọc file: `e:\Antigravity\AntigraHunter\BUGFINDING.md`
2. Đọc file: `e:\Antigravity\AntigraHunter\SKILL.md`  
3. Đọc file: `e:\Antigravity\AntigraHunter\AGENTS.md`
4. Thực thi pipeline theo đúng BUGFINDING.md

**User KHÔNG cần copy bất kỳ file nào vào folder target.**  
**Flag `-f` chỉ đường dẫn đến code cần scan, không liên quan đến AntigraHunter files.**
