# lazy-taskchampion

懒猫微服 lpk 包装：[GothenburgBitFactory/taskchampion-sync-server](https://github.com/GothenburgBitFactory/taskchampion-sync-server)

> **TaskChampion Sync Server** 是 Taskwarrior (3.x+) 的官方同步服务端。
> 客户端把任务用 `sync.encryption_secret` 端到端加密后上传，服务端
> 只存密文增量 —— 多台机器跑 `task` / `vit` 时任务数据保持一致。

## 安装

通过懒猫微服「应用商店」搜 *TaskChampion Sync Server* 安装。

- 子域名：`https://taskchampion.<your-box>.heiyu.space`
- 可选参数：`CLIENT_ID` —— 把服务限定到某个 UUID 才能注册；留空 = 允许任意 UUID（推荐，安全交给加密 secret）

## 客户端配置

### Taskwarrior 3.x+ (`task` CLI)

```bash
# 每台机器一个 UUID
CID=$(uuidgen)

# 所有 replica 共用同一个加密 secret —— 一旦确定就别再改
SEC=$(pwgen -s 32 1)

task config sync.server.url       https://taskchampion.<your-box>.heiyu.space
task config sync.server.client_id $CID
task config sync.encryption_secret $SEC

task sync   # 首次推送
```

### vit（Taskwarrior 终端 UI）

vit **不做**自己的同步 —— 它是 `task` 的 curses 前端，按大写 `S` 等价于
shell 里跑 `task sync`，读的是同一份 `~/.taskrc`。配好上面的 taskwarrior
三件套，vit 就自动跟同一台 server 同步。

想让 vit 启动/退出时自动同步，编辑 `~/.vit/config.ini`：

```ini
[vit]
sync_on_start = true
sync_on_end = true
```

### 多设备同步

- 每台机器一个**不同**的 `sync.server.client_id`（UUID）
- 所有机器共用**同一个** `sync.encryption_secret`
- 在每台机器跑 `task sync`，本地 replica 自动跟服务端密文合并

## 数据

- 服务端只存 SQLite + 加密 blob，落在 `/lzcapp/var/persist`，重装应用不丢
- 任务正文加密在客户端 —— 服务端看不到明文
- 备份：复制 `/lzcapp/var/persist` 目录即可

## 资源占用

- 服务端进程：一个 Rust 二进制 + SQLite，常驻 < 30MB 内存
- 网络：每次 `task sync` 几 KB ~ 几十 KB（增量）

## License 与重打包说明

- 上游项目：[`GothenburgBitFactory/taskchampion-sync-server`](https://github.com/GothenburgBitFactory/taskchampion-sync-server)（MIT）
- 本仓库做的事：**retag** —— 引用上游官方发布的 GHCR 镜像，**不修改源码**
- 镜像 digest 在 [`docker/Dockerfile`](docker/Dockerfile) 里按 sha256 锁定

## 升级

```sh
# 在 Dockerfile 里把 digest 改成最新一版（解析命令在 CLAUDE.md）
# bump version 后打 tag 触发 release.yml
git tag v0.0.2
git push --tags
```
