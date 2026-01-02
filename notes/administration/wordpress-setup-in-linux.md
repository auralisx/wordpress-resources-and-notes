This note tries to deploy WordPress in a way that mirrors modern best industry standard practices used by professionals which is secure, maintainable and performant without overcomplicating the setup. It focuses on simplicity and ease of use while maintaining best practices for personal local setup.

> [!NOTE]
> The config files generated during the setup process are stored in `config` directory for future reference.

# Industry standard architecture

| Layer | Component | Reasoning |
|-------|-----------|-----------|
| Web Server | Nginx | Fast, lightweight, excellent with static assets and reverse proxying. |
| Application | PHP-FPM | More secure and memory efficient than mod_php, allowing for better resource management and scalability. |
| Database | MariaDB | MariaDB is a community-developed fork of MySQL, offering improved performance, security, and scalability. MariaDB or MySQL is still standard. Other databases like PostgreSQL or SQLite can also be used depending on the specific requirements of the application. |
| Caching | OPcache (PHP) + Redis or Memcached | OPcache improves PHP performance by storing precompiled script bytecode in shared memory, reducing the overhead of parsing and compiling PHP code. Redis and Memcached are popular in-memory data stores that can be used for caching frequently accessed data, improving application performance. |
| Updates | WP-CLI | WP-CLI is a command-line interface for WordPress, allowing for easy management and automation of WordPress tasks. |

Apache is still an excellent choice for many users as it is widely available, easy to use, and has a large community of users and developers who can provide support and assistance. It is the default web server used by many shared hosting providers as nginx doesn't shine in shared environments. But for sites that expects a lot of traffic and speed matters, Nginx is a better choice. A lot of websites are migrating to Nginx. A more detailed explanation of the difference between Apache and Nginx can be found on [apache vs nginx vs litespeed](https://www.wpbeginner.com/opinion/apache-vs-nginx-vs-litespeed/) and [nginx-vs-apache](https://kinsta.com/blog/nginx-vs-apache/).

> [!NOTE]
> Hybrid setup (e.g., Nginx as reverse proxy in front of Apache, where nginx handles static content and apache handles dynamic content) are common in use but is not really needed for most cases thanks to a more mature softwares at present. Adding both adds complexity without significant benefit. You should only put apache behind nginx if you absolutely need it. The hybrid stack exists but it is not the best practice anymore. 
> PHP comes included with a FastCGI process manager called PHP-FPM which can be used directly with Nginx. The flow with PHP-FPM is much the same, but it is simpler to configure PHP-FPM than Apache + mod_php. PHP-FPM also feels lighter weight and seems to use fewer resources.

# Step-by-Step Setup (Arch Linux)

> [!TIP]
> For actual production environment, you should consider using a more stable linux distribution like debian, ubuntu or centos. Here, I am using Arch linux just because I have already installed it in my system and it's what I am most familiar with. And despite using a stable distribution, only security updates should be setup to be automatically applied. Not-security updates should be tested on staging server before intalling the updates. So, it doesn't matter that much what distribution you use in my opinion. I can disable updated for specific packages with pacman.conf, so Arch can as well be used as stable environment but requires a bit more effort.

## 1. Install packages
I am on Arch linux so I will use pacman to install the packages. The package manager or package name can differ in different distributions so it's advisable to do some research before installing packages.

> [!NOTE]
> I am following the installation and configuration guide available on Arch Wiki. You can view the wiki pages for more details > [PHP](https://wiki.archlinux.org/title/PHP), [WordPress](https://wiki.archlinux.org/title/WordPress), [MariaDB](https://wiki.archlinux.org/title/MariaDB) and [Nginx](https://wiki.archlinux.org/title/Nginx).
> Here I am just installing core packages. You can install additional packages or extensions as per your requirements. You can find more about the recommended server environment with php extensions list for WordPress in [Server Environment](https://make.wordpress.org/hosting/handbook/server-environment/).

```
sudo pacman -S nginx php php-fpm mariadb wp-cli
```
For caching, you can install redis or memcached. In arch, opcache comes bundled with standard php distribution so you only need to enable it in php.ini file. Between redis and memcached, you can install either as you prefer. There are great resources available online to help you choose the right caching solution for your needs. For me, I prefer Redis because it's more lightweight and easier to configure than Memcached while still being more flexible. Redis is also cost effective at growth and integrates smoothly with managed WordPress hosting while delivering better performance than Memcached.

```
sudo pacman -S redis php-redis
```

It is also recommended to install tools like ufw, fail2ban, and certbot for better security in the server environment.

## 2. Start Services
Here we can start the services we just installed. You can start them manually or configure them to start automatically on boot.

Before enabling mariadb.service, you need to run the following command:
```
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
```

Now, mariadb.service can be started or enabled. To start the services, you can use the following command:

```
sudo systemctl start nginx php-fpm mariadb redis
```

To make the services start automatically on boot, you can enable them using the following commands:

```
sudo systemctl enable --now nginx php-fpm mariadb redis
```

> [!NOTE]
> To know how this works, you can read the wiki for systemd here: [systemd](https://wiki.archlinux.org/title/Systemd). Almost all linux distributions use systemd as their init system. It provides a powerful and flexible way to manage services and processes on a Linux system. With systemd, you can easily start, stop, restart, and monitor services, as well as manage dependencies between services and system resources.

## 3. Configure [MariaDB](https://wiki.archlinux.org/title/MariaDB)

The mariadb-secure-installation command is used to secure the MariaDB installation by setting a root password, removing anonymous users, disabling remote root login, and removing the test database. It is recommended to run this command after installing MariaDB to improve the initial security of MariaDB installation.

```
sudo mariadb-secure-installation
```
The command will interactively guide you through a number of recommended security measures. Then, run the following command:

```
sudo mariadb -u root -p
```
You will be asked for your MariaDB root password. Then, create a new database and user for WordPress:

```
MariaDB> CREATE DATABASE wordpress;
MariaDB> GRANT ALL PRIVILEGES ON wordpress.* TO "wp-user"@"localhost" IDENTIFIED BY "choose_db_password";
MariaDB> FLUSH PRIVILEGES;
MariaDB> EXIT
```
> [!NOTE]
> wordpress is your Database Name and wp-user is your User Name. You can change them if you wish. Also replace choose_db_password with your new Password for this database. You will be asked for these values along with localhost in the next section.

## 4. Configure PHP

The main configuration file for PHP is located at `/etc/php/php.ini`. Open this file with your preferred text editor (I am going to use nvim) and make the preferred changes:

```
sudo nvim /etc/php/php.ini
```
I am going to make some recommended changes I have found useful:

```
date.timezone = "Asia/Kathmandu"

memory_limit = 256M
expose_php=off
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,parse_ini_file,show_source
session.use_strict_mode = 1
allow_url_fopen = Off

zend_extension=opcache
extension=pdo_mysql
extension=mysqli

[opcache]
opcache.enable=1
opcache.enable_cli=1 
opcache.enable_file_override=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
```

Let's enable redis extension as well:
```
sudo nvim /etc/php/conf.d/redis.ini
```
and uncomment the following line:

```
extension=redis
```
You can see in the comment that it requires igbinary to be activated as well. So, let's activate that as well.
```
sudo nvim /etc/php/conf.d/igbinary.ini
```
and uncomment the following line:

```
extension=igbinary.so
```

## 6. Configure [Nginx](https://wiki.archlinux.org/title/Nginx)
The configuration for nginx can be found at `/etc/nginx/nginx.conf`. For nginx configuration you can use configuration tools that have been provided by [Digital Ocean](https://www.digitalocean.com/community/tools/nginx/) and [Monzilla](https://ssl-config.mozilla.org/). 

For using the `sites-enabled` and `sites-available` approach, you can create the respective directories and files under sites-available that contains one or more server blocks;
```
sudo mkdir /etc/nginx/sites-available
sudo mkdir /etc/nginx/sites-enabled
```
To enable the site, simply create a symlink:
```
sudo ln -s /etc/nginx/sites-available/example.conf /etc/nginx/sites-enabled/example.conf
```
To disable the site, simply remove the symlink:
```
sudo unlink /etc/nginx/sites-enabled/example.conf
```

> [!TIP]
> Copy pasting configuration from generator tools is not recommended as it may contain unnecessary configurations that may cause issues. It is recommended to create your own configuration files from scratch. You can use the configuration tools provided by Digital Ocean and Mozilla to generate a basic configuration and then modify it according to your needs. If you are unable to start nginx, try 
```
sudo nginx -t
```
> `nginx -t` validates the configuration without starting nginx. If there are any errors, it will display them and exit with a non-zero status code. If there are no errors, it will display a success message and exit with a zero status code.


# References
Look into these references for even more detailed information. I went through these references and found them useful. My notes just summarize what I learned and I applied for my local setup.
1. [SpinupWP](https://spinupwp.com/install-wordpress-ubuntu/)
2. [Advanced Administration](https://developer.wordpress.org/advanced-administration/)
