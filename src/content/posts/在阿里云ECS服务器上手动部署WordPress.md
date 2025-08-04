---
title: 在阿里云ECS服务器上手动部署WordPress[TEST]
published: 2025-08-04
description: ''
image: './334b37a8c6b89a6b9c5d2544c445d34d.jpg'
tags: [guide]
category: 'ECS'
draft: false 
lang: ''
---

# !!!本指南仅为理论指导 并未进行过任何实际测试

本指南将详细讲解如何在阿里云ECS服务器上手动部署WordPress，不使用宝塔面板等运维软件。适合有一定Linux基础的用户。

## 目录

1. 准备工作

2. 服务器环境配置

3. 安装Apache Web服务器

4. 安装MySQL数据库

5. 安装PHP环境

6. 配置WordPress

7. 配置Apache虚拟主机

8. 完成安装

9. 安全加固建议

10. 常见问题

---

## 1. 准备工作

### 1.1 阿里云ECS准备

1. 购买阿里云ECS实例（推荐Ubuntu 20.04/22.04或CentOS 7/8）

2. 配置安全组规则：
   
   - 开放80端口 (HTTP)
   
   - 开放443端口 (HTTPS)
   
   - 开放22端口 (SSH)

3. 获取服务器公网IP地址

### 1.2 工具准备

- SSH客户端 (如PuTTY、Xshell或终端)

- SFTP客户端 (如FileZilla)

- 本地文本编辑器

### 1.3 域名准备（可选）

- 已注册域名

- 设置DNS解析到服务器IP

---

## 2. 服务器环境配置

### 2.1 连接到服务器

```bash
ssh root@your_server_ip
```

### 2.2 更新系统

# Ubuntu/Debian

```bash
sudo apt update && sudo apt upgrade -y
```

# CentOS/RHEL

```bash
sudo yum update -y
```

### 2.3 创建新用户（可选但推荐）

```bash
adduser yourusername
usermod -aG sudo yourusername
```

---

## 3. 安装Apache Web服务器

### 3.1 安装Apache

# Ubuntu/Debian

```bash
sudo apt install apache2 -y
```

# CentOS/RHEL

```bash
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
```

### 3.2 验证安装

浏览器访问：`http://your_server_ip`  
应看到Apache默认页面

### 3.3 配置防火墙

# Ubuntu/Debian (UFW)

```bash
sudo ufw allow 'Apache Full'
```

# CentOS/RHEL (FirewallD)

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

---

## 4. 安装MySQL数据库

### 4.1 安装MySQL

# Ubuntu/Debian

```bash
sudo apt install mysql-server -y
```

# CentOS/RHEL

```bash
sudo yum install mysql-server -y
sudo systemctl start mysqld
sudo systemctl enable mysqld
```

### 4.2 安全配置

按提示操作：

1. 设置root密码

2. 移除匿名用户

3. 禁止远程root登录

4. 移除测试数据库

5. 刷新权限表

### 4.3 创建WordPress数据库

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE wordpressdb;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON wordpressdb.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```



---

## 5. 安装PHP环境

### 5.1 安装PHP及扩展

# Ubuntu 22.04

```bash
sudo apt install php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y 
```

# CentOS 7

```bash
sudo yum install epel-release -y
sudo yum install php php-mysqlnd php-gd php-mbstring php-xml php-soap php-zip -y
```

### 5.2 配置PHP

```bash
sudo nano /etc/php/8.1/apache2/php.ini # Ubuntu 22.04路径示例
```

修改以下参数：

```ini
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
```



### 5.3 重启Apache

# Ubuntu/Debian

```bash
sudo systemctl restart apache2
```

# CentOS/RHEL

```bash
sudo systemctl restart httpd
```

____

# 6.配置WordPress

### 6.1 下载WordPress

```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gzcd /tmp
```

### 6.2 移动文件到网站目录

```bash
sudo mv wordpress /var/www/html/yourdomain.com
sudo chown -R www-data:www-data /var/www/html/yourdomain.com # Ubuntu
sudo chown -R apache:apache /var/www/html/yourdomain.com # CentOS
```

### 6.3 配置wp-config.php

```bash
cd /var/www/html/yourdomain.com
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

修改以下配置：

```php
define('DB_NAME', 'wordpressdb');
define('DB_USER', 'wpuser');
define('DB_PASSWORD', 'strong_password');
define('DB_HOST', 'localhost');

// 添加以下安全密钥（从https://api.wordpress.org/secret-key/1.1/salt/获取）
define('AUTH_KEY', 'put your unique phrase here');
define('SECURE_AUTH_KEY', 'put your unique phrase here');
define('LOGGED_IN_KEY', 'put your unique phrase here');
define('NONCE_KEY', 'put your unique phrase here');
define('AUTH_SALT', 'put your unique phrase here');
define('SECURE_AUTH_SALT', 'put your unique phrase here');
define('LOGGED_IN_SALT', 'put your unique phrase here');
define('NONCE_SALT', 'put your unique phrase here');
```

### 6.4 设置文件权限

```bash
sudo find /var/www/html/yourdomain.com/ -type d -exec chmod 750 {} \;
sudo find /var/www/html/yourdomain.com/ -type f -exec chmod 640 {} \;
sudo chmod 600 /var/www/html/yourdomain.com/wp-config.php
```

---

## 7. 配置Apache虚拟主机

### 7.1 创建虚拟主机文件

# Ubuntu/Debian

```bash
sudo nano /etc/apache2/sites-available/yourdomain.com.conf
```

# CentOS/RHEL

```bash
 sudo nano /etc/httpd/conf.d/yourdomain.com.conf
```

### 7.2 虚拟主机配置

```apacheconf
<VirtualHost *:80>
 ServerAdmin admin@yourdomain.com
 ServerName yourdomain.com
 ServerAlias www.yourdomain.com
 DocumentRoot /var/www/html/yourdomain.com

<Directory /var/www/html/yourdomain.com>
    Options FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>

ErrorLog ${APACHE_LOG_DIR}/yourdomain.com_error.log
CustomLog ${APACHE_LOG_DIR}/yourdomain.com_access.log combined
```

</VirtualHost>

### 7.3 启用配置

# Ubuntu/Debian

```bash
sudo a2ensite yourdomain.com.conf
sudo a2enmod rewrite
sudo systemctl reload apache2
```

# CentOS/RHEL

```bash
sudo systemctl restart httpd
```

---

## 8. 完成安装

1. 在浏览器访问：`http://yourdomain.com` 或 `http://your_server_ip`

2. 按照WordPress安装向导完成：
   
   - 选择语言
   
   - 填写站点信息（标题、用户名、密码、邮箱）

3. 登录WordPress后台：`http://yourdomain.com/wp-admin`

---

## 9. 安全加固建议

1. **启用HTTPS**
   
   ```bash
   sudo apt install certbot python3-certbot-apache -y # Ubuntu
   sudo certbot --apache -d yourdomain.com -d www.yourdomain.com
   ```

2. **限制文件权限**
   
   ```bash
    sudo chown -R www-data:www-data /var/www/html/yourdomain.com # Ubuntu
    sudo chown -R apache:apache /var/www/html/yourdomain.com # CentOS
   ```

3. **禁用目录浏览**
   
   在Apache配置中添加：
   
   ```apacheconf
   Options -Indexes
   ```

4. **定期更新**
   
   # Ubuntu
   
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
   
   # CentOS
   
   ```bash
   sudo yum update -y
   ```

5. **安装安全插件**
   
   - Wordfence
   
   - iThemes Security

---

## 10. 常见问题

### Q1: 访问网站出现403 Forbidden错误

# 检查目录权限

```bash
sudo chown -R www-data:www-data /var/www/html/yourdomain.com
sudo chmod -R 755 /var/www/html/yourdomain.com
```

### Q2: WordPress无法写入文件

# 临时解决方案（生产环境不推荐）

```bash
sudo chmod -R 775 /var/www/html/yourdomain.com/wp-content
```

### Q3: 数据库连接错误

- 检查wp-config.php中的数据库凭据

- 确认MySQL服务正在运行：`sudo systemctl status mysql`

### Q4: 页面样式丢失

- 检查.htaccess文件是否存在

- 确保Apache的mod_rewrite已启用

### Q5: 内存不足错误

编辑wp-config.php，添加：

    define('WP_MEMORY_LIMIT', '256M');

---

**完成部署！** 您现在拥有一个手动部署的WordPress网站，无需依赖宝塔面板等运维软件。定期备份您的网站和数据库以确保数据安全。
