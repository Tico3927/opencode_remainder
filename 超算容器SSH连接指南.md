# 并行智算云 容器云 SSH 连接完整指南

> 适用：ai.paratera.com 容器云 | GPU: RTX 5090, 4090, 3090
> 更新：2026-05-04

---

## 当前容器配置（可直接使用）

| 项目 | 值 |
|------|------|
| **容器名称** | kcs-mjhjmfwq |
| **容器 ID** | ackcs-00gjgshx |
| **站点** | 山东二区 |
| **计费模式** | 按量计费 |
| **GPU** | NVIDIA RTX 5090 × 1卡 × 32GB 显存 |
| **CPU** | 14 vCPU |
| **内存** | 120 GB |
| **操作系统** | Ubuntu 24.04 |
| **开发框架** | PyTorch 2.7.0 (镜像 ackci-sqjx000000 / PyTorch-25.03-py3) |
| **CUDA** | 13.0 (driver 580.82.07), 预装 CUDA 12.8+ |
| **SSH 地址** | ssh.bj8.bz1.paratera.com |
| **SSH 端口** | 2233 |
| **SSH 用户名** | root@ackcs-00gjgshx |
| **SSH 密码** | 每次从控制台「实例详情 → 获取SSH连接」重新获取 |

### 快速连接

```bash
# 密码需从控制台重新获取
ssh -p 2233 root@ackcs-00gjgshx@ssh.bj8.bz1.paratera.com
```

```python
# paramiko 自动化（密码需从控制台重新获取）
import paramiko
t = paramiko.Transport(('ssh.bj8.bz1.paratera.com', 2233))
t.connect(username='root@ackcs-00gjgshx', password='<控制台获取的密码>')
```

---

## 一、架构概述

```
用户本地机器
    │
    │ SSH (password auth)
    ▼
SSHPiper 隧道代理 ←─────────────────── 并行智算云基础设施
  (ssh.bj8.bz1.paratera.com:2233)
    │
    │ 根据用户名中的容器ID路由
    ▼
容器实例 (ackcs-xxxxxxxx)
  Ubuntu 24.04, CUDA 12.8+, PyTorch 2.7.0
  NVIDIA RTX 5090 32GB
```

**关键点**：SSH 连接到的是 SSHPiper 隧道代理，不是直接连容器。代理根据**用户名中的容器 ID** 将连接路由到正确的容器。

---

## 二、获取 SSH 凭证

### 步骤 1：登录控制台

```
https://ai.paratera.com
```
- 手机号 + 验证码登录
- 需要先完成「个人实名认证」

### 步骤 2：创建容器实例

1. 导航到「容器云」→「容器实例」→「创建」
2. 选择：资源类型 = **RTX5090**，GPU 数量 = 1，开发框架 = PyTorch 2.7.0
3. 等待实例启动

### 步骤 3：获取 SSH 连接信息

1. 进入实例详情页
2. 点击「SSH 连接」或「获取SSH连接」
3. 页面会显示：

```
IP地址:    ssh.bj8.bz1.paratera.com
端口:      2233
用户名:    root@ackcs-00gjgshx       ← 注意：用户名包含容器ID
密码:      eb30713a7eb992982...      ← 每次可能不同
```

### ⚠️ 重要

- **用户名格式不是 `root`，而是 `root@<容器ID>`**
- 密码每次点开「获取SSH连接」可能刷新，注意确认
- 实例关机后再次开机，SSH 地址/端口/用户名不变，密码可能变

---

## 三、SSH 连接方式

### 方式 A：浏览器终端（最简单）

直接在实例详情页点击「SSH 连接」→ 浏览器内打开终端，无需任何配置。

**适用**：临时操作、查看状态

---

### 方式 B：本地终端（需要处理密码输入）

#### Linux / macOS

需要 `sshpass`：

```bash
# 安装 sshpass
apt install sshpass        # Debian/Ubuntu
brew install sshpass       # macOS

# 连接
sshpass -p '你的密码' ssh -o StrictHostKeyChecking=no \
    -p 2233 root@容器ID@ssh.bj8.bz1.paraterra.com

# 示例
sshpass -p 'eb30713a7eb9929820d765bc626b3ccf' ssh \
    -o StrictHostKeyChecking=no \
    -p 2233 \
    root@ackcs-00gjgshx@ssh.bj8.bz1.paraterra.com
```

#### Windows

Windows 原生 `ssh` 不支持管道密码，有三种方案：

##### 方案 1：Python + Paramiko（推荐）

```python
import paramiko, socket, time

# 连接参数
HOST = 'ssh.bj8.bz1.paratera.com'
PORT = 2233
USER = 'root@ackcs-00gjgshx'       # 注意：包含容器ID
PASS = '你的密码'

# 建立连接
transport = paramiko.Transport((HOST, PORT))
transport.connect(username=USER, password=PASS)

# 执行命令
def run(cmd, timeout=30):
    chan = transport.open_session(timeout=timeout)
    chan.exec_command(cmd)
    out = b''
    while not chan.exit_status_ready():
        if chan.recv_ready():
            out += chan.recv(65536)
        time.sleep(0.05)
    out += chan.recv(65536)
    out += chan.recv_stderr(65536)
    chan.close()
    return out.decode('utf-8', errors='replace')

# 示例：查看 GPU
print(run('nvidia-smi --query-gpu=name,memory.total --format=csv,noheader'))

transport.close()
```

##### 方案 2：密钥认证（一劳永逸）

1. 生成密钥对
```bash
ssh-keygen -t ed25519 -f ~/.ssh/paratera_key -N ""
```

2. 通过浏览器终端登录，添加公钥
```bash
mkdir -p ~/.ssh && echo "你的公钥内容" >> ~/.ssh/authorized_keys
```

3. 后续使用密钥连接
```bash
ssh -i ~/.ssh/paratera_key -p 2233 root@容器ID@ssh.bj8.bz1.paraterra.com
```

---

### 方式 C：SFTP 文件传输

```python
import paramiko

transport = paramiko.Transport(('ssh.bj8.bz1.paratera.com', 2233))
transport.connect(username='root@ackcs-00gjgshx', password='密码')

sftp = paramiko.SFTPClient.from_transport(transport)

# 上传文件
sftp.put('本地文件.py', '/root/远程文件.py')

# 下载文件
sftp.get('/root/远程文件.pth', '本地保存路径.pth')

# 上传文件对象（不落盘）
from io import BytesIO
sftp.putfo(BytesIO(b'文件内容'), '/root/test.txt')

sftp.close()
transport.close()
```

---

## 四、常见操作

### 检查 GPU

```bash
nvidia-smi
nvidia-smi --query-gpu=name,memory.total,memory.used,utilization.gpu,power.draw --format=csv
```

### 检查环境

```bash
python3 --version
python3 -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.device_count())"
```

### 后台运行训练

```bash
nohup python3 -u train.py > train.log 2>&1 &
# -u 参数必须！否则 Python 输出缓冲，日志为空
```

### 查看进程

```bash
ps aux | grep python3
```

### 终止进程

```bash
pkill -f train.py
# 或
kill -9 <PID>
```

### 查看磁盘

```bash
df -h /            # 系统盘 30GB
ls /root/shared-nvme   # 共享存储（50GB 免费）
```

---

## 五、常见问题与解决

| 问题 | 原因 | 解决 |
|------|------|------|
| `EOFError` 连接后立即断开 | 用户名不含容器ID | 用户名必须是 `root@容器ID` |
| `Authentication failed` | 密码错误或过期 | 重新在控制台获取SSH密码 |
| `train.log` 文件 0 字节 | Python 输出缓冲 | 启动命令加 `-u` 参数 |
| GPU 利用率低 | MCTS 自对弈 CPU 密集 | 正常现象，训练阶段 GPU 利用率上升 |
| 中文文件名/参数乱码 | SSH 通道编码问题 | 在远程机上直接创建脚本，避免传输中文参数 |
| `paramiko.ssh_exception.SSHException` | paramiko 版本兼容 | 使用 `Transport` 而非 `SSHClient` |

---

## 六、完整自动化脚本模板

```python
#!/usr/bin/env python3
"""
通用 paramiko SSH 自动化工具 for 并行智算云容器云
"""
import paramiko, time, sys

class ParaContainer:
    def __init__(self, host, port, container_id, password):
        self.transport = paramiko.Transport((host, port))
        self.transport.connect(
            username=f'root@{container_id}',
            password=password
        )
        self._sfp = None

    def run(self, cmd, timeout=30):
        """执行命令并返回输出"""
        chan = self.transport.open_session(timeout=timeout)
        chan.exec_command(cmd)
        out = b''
        while not chan.exit_status_ready():
            if chan.recv_ready():
                out += chan.recv(65536)
            time.sleep(0.05)
        out += chan.recv(65536)
        out += chan.recv_stderr(65536)
        chan.close()
        return out.decode('utf-8', errors='replace')

    @property
    def sftp(self):
        if self._sfp is None:
            self._sfp = paramiko.SFTPClient.from_transport(self.transport)
        return self._sfp

    def upload(self, local_path, remote_path):
        self.sftp.put(local_path, remote_path)

    def download(self, remote_path, local_path):
        self.sftp.get(remote_path, local_path)

    def close(self):
        if self._sfp:
            self._sfp.close()
        self.transport.close()


# ========== 使用示例 ==========
if __name__ == '__main__':
    c = ParaContainer(
        host='ssh.bj8.bz1.paraterra.com',
        port=2233,
        container_id='ackcs-00gjgshx',
        password='你的密码'
    )

    # 检查GPU
    print(c.run('nvidia-smi --query-gpu=name,memory.total --format=csv,noheader'))

    # 上传并运行训练
    c.upload('train.py', '/root/train.py')
    c.run('nohup python3 -u /root/train.py > /root/train.log 2>&1 &')

    c.close()
```

---

## 七、集群 (Slurm) vs 容器云对比

| | 集群 N50R5 | 容器云 |
|------|:--:|:--:|
| SSH 方式 | `ssh 用户名@ln07` | `ssh root@容器ID@ssh.bj8.bz1.paraterra.com:2233` |
| 任务提交 | Slurm (sbatch/salloc) | 直接运行，无需调度器 |
| GPU | 5090 × 8卡/节点 | 5090/4090/3090 按需 |
| 数据存放 | `~/run` (300GB) | `~/shared-nvme` (50GB免费) |
| 连接方式 | 纯 SSH | SSH + WebSSH + Jupyter + VSCode |
| 适用场景 | 批量计算任务 | 交互开发、持续训练 |

---

## 八、相关链接

- 控制台：https://ai.paratera.com
- 文档中心：https://ai.paratera.com/document
- 容器云指南：https://ai.paratera.com/document/container
- 集群指南：https://ai.paratera.com/document/cluster
- 客服：sales@paratera.com / 010-82780567
