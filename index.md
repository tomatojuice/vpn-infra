# 🏠 Home VPN / Proxy Infrastructure

Rocky Linux 10 上に構築した、自宅向け VPN / Proxy 基盤です。

WireGuard、Xray、Nginx、dnsmasq、Let's Encrypt、Cloudflare DDNS を組み合わせ、
Ansible で再構築できるようにしています。

This is a self-hosted VPN / proxy infrastructure project running on Rocky Linux 10.

---

## ✨ Features / 主な特徴

- 🔐 WireGuard VPN
- 🧰 wg-easy
- 🌐 Xray
- 📡 VLESS + WebSocket + TLS
- 🔁 Nginx Reverse Proxy
- 🧭 Split DNS with dnsmasq
- 🌍 IPv6 ULA
- 🔒 Let's Encrypt
- ☁️ Cloudflare DNS / DDNS
- 🧱 firewalld
- 🐳 Docker / Docker Compose
- 🤖 Ansible automation
- ✅ Deployment verification

---

## 🗺️ Architecture / 構成

```text
                    Internet
                        │
                 Cloudflare DNS
                        │
                  Dynamic IPv4
                        │
                    Home Router
                ┌───────┴───────┐
                │               │
           UDP 51820         TCP 443
                │               │
           WireGuard           Nginx
                │          ┌────┴──────────────┐
             wg-easy       │                   │
                         wg-admin.*          xray.*
                            │                   │
                     127.0.0.1:51821      /secret-tunnel
                                                │
                                         127.0.0.1:10080
                                                │
                                               Xray
                                                │
                                             Internet
```

---

## 🔐 WireGuard

WireGuard は `wg-easy` を Docker で動かしています。

管理画面は直接公開せず、

```text
127.0.0.1:51821
```

にのみバインドし、Nginx 経由でアクセスします。

主な用途:

- 自宅 LAN へのアクセス
- 自宅サーバーへの接続
- Full Tunnel VPN
- PC / Smartphone / Mac からの利用

---

## 🌐 Xray

Xray は WireGuard とは別系統の Proxy として動作します。

```text
VLESS
+
WebSocket
+
TLS
```

通信経路:

```text
Client
  │
TCP 443
  │
Nginx
  │
/secret-tunnel
  │
Xray
  │
Internet
```

TLS 終端は Nginx が担当し、
Xray の内部ポートはインターネットへ直接公開しません。

---

## 🧭 Split DNS

LAN 内では dnsmasq を使い、
公開ドメインをローカル IP に解決します。

```text
LAN
xray.example.com
        → Local IP

Internet
xray.example.com
        → Public IPv4
```

これにより Hairpin NAT に依存せず、
LAN 内外で同じホスト名を利用できます。

---

## ☁️ Cloudflare DDNS

自宅回線の動的 IPv4 を Cloudflare DNS に自動反映します。

```text
Current Public IPv4
        │
        ▼
Compare with Cloudflare
        │
   ┌────┴─────┐
   │          │
 Same      Changed
   │          │
 No update   Update
```

systemd timer から定期実行します。

---

## 🤖 Ansible

この環境は Ansible で自動構築できます。

```text
playbooks/
├── 01_base.yml
├── 02_network.yml
├── 03_docker.yml
├── 04_wg_easy.yml
├── 05_xray.yml
├── 06_dnsmasq.yml
├── 07_certbot.yml
├── 08_nginx.yml
├── 09_ddns.yml
├── 10_firewalld.yml
├── 11_verify.yml
└── site.yml
```

全体実行:

```bash
ansible-playbook \
  -i inventory/hosts.ini \
  playbooks/site.yml
```

---

## ♻️ Idempotency / 冪等性

設定変更後は `changed > 0` になりますが、
変更がない状態で再実行すると、

```text
changed=0
failed=0
```

となることを確認しています。

---

## 🔑 Secrets / 秘密情報

実際の秘密情報は GitHub に含めません。

除外対象例:

- Cloudflare API Token
- WireGuard Private Key
- WireGuard Preshared Key
- Xray UUID
- TLS Private Key
- wg-easy Database
- Ansible Vault

サンプル:

```text
inventory/group_vars/all/vault.example.yml
```

---

## 🛠️ Environment / 環境

```text
OS:
Rocky Linux 10

Platform:
Intel N100 Mini PC

Container:
Docker
Docker Compose

Automation:
Ansible
```

---

## 📚 What I learned / 学習内容

このプロジェクトでは、以下を実践しています。

- Linux server administration
- IPv4 / IPv6
- VPN
- Reverse proxy
- TLS
- DNS / Split DNS
- Dynamic DNS
- Docker
- Docker Compose
- Ansible
- Firewall
- Infrastructure as Code

---

## 📘 Documentation

詳細な構築内容は README にまとめています。

- [README.md](./README.md)
- [Ansible Playbooks](./playbooks/)
- [Templates](./templates/)

---

# 🇺🇸 English

## Overview

This project is a self-hosted VPN and proxy infrastructure environment
running on Rocky Linux 10.

It combines:

- WireGuard
- wg-easy
- Xray
- VLESS
- WebSocket
- TLS
- Nginx
- dnsmasq
- Split DNS
- Cloudflare DDNS
- Docker
- Ansible

The environment is designed to be reproducible and maintainable as Infrastructure as Code.

---

## Main Goals

- Secure remote access to the home network
- Full tunnel VPN access
- Separate WireGuard and Xray use cases
- Automated server deployment
- Easy rebuilding and recovery
- Practical Linux and networking learning

---

## Security

Sensitive information is intentionally excluded from this repository.

Examples include:

- API tokens
- Private keys
- Client UUIDs
- TLS private keys
- Databases containing credentials

---

## Repository

For technical details, see:

- [README.md](./README.md)
- [Playbooks](./playbooks/)
- [Templates](./templates/)

---

## ⚠️ Disclaimer

This project is primarily intended for personal infrastructure management
and technical learning.

Use at your own risk.