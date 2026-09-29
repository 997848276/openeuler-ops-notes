# day19 · Ansible 自动化运维入门（254 控 254 + 128）

> 本篇 = 仓库 day19。9/28 实操，对应手册 `Ansible命令解读与类比.md` §一~§九。
> 拓扑：254(oe-docker) 为控制节点 → 管 254 本机(local 连接) + 128(oe-base, devops)。Ansible 2.9.27（openEuler 24.03 dnf 源经典版）。工作区 `/root/ansible-lab/`。

## 一、安装与免密通路（Ansible 的地基）

```bash
python3 --version                         # /usr/bin/python3 -> Python 3.11.6（被控端前提满足）
dnf list available 'ansible*'              # ansible.noarch 2.9.27-8.oe2403（经典老版，教学够用）
dnf install -y ansible                     # 17 包一次过（cryptography/jinja2/paramiko/sshpass）
ansible --version                          # ansible 2.9.27, python 3.11.6
```

免密通路撞死锁（128 是 9/16 SSH 加固机，PasswordAuth no）：
```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519     # 254 root 生成钥匙
ssh-copy-id devops@192.168.252.128
# 指纹 yes 后【没提示输密码】直接 Permission denied (publickey,gssapi-keyex,...)
# 根因：ssh-copy-id 靠密码登入塞 key，密码通道被关 → 连门进不去（报错括号里无 password）
```
破局（从 128 控制台内侧放 key）：
```bash
# 254: cat ~/.ssh/id_ed25519.pub → 复制整行
# 128(devops):
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... root@oe-docker' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
restorecon -Rv ~/.ssh        # ★ SELinux Enforcing 老坑：动 ~/.ssh 必修标签 ssh_home_t
# 254 复验：ssh -o BatchMode=yes devops@192.168.252.128 'hostname; python3 --version'
# → oe-base + Python 3.11.6（通路全通）
```

> ssh 失败三态分诊：① `Host key verification failed` = 门口没登记指纹；② `Permission denied (publickey,...)` 看括号有无 password 判断密码通道活否；③ `refused/timeout` = 网络层。

## 二、sudoers.d 免密提权（become 的燃料）

```bash
# 128(devops, 最后一次输密码):
sudo tee /etc/sudoers.d/devops <<'EOF'
devops ALL=(ALL) NOPASSWD: ALL
EOF
sudo chmod 440 /etc/sudoers.d/devops
sudo visudo -c /etc/sudoers.d/devops      # /etc/sudoers.d/devops: parsed OK
# 254 复验: ansible oe128 -m shell -a 'sudo -n true && echo NOPASSWD-YES || echo NEED-PASSWORD' → NOPASSWD-YES
```
> ★ 坑：sudoers 写错 = sudo 全锁死。铁律：独立文件 + visudo -c 校验 + sudo tee 写入（不用 `>` 重定向，sudo 权限传不过去）。

## 三、inventory + 第一次点名 + ad-hoc 摸欧拉

```bash
mkdir -p /root/ansible-lab && cd /root/ansible-lab
cat > ansible.cfg <<'EOF'
[defaults]
inventory = ./inventory
EOF
cat > inventory <<'EOF'
[oe]
oe128 ansible_host=192.168.252.128 ansible_user=devops
[control]
localhost ansible_connection=local
EOF
ansible all -m ping
# 双机 SUCCESS {"ping": "pong"}（★ Ansible ping ≠ ICMP，是 ssh+python 全链路体检）
ansible oe128 -m shell -a 'uname -r; uptime'
# 6.6.0-145...oe2403.x86_64   ← 实锤 128 是 x86_64（国密加练走 openssl 软件层）
ansible oe128 -m setup -a "filter=ansible_distribution*"
# distribution="openEuler" version="24.03"
```
> ★ command 模块不走 shell（管道/分号报错）→ 要用管道上 shell 模块。

## 四、第一个 playbook：双机巡检（register + debug）

hosts: all 一条命令管全清单；register 存任务输出；debug 念结果。RECAP 双机 ok=5 changed=3。

## 五、幂等三跑曲线（copy 模块）

双机统一 /etc/motd。实测曲线：
| 跑次 | localhost | oe128 |
|---|---|---|
| 1 | ❌ failed（libselinux 缺失） | changed=1 首写 |
| 2 | ❌ | changed=1（属性收敛） |
| 3（补装 binding） | changed=1 | changed=0 |
| 4（--diff） | changed=0 | changed=0 |

★ 坑：`Aborting, target uses selinux but python bindings (libselinux-python) aren't installed!` → `dnf install -y python3-libselinux`。★ changed 异常 → 上 `--diff` 拿真差异（显示≠事实的 Ansible 版）。
幂等本质：RECAP 的 changed 是"该有的样子 vs 现在的样子"的差距，ok 是已达标。

## 六、四模块全家桶：nginx 双机部署

yum + copy + service + shell(curl, changed_when:false)。实测：oe128 yum=ok(已装跳过)、service=ok(在跑)；localhost 全 changed(新装新启)。RECAP 双机 HTTP=200。同一任务两机不同命运 = 声明式活教材。

## 坑清单（入门篇）
1. ssh-copy-id 撞 PasswordAuth no → 内侧放 key + restorecon
2. SELinux 动 ~/.ssh → restorecon
3. copy 缺 python3-libselinux → dnf 装
4. sudoers 写错锁死 → 独立文件 + visudo -c + sudo tee
5. shell 永远 changed → 查状态加 changed_when:false；command 不走管道
6. changed 异常 → --diff
7. playbook not found → 先查路径再查语法

## 面试一句话
> "我从零搭过 Ansible 管控：给加固机（PasswordAuth no）下发免密撞过 ssh-copy-id 死锁——密码通道关了只能从控制台内侧放公钥再 restorecon 修 SELinux；提权用 sudoers.d + visudo -c 防锁死。亲眼验证过幂等：copy 同文件跑四次 changed 从 1 到 0 稳定；中途踩过 SELinux 缺 python3-libselinux 致 copy 弃权，也学会 changed 异常上 --diff。最后 yum+copy+service 一个 playbook 双机 nginx，同一任务在已装机 ok、新机 changed——声明式不是概念是 RECAP 里看得见的数字。"
