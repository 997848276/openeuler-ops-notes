# Day 02 · dnf / rpm 深度
- dnf history undo：删掉 nginx 后用 ID 回滚，14 个包全部恢复
- systemctl enable 的实质：在 multi-user.target.wants/ 下建软链接
- nginx master/worker 架构：worker 数 = CPU 核数（本机 2 核 2 个 worker）
- dnf download --resolve：内网隔离环境离线部署的关键手法
- rpm -Va：校验系统文件是否被篡改，配置文件出现标记属正常
