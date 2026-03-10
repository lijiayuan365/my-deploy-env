# Verdaccio 私有 npm 仓库

> [!NOTE]
> 本项目使用 Docker 部署，旨在为团队提供一个轻量级、安全且高效的私有 npm 镜像库。支持自动代理 npmjs.org，并具备灵活的用户权限管理。

---

## 🚀 快速启动

### 启动服务
确保已安装 Docker 和 Docker Compose，然后在根目录执行：

```bash
docker compose up -d
```

服务启动后，通过浏览器访问：[http://localhost:4873](http://localhost:4873)

### ⚠️ 重要权限配置
由于 `storage` 目录映射到本地，容器内的 `verdaccio` 用户（UID `10001`）默认可能没有写入权限，这会导致发布包时报错。

**推荐方案（更安全）：**
```bash
sudo chown -R 10001:65533 ./verdaccio/storage
```

**简单方案：**
```bash
sudo chmod -R 777 ./verdaccio/storage
```

---

## 👤 用户管理

### 方式一：npm 命令行注册
如果配置允许自注册，可直接执行：

```bash
npm adduser --registry http://localhost:4873
```

### 方式二：手动管理（htpasswd）
管理员可以直接在 `./verdaccio/storage/htpasswd` 中添加用户。格式为 `用户名:密码哈希:邮箱`。

**添加示例：**
```bash
# 添加用户 'lee'，密码 '123456'
sudo sh -c 'echo "lee:$(openssl passwd -apr1 123456):lee@example.com" >> ./verdaccio/storage/htpasswd'

# 修正权限以确保容器可读
sudo chown 10001:10001 ./verdaccio/storage/htpasswd
```

---

## 🛠️ 客户端使用指导

### 管理源地址 (推荐使用 nrm)
```bash
# 添加私有源
nrm add my-npm http://localhost:4873

# 切换到私有源
nrm use my-npm
```

### 配置文件 (.npmrc)
你可以直接在项目根目录创建 `.npmrc` 文件来锁定源：

```text
registry=http://localhost:4873/
# 如果涉及 scoped 包
# @my-scope:registry=http://localhost:4873/
```

### 常用命令
| 操作 | 命令 |
| :--- | :--- |
| **登录** | `npm login --registry http://localhost:4873` |
| **发布** | `npm publish --registry http://localhost:4873` |
| **安装** | `npm install <package-name>` |

---

## ⚙️ 当前配置核心说明

基于 `config.yaml` 的当前配置摘要：

| 配置项 | 当前设定 | 业务含义 |
| :--- | :--- | :--- |
| `max_users` | `0` | **禁止自注册**。必须由管理员手动添加。 |
| `access` | `$all` | 所有人（包括匿名用户）均可下载包。 |
| `publish` | `$authenticated` | 仅登录用户可以发布新的包。 |
| `proxy` | `npmjs` | 在本地找不到包时，自动从 npmjs.org 拉取。 |
| `storage` | `/verdaccio/storage` | 数据持久化存储路径。 |

---

## ❓ 常见问题排查

### Publish 报错 404 "no such package available"
*   **现象**：执行 `npm publish` 时返回 404 错误。
*   **原因**：权限不足导致无法在 storage 目录创建文件夹。
*   **解决**：
    1. 检查权限：`ls -ld ./verdaccio/storage`
    2. 运行权限修复命令（见“快速启动”章节）。
    3. 重启容器：`docker compose restart verdaccio`

### 无法下载私有包
*   **检查**：确保用户已正确配置 registry 地址，或在 `.npmrc` 中指定了正确的 scope 映射。

sudo sh -c 'echo "lee:$(openssl passwd -apr1 123456):lee@openvision.cc" >> ./verdaccio/storage/htpasswd'