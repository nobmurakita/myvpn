# WireGuard VPN Server on GCE

Docker Composeを使用したWireGuard VPNサーバー。DuckDNSで動的IPに対応。

## セットアップ

### 1. DuckDNSの準備

1. https://www.duckdns.org/ でアカウント作成
2. サブドメインを作成（例: `myvpn`）
3. トークンをコピー

### 2. GCEインスタンスの作成

1. [Compute Engine] > [VMインスタンス] を開く
2. [インスタンスを作成] をクリック
3. 以下を設定:
   - 名前: 任意（例: `wireguard`）
   - リージョン: `asia-northeast1`（東京）
   - マシンタイプ: `e2-micro`
   - ブートディスク: `Debian GNU/Linux 12`（デフォルト）
   - ネットワークタグ: `wireguard`
4. [作成] をクリック

### 3. ファイアウォール設定

1. [VPCネットワーク] > [ファイアウォール] を開く
2. [ファイアウォールルールを作成] をクリック
3. 以下を設定:
   - 名前: `allow-wireguard`
   - ターゲットタグ: `wireguard`
   - ソースIPの範囲: `0.0.0.0/0`
   - プロトコルとポート: `UDP` の `51820`
4. [作成] をクリック

### 4. Dockerのインストール

VMにSSH接続して以下を実行:

```bash
# Docker インストール
curl -fsSL https://get.docker.com | sudo sh

# 現在のユーザーをdockerグループに追加
sudo usermod -aG docker $USER

# 一度ログアウトして再接続
exit
```

### 5. myvpnディレクトリの配置

```bash
git clone <リポジトリURL> myvpn
cd myvpn
```

### 6. 環境変数の設定

```bash
cp .env.example .env
```

`.env` を編集（`PUID`/`PGID` は `id` コマンドで確認）:
```
DUCKDNS_SUBDOMAIN=myvpn
DUCKDNS_TOKEN=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
PEERS=client1
PUID=1000
PGID=1000
MTU=1392
```

### 7. 起動

```bash
docker compose up -d
```

### 8. クライアント設定の取得

**スマホ:** QRコードをWireGuardアプリでスキャン
```bash
docker exec wireguard /app/show-peer client1
```

**PC:** 設定ファイルをダウンロード
1. GCEコンソールのSSH接続画面右上の歯車アイコン → [ファイルをダウンロード]
2. パスを入力: `myvpn/config/peer_client1/peer_client1.conf`

## Peer管理

### 新しいデバイスを追加

`.env` の `PEERS` に端末名を追加してコンテナを再作成:

```bash
# .envを編集（例: client2を追加）
PEERS=client1,client2

docker compose up -d --force-recreate
```

新しいpeerの設定は `config/peer_<name>/` に生成される。

### デバイスを削除

```bash
# .envからPEERSの該当名を削除（例: client2 を削除）
PEERS=client1

# 該当peerのディレクトリを削除
rm -rf config/peer_client2

docker compose up -d --force-recreate
```

## ログ確認

```bash
# WireGuardログ
docker compose logs -f wireguard

# DuckDNSログ
docker compose logs -f duckdns
```

## 停止

```bash
docker compose down
```

## MTUについて

GCEのVPCネットワークはデフォルトMTUが1460のため、WireGuardのMTUは1392に設定している。

VPN有効時に速度が極端に遅い場合（特にダウンロード）はMTUの調整が必要。

調整方法：
1. クライアントの設定ファイルの`[Interface]`セクションに`MTU = 1280`を追加
2. 速度が改善したらMTU値を上げていき、速度が落ちる直前の値が最適値
3. `.env`の`MTU`を更新してコンテナを再作成

変更後は既存のクライアント設定も再取得する。
