This note tries to deploy WordPress in a way that mirrors modern best industry standard practices used by professionals which is secure, maintainable and performant without overcomplicating the setup. It focuses on simplicity and ease of use while maintaining best practices.

# Industry standard architecture

Apache is still an excellent choice for many users as it is widely available, easy to use, and has a large community of users and developers who can provide support and assistance. It is the default web server used by many shared hosting providers as nginx doesn't shine in shared environments. But for sites that expects a lot of traffic and speed matters, Nginx is a better choice. A lot of websites are migrating to Nginx. A more detailed explanation of the difference between Apache and Nginx can be found on [apache vs nginx vs litespeed](https://www.wpbeginner.com/opinion/apache-vs-nginx-vs-litespeed/) and [nginx-vs-apache](https://kinsta.com/blog/nginx-vs-apache/).

> [!NOTE]
> Hybrid setup (e.g., Nginx as reverse proxy in front of Apache, where nginx handles static content and apache handles dynamic content) are common in use but is not really needed for most cases thanks to a more mature softwares at present. Adding both adds complexity without significant benefit. You should only put apache behind nginx if you absolutely need it. The hybrid stack exists but it is not the best practice anymore.

| Layer | Component | Reasoning |
|-------|-----------|-----------|
| Web Server | Nginx | Fast, lightweight, excellent with static assets and reverse proxying. |
| Application | PHP-FPM | More secure and memory efficient than mod_php, allowing for better resource management and scalability. |
| Database | MariaDB | MariaDB is a community-developed fork of MySQL, offering improved performance, security, and scalability. MariaDB or MySQL is still standard. Other databases like PostgreSQL or SQLite can also be used depending on the specific requirements of the application. |
| Caching | OPcache (PHP) + Redis or Memcached | OPcache improves PHP performance by storing precompiled script bytecode in shared memory, reducing the overhead of parsing and compiling PHP code. Redis and Memcached are popular in-memory data stores that can be used for caching frequently accessed data, improving application performance. |
| Updates | WP-CLI | WP-CLI is a command-line interface for WordPress, allowing for easy management and automation of WordPress tasks. |


# Step-by-Step Setup (Arch Linux)

## 1. Install packages
I am on Arch linux so I will use pacman to install the packages. The package manager or package name can differ in different distributions so it's advisable to do some research before installing packages.

> [!NOTE]
> I am following the installation and configuration guide available on Arch Wiki. You can view the wiki pages for more details > [PHP](https://wiki.archlinux.org/title/PHP), [WordPress](https://wiki.archlinux.org/title/WordPress), [MariaDB](https://wiki.archlinux.org/title/MariaDB) and [Nginx](https://wiki.archlinux.org/title/Nginx).
> Here I am just installing core packages. You can install additional packages or extensions as per your requirements. You can find more about the recommended server environment with php extensions list for WordPress in [Server Environment](https://make.wordpress.org/hosting/handbook/server-environment/).

```
sudo pacman -S nginx php-fpm mariadb wp-cli
```
For caching, you can install redis or memcached. In arch, opcache comes bundled with standard php distribution so you only need to enable it in php.ini file. Between redit and memcached, you can install either as you prefer. There are great resources available online to help you choose the right caching solution for your needs. For me, I prefer Redis because it's more lightweight and easier to configure than Memcached while still being more flexible. Redis is also cost effective at growth and integrates smoothly with managed WordPress hosting while delivering better performance then Memcached. Here's how to install Redis:

```
sudo pacman -S redis php-redis
```

## 2. Start Services
Here we can start the services we just installed. You can start them manually or configure them to start automatically on boot.

To start the services, you can use the following command:

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
