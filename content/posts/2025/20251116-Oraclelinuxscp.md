+++
date = '2025-11-16T08:31:44+03:00'
draft = true
title = 'Oracle Linux：scp 能用但 sftp 无法连接的解决记录（完整排查指南）'
+++

🛠 Oracle Linux：scp 能用但 sftp 无法连接的解决记录（完整排查指南）

在部署 Oracle Linux 服务器时，我遇到一个奇怪的问题：
	•	使用 scp 可以正常通过 SSH 复制文件
	•	使用 sftp（无论是命令行还是 FileZilla）却始终报错：

Fatal: unable to initialise SFTP on server: could not connect

这篇文章详细记录了从排查到最终解决问题的整个过程，希望对遇到相同问题的朋友有所帮助。

⸻

📌 现象与错误日志

从本地使用 SFTP 客户端连接 Oracle Linux 服务器时，日志输出：

Connected to <server-ip>
Fatal: unable to initialise SFTP on server: could not connect

但是 scp 却正常工作。

这说明：
	•	SSH 服务正常
	•	用户、密钥、端口都正常
	•	唯独 SFTP 子系统无法初始化

⸻

🔍 1. 初步排查：确认 SFTP 子系统配置

Oracle Linux 使用 OpenSSH，SFTP 子系统通常由 sshd_config 中以下内容提供：

sudo grep -i sftp /etc/ssh/sshd_config

输出：

Subsystem sftp /usr/libexec/openssh/sftp-server

看似没问题，但路径是否真实存在仍需确认：

ls -l /usr/libexec/openssh/sftp-server

部分 Oracle 镜像中，这个路径可能不存在（尤其是升级后），导致客户端初始化 SFTP 失败。

⸻

🔍 2. 服务器本地测试 SFTP

为了确认问题是不是客户端导致的，在服务器上直接执行：

sftp opc@localhost

得到：

Permission denied (publickey,gssapi-keyex,gssapi-with-mic)
Couldn't read packet: Connection reset by peer

这是因为本地执行时没有带私钥，认证失败，但明确说明：

✔ SFTP 请求确实进入 sshd
✔ 但子系统可能初始化失败或路径错误

⸻

🔧 3. 问题根因：SFTP 子系统可执行路径不匹配

系统配置了：

Subsystem sftp /usr/libexec/openssh/sftp-server

但该文件很可能：
	•	不存在
	•	无执行权限
	•	与当前 OpenSSH 版本不兼容

最终导致客户端无法初始化 SFTP。

⸻

🛠 4. 最终方案：使用 internal-sftp 替代外部可执行文件

为避免路径匹配问题，可以使用 OpenSSH 内置的 SFTP 子系统：

⸻

✔ Step 1：备份原配置

sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F)


⸻

✔ Step 2：移除旧配置并添加 internal-sftp

sudo sed -i '/^[# ]*Subsystem[ \t]\+sftp/d' /etc/ssh/sshd_config
echo 'Subsystem sftp internal-sftp' | sudo tee -a /etc/ssh/sshd_config

此时文件中应该只有一行：

Subsystem sftp internal-sftp


⸻

✔ Step 3：检查配置并重启

sudo sshd -t
sudo systemctl restart sshd


⸻

✔ 5. 测试成功！

本地使用 SFTP 客户端：

sftp -i ~/.ssh/ssh_key_oracle opc@<server-ip>

成功进入：

sftp>

FileZilla 也可正常连接，问题彻底解决 🎉

⸻

📘 总结：为什么 scp 能用，但 sftp 不行？

功能	工作机制	受影响？
scp	使用 SSH shell 传输文件	❌ 不依赖 SFTP 子系统，所以正常
sftp	依赖 SFTP 子系统 (Subsystem sftp)	✅ 路径错误会导致无法初始化

因此，当：
	•	Subsystem sftp 配置缺失
	•	指定的 sftp-server 路径不存在
	•	权限问题或 SELinux 限制
	•	多个冲突的 SFTP 配置

都会导致 scp 正常但 sftp 失败。

最稳妥的解决方案就是：

Subsystem sftp internal-sftp

稳定、高兼容性、无需担心路径变化。

⸻

🏁 结语

这是一次典型的“SSH 能连但 SFTP 不工作”的问题，由于 SFTP 子系统依赖额外配置，因此更容易出错。

希望这篇排查记录能帮助到有类似问题的朋友。如果你在 Oracle Linux / OpenSSH / SFTP 配置方面还有问题，也欢迎继续讨论 👇

我可以帮你写：
	•	一键配置脚本
	•	更安全的 SFTP chroot 环境
	•	仅允许用户 SFTP 的最小权限方案

随时告诉我！