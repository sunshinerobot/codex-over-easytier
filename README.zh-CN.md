# Codex over EasyTier：我以为启动 EasyTier 就够了

这篇文章记录了我把 Codex 搬到远程 GPU 服务器上的一次排障过程。

一开始我以为：笔记本启动 EasyTier，VS Code 就能直接连接，Codex 自然也能联网。结果 VS Code 一会儿连不上，Codex 一会儿 `Reconnecting`，还出现了：

```text
stream disconnected before completion
```

后来我才明白，这其实是三条链路：

1. EasyTier 负责“笔记本能不能找到服务器”；
2. SSH RemoteForward 负责“服务器能不能借用笔记本的 Clash”；
3. VS Code/Codex 进程必须拿到正确的代理变量。

下面就是我最终整理出的通俗流程。

## 先说隐私：哪些内容不能公开

这次经历中最容易误传的是日志。下面这些内容不要放入 GitHub、Issue、聊天或截图：

- 服务器公网 IP 或 EasyTier 节点地址；
- EasyTier 网络密钥、服务器密码、笔记本 sudo 密码；
- ChatGPT 密码、验证码、登录 Token；
- GitHub Token；
- SSH 私钥内容；
- 可复用的完整主机指纹、真实用户名和内部网络名称。

公网 IP 不一定是“密码”，但公开后会暴露服务器入口，增加扫描和定向攻击风险。用户名和主机指纹通常不能单独登录，但会帮助别人识别目标。网络密钥、密码、Token 和私钥则属于高风险秘密。

因此本文只使用：

```text
<server-ip>
<server-user>
<easytier-network-name>
<easytier-secret>
<host-fingerprint>
```

如果秘密已经出现在公开日志里，应立即更换，而不是只删除那一行文字。

## 第一幕：我先确认笔记本代理

我在笔记本上启动 Clash，并在本地终端检查端口：

```bash
lsof -nP -iTCP:7890 -sTCP:LISTEN
```

这里的 `7890` 是笔记本上的 Clash HTTP 代理端口。不要把下面这段单独输入终端：

```text
127.0.0.1:7890
```

它只是地址，不是 Bash 命令。

也可以用 GitHub 做一个基础代理测试：

```bash
curl -sS --max-time 20 \
  -o /dev/null \
  -w 'github HTTP %{http_code}\n' \
  -x http://127.0.0.1:7890 \
  https://github.com
```

如果返回 `200`，说明这个基础 HTTP 代理测试成功。直接 `curl https://chatgpt.com` 返回 `403` 不一定说明代理坏了，因为没有浏览器会话和登录信息时，站点可能拒绝裸请求。

## 第二幕：启动 EasyTier

EasyTier 只负责私有网络可达，不负责代理服务器访问互联网。

在笔记本终端输入下面的模板。先把变量中的占位符替换为管理员提供的值：

```bash
EASYTIER_BIN="$HOME/Downloads/easytier-<version>/easytier-linux-x86_64/easytier-core"
ET_NETWORK_NAME='<easytier-network-name>'
ET_BOOTSTRAP='tcp://<bootstrap-server>:<port>'

read -rsp "EasyTier network secret: " ET_SECRET
echo

sudo "$EASYTIER_BIN" \
  -d \
  --network-name "$ET_NETWORK_NAME" \
  --network-secret "$ET_SECRET" \
  -p "$ET_BOOTSTRAP"

unset ET_SECRET ET_NETWORK_NAME ET_BOOTSTRAP EASYTIER_BIN
```

提示网络密钥时终端不显示字符是正常的。EasyTier 终端要保持运行；不用时回到这个终端按 `Ctrl+C`。

## 第三幕：先确认服务器可达

另开一个笔记本终端，填写服务器的 EasyTier 地址：

```bash
SERVER_IP='<server-ip>'
ping -c 2 "$SERVER_IP"
```

能收到回复，说明 EasyTier 的私有网络路径基本可用。丢包不一定代表完全失败，但如果完全没有回复，应先解决 EasyTier。

## 第四幕：配置 SSH，让 VS Code 自动转发代理

这一步是整个故事的转折点。

在笔记本上打开 SSH 配置：

```bash
nano ~/.ssh/config
```

在文件末尾加入模板，替换尖括号中的值：

```ssh
Host robot211-example
    HostName <server-ip>
    User <server-user>
    IdentityFile ~/.ssh/id_ed25519_<server-alias>
    IdentitiesOnly yes
    ExitOnForwardFailure yes
    RemoteForward 127.0.0.1:17891 127.0.0.1:7890
```

保存 `Ctrl+O`，按回车；退出 `Ctrl+X`。

这里有两个看起来一样、实际上属于不同机器的地址：

- 笔记本的 `127.0.0.1:7890`：Clash；
- 服务器的 `127.0.0.1:17891`：SSH 转发出来的代理入口。

以后服务器上的 Codex 只能使用 `17891`，不能直接使用 `7890`。

检查 SSH 配置是否被读取：

```bash
ssh -G robot211-example | grep -E '^(hostname|user|identityfile|remoteforward|exitonforwardfailure) '
```

首次 SSH 登录时，必须人工核对管理员提供的主机指纹；指纹不一致就停止，不要继续。

## 第五幕：我犯过的错误——重复建立 SSH 转发

最初我又在终端手动运行了一条带 `-R 127.0.0.1:17891:...` 的 SSH 命令，同时让 VS Code 连接同一个主机。结果：

- 手动 SSH 占用了服务器的 `17891`；
- VS Code 也想占用 `17891`；
- `ExitOnForwardFailure=yes` 让 VS Code 直接报告无法连接。

最终正确的做法是：

1. Clash 保持运行；
2. EasyTier 保持运行；
3. 只在 VS Code 执行 `Remote-SSH: Connect to Host...`；
4. 不再同时运行手动 `ssh -R`。

VS Code 会按照 `~/.ssh/config` 自动建立 RemoteForward。

## 第六幕：把服务器终端环境永久设为 17891

连接到服务器后，在服务器终端打开：

```bash
nano ~/.bashrc
```

在文件末尾粘贴：

```bash
# Laptop Clash proxy through SSH RemoteForward
export HTTP_PROXY=http://127.0.0.1:17891
export HTTPS_PROXY=http://127.0.0.1:17891
export http_proxy=http://127.0.0.1:17891
export https_proxy=http://127.0.0.1:17891
unset ALL_PROXY
unset all_proxy
export NO_PROXY=localhost,127.0.0.1,<server-ip>
export no_proxy="$NO_PROXY"
```

保存并让当前终端立即生效：

```bash
source ~/.bashrc
```

然后为 VS Code Server 建立启动环境：

```bash
mkdir -p ~/.vscode-server
nano ~/.vscode-server/server-env-setup
```

将文件内容替换为：

```bash
#!/bin/sh
export HTTP_PROXY=http://127.0.0.1:17891
export HTTPS_PROXY=http://127.0.0.1:17891
export http_proxy=http://127.0.0.1:17891
export https_proxy=http://127.0.0.1:17891
unset ALL_PROXY
unset all_proxy
export NO_PROXY=localhost,127.0.0.1,<server-ip>
export no_proxy="$NO_PROXY"
```

保存后执行：

```bash
chmod 700 ~/.vscode-server/server-env-setup
```

## 第七幕：修正本机 VS Code 的 JSON 配置

在本机 VS Code 执行：

```text
Ctrl+Shift+P
Preferences: Open User Settings (JSON)
```

本机用户设置中可以保留 Clash 的本地代理，但不要在全局 `terminal.integrated.env.linux` 中写入 `7890`。建议保留下面结构：

```json
{
    "http.proxy": "http://127.0.0.1:7890",
    "remote.SSH.remoteEnv": {
        "HTTP_PROXY": "http://127.0.0.1:17891",
        "HTTPS_PROXY": "http://127.0.0.1:17891",
        "http_proxy": "http://127.0.0.1:17891",
        "https_proxy": "http://127.0.0.1:17891",
        "NO_PROXY": "localhost,127.0.0.1,<server-ip>",
        "no_proxy": "localhost,127.0.0.1,<server-ip>"
    },
    "terminal.integrated.env.linux": {
        "PATH": "<remote-npm-bin>:${env:PATH}"
    }
}
```

这里的 `http.proxy` 是本机 VS Code 的代理；远程 Codex 使用的是 `remote.SSH.remoteEnv` 中的 `17891`。

连接到服务器的 VS Code 窗口后，再执行：

```text
Ctrl+Shift+P
Preferences: Open Remote Settings (JSON)
```

远程设置可以写：

```json
{
    "http.proxy": "http://127.0.0.1:17891",
    "http.proxySupport": "override"
}
```

改完后执行：

```text
Remote-SSH: Kill VS Code Server on Host...
```

选择目标主机，再重新连接。原因是已经启动的 Codex app-server 不会因为后来修改了 `.bashrc` 就自动改变环境。

## 第八幕：最终诊断命令

在 VS Code 的远程终端一次性执行：

```bash
printf 'HTTP_PROXY=%s\nHTTPS_PROXY=%s\nhttp_proxy=%s\nhttps_proxy=%s\n' \\
  "$HTTP_PROXY" "$HTTPS_PROXY" "$http_proxy" "$https_proxy"

ss -ltn | grep -F '127.0.0.1:17891'

curl --noproxy "" -sS --max-time 20 \\
  -o /dev/null \\
  -w 'github HTTP %{http_code}\n' \\
  -x http://127.0.0.1:17891 \\
  https://github.com

codex login status
```

预期结果：

- 四个代理变量都是 `http://127.0.0.1:17891`；
- `ss` 显示 `LISTEN`；
- GitHub 返回 `200`；
- Codex 显示已通过 ChatGPT 登录。

如果当前终端正确，但插件仍然重连，检查实际 Codex 进程：

```bash
for p in $(pgrep -u "$USER" -f 'codex|openai\.chatgpt' 2>/dev/null); do
  echo "--- PID $p ---"
  tr '\0' '\n' < "/proc/$p/environ" | \\
    grep -iE '^(HTTP|HTTPS|ALL|NO)_PROXY=' || true
done
```

我最后发现的真正问题就是：当前 shell 已经是 `17891`，但 Codex 进程仍然拿着大写的 `7890`。网络变量必须在“实际进程”里全部一致。

## 这次经历给我的六个结论

1. EasyTier 解决的是私网可达，不是外网代理。
2. 两台机器的 `127.0.0.1` 不是同一个地方。
3. VS Code 已经会根据 SSH 配置建立转发，不要再重复手动 `ssh -R`。
4. 只检查当前终端不够，必须检查 Codex app-server 的环境。
5. 代理修改后要重启 VS Code Server 和 Codex app-server。
6. 公开分享前必须脱敏；秘密一旦泄露就应该轮换。
