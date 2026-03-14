# PostgreSQL Setup with Ansible

Ansibleを使用して、AlmaLinux 10環境におけるPostgreSQL 16 のインストール、外部接続許可、およびシステム共通設定管理用テーブルのセットアップ手順を自動化するプロジェクトです。

## クイックスタート (導入手順)

はじめてサーバーに導入する際は、まず `git` をインストールし、このプロジェクトをサーバーにダウンロード（クローン）する必要があります。

```bash
# 1. git のインストール (root権限で実行)
dnf install -y git

# 2. プロジェクトのダウンロードと移動
git clone https://github.com/kanata-aituki/ansible-postgres-server.git
cd ansible-postgres-server

# 3. セットアップの実行 (AnsibleのインストールからPlaybookの適用まで完了します)
chmod +x setup.sh
./setup.sh
```

## トラブルシューティング: ファイアウォール(firewalld)の設定について

`setup.sh` (Ansible) 実行後、`firewall-cmd --list-ports` を実行して `5432/tcp` が追加されていない場合は、Ansibleによるファイアウォール設定が正常に反映されていない可能性があります。

テスト環境にて、上記の問題が発生し接続拒否（Connection refused）となったため、以下のコマンドを手動で実行し、ポートを開放（接続を許可）した実績があります。

```bash
# 5432ポートの開放を手動で追加
firewall-cmd --permanent --add-port=5432/tcp

# ファイアウォール設定の再読み込み
firewall-cmd --reload

# ポートが正常に追加されたか確認
firewall-cmd --list-ports
```


## 設定方法

実行前に以下の設定を確認・変更してください。

### 1. 機密情報の設定 (パスワード)
`group_vars/secret.yml` にパスワードを設定します。(`setup.sh`実行時に自動でサンプルから作成されます)

- `db_password`: アプリ用データベースのパスワード

### 2. 環境設定 (IP等)
`group_vars/all.yml` を編集します。
- `db_name`: アプリ用データベース名
- `db_user`: アプリ用ユーザー名

## 自動化内容

1. パッケージのインストール (`postgresql-server`, `langpacks-ja`)
2. データベースの初期化と起動設定
3. ネットワーク・外部接続許可 (`listen_addresses = '*'`)
4. OS ファイアウォール (`firewalld`) の開放 (5432ポート)
5. 専用ユーザー・データベース作成
6. 認証・セキュリティ設定 (`pg_hba.conf` の更新)
7. 汎用共通テーブル（魔法の箱）の作成および初期データ投入

