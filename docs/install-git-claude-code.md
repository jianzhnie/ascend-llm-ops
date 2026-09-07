# 软件安装记录：git 与 Claude Code

> 环境：openEuler 22.03 (LTS-SP4) / aarch64
> 网络：无外网直连，需通过本地代理 `http://127.0.0.1:7897` 访问外网
> 整理日期：2026-09-07

---

## 一、环境要点（两个共同的坑）

### 1. 外网必须走代理

用户 shell 已配置代理环境变量（`http_proxy` / `https_proxy` = `http://127.0.0.1:7897`），
所以普通用户执行 `curl`、`npm` 等可以正常访问外网。

### 2. sudo 会清空代理环境变量

`sudo` 默认 `env_reset`，root 环境下代理变量丢失，表现为：

```
Curl error (6): Couldn't resolve host name ... [Could not resolve host: repo.huaweicloud.com]
```

**解决办法**：用 `env` 显式把代理变量传进 sudo 命令：

```bash
echo '<sudo密码>' | sudo -S env http_proxy=http://127.0.0.1:7897 https_proxy=http://127.0.0.1:7897 <命令>
```

---

## 二、安装 git（yum + 华为云镜像）

### 1. 切换 yum 镜像源（官方源太慢，实测仅 0.17 MB/s）

各镜像实测下载速度：

| 镜像 | 速度 |
|---|---|
| 华为云 repo.huaweicloud.com | ~3.5 MB/s ✅ 最快 |
| 阿里云 mirrors.aliyun.com | ~2.7 MB/s |
| 中科大 mirrors.ustc.edu.cn | ~1.2 MB/s |
| 官方 repo.openeuler.org | ~0.17 MB/s |

切换命令（同时注释 metalink，防止重定向回慢速镜像）：

```bash
sudo cp /etc/yum.repos.d/openEuler.repo /etc/yum.repos.d/openEuler.repo.bak
sudo sed -i 's|baseurl=http://repo.openeuler.org|baseurl=https://repo.huaweicloud.com/openeuler|g' /etc/yum.repos.d/openEuler.repo
sudo sed -i 's|^metalink=|#metalink=|g' /etc/yum.repos.d/openEuler.repo
```

如需还原：`sudo cp /etc/yum.repos.d/openEuler.repo.bak /etc/yum.repos.d/openEuler.repo`

### 2. 安装 git

```bash
echo '<sudo密码>' | sudo -S env http_proxy=http://127.0.0.1:7897 https_proxy=http://127.0.0.1:7897 yum install -y git
```

### 3. 验证

```bash
git --version
# git version 2.33.0
```

### 注意事项

- **不要并行跑多个 yum**：会互相等锁（"Another app is currently holding the yum lock"），
  表现为卡住不动。遇到时先 `ps -ef | grep yum` 确认，再杀掉残留进程。
- base 环境的 conda 目录无写权限，`conda install git` 到 base 会失败；如需 conda 路线，
  应建独立环境：`conda create -y -n gittools -c conda-forge git`（首次使用需先
  `conda tos accept --override-channels --channel <频道URL>` 接受服务条款）。

---

## 三、安装 Claude Code（npm + npmmirror 镜像）

### 为什么不用官方脚本

官方安装脚本 `curl -fsSL https://claude.ai/install.sh | bash` 的二进制下载源是
`https://downloads.claude.ai`，走代理实测仅 ~178 KB/s（百 MB 文件需 10 分钟以上）。

### 安装命令（推荐，国内 CDN 可达几 MB/s）

前置条件：node v20.18.0 / npm 10.8.2（已装于 `/usr/local/bin`）。

```bash
echo '<sudo密码>' | sudo -S env "PATH=$PATH" http_proxy=http://127.0.0.1:7897 https_proxy=http://127.0.0.1:7897 \
  npm install -g @anthropic-ai/claude-code --registry=https://registry.npmmirror.com
```

说明：
- `--registry=https://registry.npmmirror.com`：使用 npmmirror（淘宝镜像）加速。
- 必须传 `PATH` 和代理变量（sudo 清环境），否则找不到 npm 或无法联网。
- npm 全局前缀是 `/usr/local`，普通用户无写权限，所以用 sudo。

### 验证

```bash
which claude && claude --version
# /usr/local/bin/claude
# 2.1.197 (Claude Code)
```

### 升级

```bash
# 方式一：重新执行上面的 npm 安装命令
# 方式二：claude 自带的更新
claude update
```

### 注意事项

- npmmirror 的版本可能略滞后于官方（如官方 2.1.263 vs 镜像 2.1.197），不影响日常使用。
- 本方法适用于所有 npm 全局包的加速安装，套用 `--registry=https://registry.npmmirror.com` 即可。

---

## 四、集群节点免密登录检查（附录）

`node.txt` 中 16 个节点（10.1.0.x）已确认：本机到各节点、以及节点间任意两两 SSH 免密登录均正常。

批量检查命令（注意 `ssh -n`，否则循环读 stdin 会被 ssh 吃掉，只处理第一行）：

```bash
ips=$(awk '{print $1}' node.txt | tr '\n' ' ')
for src in $ips; do
  timeout 10 ssh -n -o BatchMode=yes -o ConnectTimeout=5 "$src" hostname >/dev/null 2>&1 \
    && echo "OK   $src" || echo "FAIL $src"
done
```

注意：16 节点并发互测（256 连接）偶发失败属正常现象，是 sshd 的 `MaxStartups` 并发限流，
并非免密配置缺失；串行复测全部通过。如需调大：`/etc/ssh/sshd_config` 中 `MaxStartups 100:30:200`。

---

## 五、批量修改目录属主（chown 并行加速）

### 背景

`/home/jianzhnie/llmtuner` 属主为 root，普通用户无写权限（连新建文档目录都会
`Permission denied`）。需将整棵目录树改为 `jianzhnie:jianzhnie`。

### 单条 `chown -R` 的问题

`chown -R` 是单线程串行处理，且 `/home` 是 200T 共享存储（网络文件系统），
文件量大时会非常慢。

### 并行做法（推荐，实测整棵树约 6 秒）

用 `find | xargs -P` 按批次多进程并行执行 chown：

```bash
echo '<sudo密码>' | sudo -S bash -c '
chown jianzhnie:jianzhnie /home/jianzhnie/llmtuner
find /home/jianzhnie/llmtuner -mindepth 1 -print0 | xargs -0 -P 16 -n 2000 chown -h jianzhnie:jianzhnie
'
```

说明：
- `-P 16`：16 路并行；`-n 2000`：每个 chown 进程处理 2000 个路径，摊薄进程启动开销。
- `-h`：修改符号链接自身的属主，而非其指向的目标。
- `-print0` / `-0`：以 NUL 分隔，兼容含空格等特殊字符的文件名。

### 验证

```bash
ls -l /home/jianzhnie/llmtuner
find /home/jianzhnie/llmtuner -maxdepth 3 ! -user jianzhnie -not -path "*/.snapshot*" | head
# 无输出即全部已属 jianzhnie
```

### 注意事项

- `.snapshot` 目录会报 `chown: ... Permission denied`——这是共享存储系统自带的
  只读快照目录，root 也无法修改，属正常现象，忽略即可。
- 通用性：该方法适用于任何大目录树的批量属主/权限修改，把 `chown -h` 换成
  `chmod` 等命令同样适用。
