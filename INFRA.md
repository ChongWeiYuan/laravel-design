テスト開発環境 構築手順書（ドラフト）
1. 本書の目的
本手順書は、シンVPS上にLaravel開発環境をゼロから構築する手順を記録したものである。本番環境（nagao.xyz）と同一の手順でテスト環境（develop.nagao.xyz）を再現するために使用する。

2. 対象環境
項目	内容
VPS	シンVPS 大容量メモリ 1GB
OS	Ubuntu 24.04 LTS
Webサーバー	Nginx 1.24
DB	MySQL 8.0
PHP	8.3（FPM）
アプリケーション	Laravel 11（13.x）
SSL	Let's Encrypt（Certbot）
3. 構築手順
Step 1: 一般ユーザーの作成とSSH設定
bash
# rootでログイン後、一般ユーザーを作成
adduser nagao
usermod -aG sudo nagao

# SSH公開鍵を登録（nagao用）
mkdir -p /home/nagao/.ssh
cp /root/.ssh/authorized_keys /home/nagao/.ssh/authorized_keys
chown -R nagao:nagao /home/nagao/.ssh
chmod 700 /home/nagao/.ssh
chmod 600 /home/nagao/.ssh/authorized_keys

# root SSHログイン禁止
sudo sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
Step 2: システム更新と基本設定
bash
sudo apt update && sudo apt upgrade -y
sudo timedatectl set-timezone Asia/Tokyo
sudo hostnamectl set-hostname nagao-dev
Step 3: ファイアウォール設定
bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
Step 4: Nginxインストール
bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
Step 5: MySQLインストール
bash
sudo apt install mysql-server -y
sudo mysql_secure_installation
Step 6: PHP 8.3インストール
bash
sudo apt install php8.3 php8.3-fpm php8.3-mysql php8.3-cli php8.3-common php8.3-mbstring php8.3-xml php8.3-curl php8.3-zip php8.3-bcmath php8.3-tokenizer php8.3-ctype php8.3-fileinfo -y
Step 7: Composerインストール
bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
sudo mv composer.phar /usr/local/bin/composer
Step 8: Laravelプロジェクト作成
bash
sudo chown nagao:nagao /var/www
cd /var/www
composer create-project laravel/laravel nagao-dev
Step 9: データベース作成
bash
sudo mysql -e "CREATE DATABASE IF NOT EXISTS nagao_dev CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
sudo mysql -e "CREATE USER IF NOT EXISTS 'nagao_dev'@'localhost' IDENTIFIED BY '【強力なパスワード】';"
sudo mysql -e "GRANT ALL PRIVILEGES ON nagao_dev.* TO 'nagao_dev'@'localhost';"
sudo mysql -e "FLUSH PRIVILEGES;"
Step 10: .env設定
bash
cd /var/www/nagao-dev
sed -i 's/^APP_ENV=.*/APP_ENV=local/' .env
sed -i 's/^APP_DEBUG=.*/APP_DEBUG=true/' .env
sed -i 's|^APP_URL=.*|APP_URL=https://develop.nagao.xyz|' .env
sed -i 's/^DB_CONNECTION=.*/DB_CONNECTION=mysql/' .env
sed -i 's/^DB_HOST=.*/DB_HOST=127.0.0.1/' .env
sed -i 's/^DB_PORT=.*/DB_PORT=3306/' .env
sed -i 's/^DB_DATABASE=.*/DB_DATABASE=nagao_dev/' .env
sed -i 's/^DB_USERNAME=.*/DB_USERNAME=nagao_dev/' .env
sed -i 's/^DB_PASSWORD=.*/DB_PASSWORD=【強力なパスワード】/' .env
php artisan key:generate
Step 11: Nginx設定
bash
sudo nano /etc/nginx/sites-available/nagao-dev
nginx
server {
    listen 80;
    server_name develop.nagao.xyz;
    root /var/www/nagao-dev/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    index index.php;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
bash
sudo ln -s /etc/nginx/sites-available/nagao-dev /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
Step 12: SSL証明書取得
bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d develop.nagao.xyz
Step 13: Git管理
bash
cd /var/www/nagao-dev
git init
git remote add origin https://github.com/ChongWeiYuan/laravel-inventory.git
git branch -M develop
git add .
git commit -m "Initial commit: development environment"
git push -u origin develop
4. 補足事項
項目	備考
SSH鍵	Windowsクライアントの場合、秘密鍵のパーミッションに注意（icaclsで修正）
メモリ不足対策	スワップ2GBがデフォルトで設定済み。npm buildはローカルDockerで行う
.env管理	.gitignore に .env を追加し、GitHubにプッシュしないこと
パケットフィルター	シンVPSのコントロールパネル側でもポート80/443を開放すること
