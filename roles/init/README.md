init
====

新机器基础配置角色，对 `web` 组主机执行一次性的初始化设置。

职责
----

1. **设置主机名** —— 使用 `hostname` 模块，将主机名设为 `inventory_hostname`。
2. **新增 ops 用户** —— 创建运维用户，带家目录和 `/bin/bash`。
3. **ops 免密 sudo** —— 通过 `visudo -cf` 校验后再下发 `/etc/sudoers.d/ops-nopasswd`。
4. **配置 ops 的 SSH 公钥** —— 将控制机 `~/.ssh/id_rsa.pub` 写入 ops 的 `authorized_keys`。
5. **配置 Debian apt 源** —— 用模板渲染 `/etc/apt/sources.list`，并备份原文件；变更后通知更新 apt 缓存。
6. **SSH 加固** —— 禁止 root 密码登录（`PermitRootLogin prohibit-password`）、禁止密码认证（`PasswordAuthentication no`），变更后通知重启 ssh。

变量
----

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `apt_mirror` | `http://deb.debian.org/debian` | Debian 主源镜像 |
| `apt_mirror_security` | `http://security.debian.org/debian-security` | Debian 安全源镜像 |

依赖
----

无。

示例 Playbook
--------------

    - hosts: web
      become: true
      roles:
        - init

License
-------

MIT
