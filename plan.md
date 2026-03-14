これまでお伝えした内容を、そのまま一つの README.md や setup_postgresql.md として保存できる完成形のMarkdownにまとめました。コピーして、.md 拡張子で保存してご利用ください。PostgreSQL 16 構築・運用インフラ構成書 (AlmaLinux 10)このドキュメントは、AlmaLinux 10 環境における PostgreSQL のインストール、外部接続許可、およびシステム共通設定管理用テーブル（通称：魔法の箱）のセットアップ手順を記録したものです。1. パッケージのインストールAlmaLinux 10 の標準リポジトリを使用し、最新の安定版を導入します。Bash# パッケージ情報の更新と検索
dnf update -y
dnf search postgresql

# サーバー本体、クライアント、および日本語言語パックのインストール
dnf -y install postgresql postgresql-server langpacks-ja
2. データベースの初期化と起動設定Bash# データベースクラスタの初期化
/usr/bin/postgresql-setup --initdb

# サービスの有効化（OS起動時自動開始）と即時起動
systemctl enable postgresql
systemctl start postgresql
3. ネットワーク・外部接続許可デフォルトではローカルホストからしか接続できないため、リスニング設定を変更します。3.1. PostgreSQL 本体の設定Bash# 全てのインターフェースで接続を待ち受ける設定に変更
vi /var/lib/pgsql/data/postgresql.conf

# [変更箇所]
# listen_addresses = 'localhost'
# ↓
# listen_addresses = '*'

# 設定反映のために再起動
systemctl restart postgresql
3.2. OS ファイアウォール (firewalld) の開放Bash# PostgreSQL標準ポート 5432/tcp を開放
firewall-cmd --permanent --add-port=5432/tcp
firewall-cmd --reload

# 開放状況の確認
firewall-cmd --list-ports
4. ユーザー・データベース作成アプリケーションが接続するための専用環境を構築します。SQL/* postgres ユーザーでログインして実行 */
sudo -u postgres psql

-- アプリ用ユーザーの作成
CREATE USER appuser WITH PASSWORD 'mypassword';

-- アプリ用データベースの作成
CREATE DATABASE appdb;

-- 権限付与
GRANT ALL PRIVILEGES ON DATABASE appdb TO appuser;
\q
5. 認証・セキュリティ設定 (pg_hba.conf)接続元 IP や認証方式を定義します。5.1. 一般ユーザーの接続許可Bashvi /var/lib/pgsql/data/pg_hba.conf

# 末尾に追記: 全てのネットワーク(0.0.0.0/0)からパスワード認証(scram-sha-256)を許可
host all all 0.0.0.0/0 scram-sha-256
5.2. 管理者ユーザー (postgres) の保護管理ユーザーには強力なパスワードを設定し、特定の保守用 IP からのみ接続を許可します。SQL/* パスワード設定 */
sudo -u postgres psql
ALTER USER postgres WITH PASSWORD 'StrongPassword123!';
\q
Bash# 特定IP（例: 103.5.140.180）のみに管理者ログインを制限
vi /var/lib/pgsql/data/pg_hba.conf

# [追記内容]
host all postgres 103.5.140.180/32 scram-sha-256
6. 汎用共通テーブル（魔法の箱）の作成アプリケーションの動作（メンテナンスモード等）を動的に制御するための Key-Value テーブルです。SQL/* appdb に接続して実行 */
\c appdb

-- テーブル作成
CREATE TABLE system_parameters (
    parameter_key   TEXT PRIMARY KEY,         -- 設定を特定するユニークなキー
    parameter_value TEXT NOT NULL,            -- 設定値（すべて文字列で保持）
    description     TEXT,                     -- 設定の意味・用途の説明
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP -- 自動更新日時
);

-- 自動更新トリガーの設定
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER trg_system_parameters_update
    BEFORE UPDATE ON system_parameters
    FOR EACH ROW EXECUTE FUNCTION update_timestamp();

-- 初期データ：メンテナンスモード（デフォルトはOFF）
INSERT INTO system_parameters (parameter_key, parameter_value, description)
VALUES ('is_maintenance_mode', 'false', 'システムメンテナンス中フラグ (true/false)');
7. 運用操作マニュアル（逆引き）操作内容実行SQLメンテナンスを開始するUPDATE system_parameters SET parameter_value = 'true' WHERE parameter_key = 'is_maintenance_mode';メンテナンスを終了するUPDATE system_parameters SET parameter_value = 'false' WHERE parameter_key = 'is_maintenance_mode';新しい設定値を追加するINSERT INTO system_parameters (parameter_key, parameter_value, description) VALUES ('key_name', 'value', '説明文');現在の設定を一覧確認するSELECT * FROM system_parameters;