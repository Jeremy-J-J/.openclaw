# TOOLS.md - Local Notes

Skills define _how_ tools work. This file is for _your_ specifics — the stuff that's unique to your setup.

## What Goes Here

Things like:

- Camera names and locations
- SSH hosts and aliases
- Preferred voices for TTS
- Speaker/room names
- Device nicknames
- Anything environment-specific

## Examples

```markdown
### Cameras

- living-room → Main area, 180° wide angle
- front-door → Entrance, motion-triggered

### SSH

- home-server → 192.168.1.100, user: admin

### TTS

- Preferred voice: "Nova" (warm, slightly British)
- Default speaker: Kitchen HomePod
```

## Why Separate?

Skills are shared. Your setup is yours. Keeping them apart means you can update skills without losing your notes, and share skills without leaking your infrastructure.

---

Add whatever helps you do your job. This is your cheat sheet.

## SSH 服务器

- **6000pro** = 221.12.22.150，端口 1208，用户名 jeremyj

- **a800** = 221.12.22.187，端口 30669，用户名 root
  - 备注：/workspace 基本为空，磁盘 445GB 已用 55%

- **nuc** = 192.168.10.243，用户名 jeremyj
  - 备注：免密登录已配置

- **227** = 内网 10.160.199.227，用户名 jeremyj
  - 连接方式：跳板机跳转到 221.12.22.162:12022，跳板机密钥 ~/.ssh/id_ed25519_227
  - 命令：ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -t -p 12022 root@221.12.22.162 "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i ~/.ssh/id_ed25519_227 jeremyj@10.160.199.227"

- **tx** = 124.221.85.91，用户名 ubuntu
  - 密钥：/Users/jeremyj_pc/.ssh/TencentCloud_Key_0817.pem
  - 密码：jiangcc（需交互输入）

- **foretify** = 10.14.0.98，用户名 user05
  - 密钥：~/.ssh/id_rsa
  - 密码：zuser05（需交互输入）
  - workspace 路径：/home/user05/workspace（根目录 /workspace 不存在）
