# MEMORY.md - Long-Term Memory

## 用户偏好

- **GitHub 项目分析**：默认使用 `github-explorer` 技能（除非用户明确指定其他技能）

## 关于用户

- 使用 OpenClaw TUI (openclaw-tui) 与我交流
- 时区：Asia/Shanghai

## SSH 登录服务器最佳实践

**目标服务器：**
- nuc：192.168.10.243，账号 jeremyj
- A800：221.12.22.187，端口 30669，账号 root

**流程：**
1. 直接使用 `ssh jeremyj@192.168.10.243 'command'` 执行单条命令
2. 非默认端口（如 A800 的 30669）需加 `-p` 参数：`ssh -p 30669 root@221.12.22.187 'command'`
2. 连接后执行命令，自动断开，不维持长连接
3. 每次操作都是一次性的：连接→执行→返回→断开

**注意：** 使用密钥登录，避免每次输入密码。**不断开连接**（除非用户明确要求保持会话）。

## 项目相关

- claude-code-best-practice 仓库分析记录：见 memory/2026-04-01.md
