# Ansible Project

基于 Ansible Roles 的 Debian 服务器批量初始化工具，实现主机基础配置（主机名、用户、SSH 加固、apt 源）和 Nginx 的一键部署。

## 拓扑

| 角色 | 主机名 | IP | 系统 / 说明 |
|------|--------|----|-------------|
| 控制机 | vm1 | 192.168.118.100 | Debian，装 ansible-core |
| 受控机 | debian1 | 192.168.118.101 | Debian，装 nginx |
| 受控机 | debian2 | 192.168.118.102 | Debian，装 nginx |
| 受控机 | debian3 | 192.168.118.103 | Debian，装 nginx |

- 网络：单一网段 `192.168.118.0/24`（Host-Only 内网），**未配置双网（无 NAT 外网卡）**，所有验证均在该网段内完成。
- 控制机与被控机之间通过 SSH 密钥认证，控制机 100 统一批量下发配置。

## 目录结构

```
.
├── inventory.ini          # 主机清单（web 组：debian1~3）
├── Init.yaml              # 基础初始化 playbook（roles/init）
├── site.yaml              # Nginx 部署 playbook（roles/nginx）
├── secrets.yml            # Ansible Vault 加密的敏感变量
└── roles/
    ├── init/              # 基础初始化角色
    │   ├── tasks/         # 主机名、ops 用户、sudo、SSH 密钥、apt 源、SSH 加固
    │   ├── templates/     # sources.list.j2（apt 源模板）
    │   ├── handlers/      # 更新 apt 缓存、重启 ssh
    │   ├── defaults/      # 可覆盖的默认变量（apt 镜像地址）
    │   └── vars/
    └── nginx/             # Nginx 部署角色
        ├── tasks/         # 安装、启动、渲染配置、部署首页
        ├── templates/     # nginx.j2
        ├── files/         # index.html
        └── handlers/      # 重启 nginx
```

## 功能说明

### init 角色（基础初始化）

| 步骤 | 说明 |
|------|------|
| 设置主机名 | 主机名与 inventory 中的名称保持一致（`hostnamectl set-hostname`） |
| 创建 ops 用户 | 指定 bash shell 并创建家目录 |
| 免密 sudo | 下发 `/etc/sudoers.d/ops-nopasswd`，使用 `visudo -cf` 预校验，防止语法错误导致 sudo 不可用 |
| SSH 密钥 | 将控制机 `~/.ssh/id_rsa.pub` 写入 ops 用户的 `authorized_keys` |
| apt 源 | 通过模板渲染 `/etc/apt/sources.list`（自动备份），变更后刷新 apt 缓存 |
| SSH 加固 | `PermitRootLogin prohibit-password` + `PasswordAuthentication no`，变更后重启 ssh |

### nginx 角色（Web 服务部署）

- 安装并启动 Nginx（开机自启）
- 使用 `nginx.j2` 模板渲染站点配置，变更自动重启
- 部署自定义首页到 `/var/www/html/`

## 环境要求

- 控制机：Ansible + `ansible.posix` collection（`ansible-galaxy collection install ansible.posix`）
- 受控机：Debian，Python 3.13（`/usr/bin/python3.13`）
- 控制机需存在 SSH 密钥对 `~/.ssh/id_rsa.pub`

## 使用方法

### 1. 修改主机清单

编辑 `inventory.ini`，填入你的主机 IP：

```ini
[web]
debian1 ansible_host=192.168.118.101
debian2 ansible_host=192.168.118.102
debian3 ansible_host=192.168.118.103

[web:vars]
ansible_user=ops
ansible_python_interpreter=/usr/bin/python3.13
```

### 2. 执行基础初始化

> 首次执行时 ops 用户尚不存在，需临时指定可用的连接用户（如 root）：

```bash
ansible-playbook Init.yaml -i inventory.ini -u root -k
```

### 3. 部署 Nginx

```bash
ansible-playbook site.yaml -i inventory.ini
```

如需使用加密变量：

```bash
ansible-playbook site.yaml -i inventory.ini --vault-password-file ~/.vault_pass
# 或交互式输入密码
ansible-playbook site.yaml -i inventory.ini --ask-vault-pass
```

### 4. 验证方法

```bash
# 1) 连通性：三台应全部返回 pong
ansible web -i inventory.ini -m ping

# 2) 服务可用：三台首页均能访问（curl 200）
curl http://192.168.118.101/
curl http://192.168.118.102/
curl http://192.168.118.103/

# 3) 初始化结果：主机名、ops 用户、sudo、SSH 加固
ssh ops@192.168.118.101 hostnamectl status
ansible web -i inventory.ini -m shell -a 'sudo -n true && echo sudo-ok'
ansible web -i inventory.ini -m shell -a 'grep -E "^(PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config'

# 4) 幂等性：再跑一遍，changed=0 即证明幂等
ansible-playbook Init.yaml -i inventory.ini
ansible-playbook site.yaml -i inventory.ini
```

预期：第 4 步 `PLAY RECAP` 中 `changed=0`、`failed=0`。

### 5. 常用检查命令

```bash
# 演练模式（不实际执行）
ansible-playbook site.yaml -i inventory.ini --check

# 查看任务执行明细
ansible-playbook site.yaml -i inventory.ini -v
```

## 安全说明

- `secrets.yml` 使用 **Ansible Vault** 加密（AES256），可安全提交到仓库；查看/编辑需使用 `ansible-vault view/edit secrets.yml`
- SSH 加固后会**禁用密码登录**，请确认密钥登录可用后再执行 init 角色，否则可能失去远程访问
- `.gitignore` 已排除 vim 交换文件（`*.wap`、`*.swp` 等）

## License

MIT-0
