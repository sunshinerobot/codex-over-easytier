# Codex over EasyTier: I Thought Starting EasyTier Was Enough

This is a sanitized story about moving Codex onto a remote GPU server.

At first I thought the recipe was simple: start EasyTier on the laptop, connect VS Code, and let Codex work. Instead, VS Code alternated between connection failures and Codex showed:

```text
stream disconnected before completion
```

The real setup had three separate paths:

1. EasyTier answered: “Can the laptop reach the server?”
2. SSH RemoteForward answered: “Can the server borrow the laptop's Clash proxy?”
3. The VS Code and Codex processes had to receive the correct proxy variables.

This guide tells the story as a sequence you can follow.

## Privacy first: what must never be published

Do not put any of these in GitHub, an issue, chat, or a screenshot:

- the server's public IP or EasyTier node address;
- the EasyTier network secret, server password, or laptop sudo password;
- ChatGPT passwords, verification codes, or login tokens;
- GitHub tokens;
- SSH private-key contents;
- reusable full host fingerprints, real usernames, or internal network names.

A public IP is not a password, but it exposes an entry point and can increase scanning and targeted-attack risk. A username or host fingerprint normally cannot log anyone in by itself, but it helps identify a target. Network secrets, passwords, tokens, and private keys are high-risk credentials.

This guide uses placeholders only:

```text
<server-ip>
<server-user>
<easytier-network-name>
<easytier-secret>
<host-fingerprint>
```

If a secret has appeared in a public log, rotate it immediately; deleting the line is not enough.

## Act I: I checked the laptop proxy first

I started Clash on the laptop and checked its local HTTP port:

```bash
lsof -nP -iTCP:7890 -sTCP:LISTEN
```

Here, `7890` belongs to the laptop's Clash proxy. Do not type this by itself:

```text
127.0.0.1:7890
```

That is an address, not a Bash command.

A basic proxy test through GitHub is useful:

```bash
curl -sS --max-time 20 \
  -o /dev/null \
  -w 'github HTTP %{http_code}\n' \
  -x http://127.0.0.1:7890 \
  https://github.com
```

An HTTP `200` means this basic proxy test worked. A raw unauthenticated request to `chatgpt.com` may still return `403`; that does not, by itself, prove that the proxy is broken.

## Act II: I started EasyTier

EasyTier provides private reachability. It does not become an Internet proxy.

On the laptop, I used this template after replacing the placeholders with administrator-provided values:

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

The secret is intentionally hidden while typing. Keep this terminal running; press `Ctrl+C` there when the connection is no longer needed.

## Act III: I checked server reachability

In a second laptop terminal, I filled in the EasyTier address and pinged the server:

```bash
SERVER_IP='<server-ip>'
ping -c 2 "$SERVER_IP"
```

A reply means the private network path is basically working. Packet loss is not automatically fatal, but no replies at all means EasyTier needs attention first.

## Act IV: I configured SSH to forward the proxy automatically

This was the turning point.

On the laptop, I opened the SSH configuration:

```bash
nano ~/.ssh/config
```

At the end, I added this template:

```ssh
Host robot211-example
    HostName <server-ip>
    User <server-user>
    IdentityFile ~/.ssh/id_ed25519_<server-alias>
    IdentitiesOnly yes
    ExitOnForwardFailure yes
    RemoteForward 127.0.0.1:17891 127.0.0.1:7890
```

Save with `Ctrl+O`, press Enter, and exit with `Ctrl+X`.

Two addresses now look similar but belong to different machines:

- laptop `127.0.0.1:7890`: Clash;
- server `127.0.0.1:17891`: the SSH-forwarded proxy endpoint.

Codex on the server must use `17891`, never the laptop-only `7890`.

To verify that SSH reads the configuration:

```bash
ssh -G robot211-example | grep -E '^(hostname|user|identityfile|remoteforward|exitonforwardfailure) '
```

On the first login, manually compare the host fingerprint with the administrator-provided value. Stop if it does not match.

## Act V: the mistake—creating a duplicate SSH forward

I initially ran a manual SSH command with `-R 127.0.0.1:17891:...` while VS Code was also connecting to the same host. The result was predictable:

- manual SSH occupied server port `17891`;
- VS Code also tried to occupy `17891`;
- `ExitOnForwardFailure=yes` made VS Code report a connection failure.

The final workflow is simpler:

1. Keep Clash running.
2. Keep EasyTier running.
3. Use only `Remote-SSH: Connect to Host...` in VS Code.
4. Do not run a second manual `ssh -R` at the same time.

VS Code creates the RemoteForward from `~/.ssh/config`.

## Act VI: I made the server environment permanently use 17891

After connecting to the server, I opened:

```bash
nano ~/.bashrc
```

At the end I pasted:

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

I applied it to the current shell with:

```bash
source ~/.bashrc
```

Then I prepared the VS Code Server startup environment:

```bash
mkdir -p ~/.vscode-server
nano ~/.vscode-server/server-env-setup
```

I replaced the file with:

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

Then I ran:

```bash
chmod 700 ~/.vscode-server/server-env-setup
```

## Act VII: I fixed the laptop VS Code JSON settings

In local VS Code I opened:

```text
Ctrl+Shift+P
Preferences: Open User Settings (JSON)
```

The laptop may keep its local Clash proxy, but the global `terminal.integrated.env.linux` must not inject `7890` into remote processes. The relevant structure should look like this:

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

Here `http.proxy` is for local VS Code, while remote Codex uses `17891` from `remote.SSH.remoteEnv`.

In the connected remote VS Code window, I also opened:

```text
Ctrl+Shift+P
Preferences: Open Remote Settings (JSON)
```

The remote settings can contain:

```json
{
    "http.proxy": "http://127.0.0.1:17891",
    "http.proxySupport": "override"
}
```

After changing settings, I ran:

```text
Remote-SSH: Kill VS Code Server on Host...
```

I selected the host and reconnected. A running Codex app-server does not change its environment just because `.bashrc` was edited later.

## Act VIII: the final diagnostic command

In the VS Code remote terminal, I ran:

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

Expected results:

- all four proxy variables are `http://127.0.0.1:17891`;
- `ss` shows `LISTEN`;
- GitHub returns `200`;
- Codex reports a ChatGPT login.

If the shell is correct but the extension still reconnects, I inspect the actual Codex process:

```bash
for p in $(pgrep -u "$USER" -f 'codex|openai\.chatgpt' 2>/dev/null); do
  echo "--- PID $p ---"
  tr '\0' '\n' < "/proc/$p/environ" | \\
    grep -iE '^(HTTP|HTTPS|ALL|NO)_PROXY=' || true
done
```

The final root cause in my case was that the current shell used `17891`, but the Codex process still had uppercase `7890`. Every proxy variable must agree inside the real process.

## Six conclusions I kept

1. EasyTier provides private reachability, not an Internet proxy.
2. `127.0.0.1` means the machine where the command is running.
3. When VS Code uses the SSH configuration, do not create a duplicate `ssh -R`.
4. Inspect the real process environment, not only the current shell.
5. Restart VS Code Server and Codex app-server after proxy changes.
6. Sanitize before publishing; rotate any secret that appeared in a log.
