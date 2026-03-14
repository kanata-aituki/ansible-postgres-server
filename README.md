# PostgreSQL Setup with Ansible

Ansibleを使用して、AlmaLinux 10環境におけるPostgreSQL 16 のインストール、外部接続許可、およびシステム共通設定管理用テーブルのセットアップ手順を自動化するプロジェクトです。

## クイックスタート

以下のスクリプトを実行するだけで、AnsibleのインストールからPlaybookの適用まで完了します。

```bash
chmod +x setup.sh
./setup.sh
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
