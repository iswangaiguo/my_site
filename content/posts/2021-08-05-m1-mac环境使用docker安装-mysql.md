---
title: M1 Mac环境使用Docker安装 MySQL
date: 2021-08-05T12:02:36+00:00
categories:
  - Docker
  - 编程
tags:
  - docker
  - mysql
---

## MySQL镜像

在dockerhub搜索MySQL镜像可以发现docker官方并没有提供arm版本的MySQL镜像

![docker官方mysql镜像](https://objects.wangaiguo.com/wp-content/uploads/2021/08/image.png)

不过还好MySQL官方提供了arm版本镜像

![arm版本mysql镜像](https://objects.wangaiguo.com/wp-content/uploads/2021/08/image-1.png)

## 安装MySQL

打开终端直接运行以下命令：

```bash
docker run --name mysql -e MYSQL_ROOT_PASSWORD=123456 -dp 3306:3306 mysql/mysql-server --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --skip-character-set-client-handshake
```

这里的`--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci`用来设置服务端字符编码，
`--skip-character-set-client-handshake`用来使客户端编码和服务端保持一致。

相当于MySQL配置文件中的：

```ini
[client]
default-character-set=utf8mb4

[mysql]
default-character-set=utf8mb4

[mysqld]
collation-server = utf8mb4_unicode_ci
character-set-server = utf8mb4
```

接下来进入mysql查看下编码情况：

```bash
docker exec -it mysql /bin/bash -c "mysql -uroot -p123456"
```

```sql
mysql> show variables like 'char%';
+--------------------------+--------------------------------+
| Variable_name            | Value                          |
+--------------------------+--------------------------------+
| character_set_client     | utf8mb4                        |
| character_set_connection | utf8mb4                        |
| character_set_database   | utf8mb4                        |
| character_set_filesystem | binary                         |
| character_set_results    | utf8mb4                        |
| character_set_server     | utf8mb4                        |
| character_set_system     | utf8mb3                        |
| character_sets_dir       | /usr/share/mysql-8.0/charsets/ |
+--------------------------+--------------------------------+
8 rows in set (0.01 sec)
```

可以看到内部的字符集编码已经全部变成了utf8mb4（在mysql中utf8仅支持3个字节的字符编码，最大能编码的 Unicode 字符是 0xffff，也就是 Unicode 中的基本多文种平面(BMP)，无法表达辅助平面字符，utf8mb4作为后来补充的字符集，能够完美的支持4字节的字符编码，也就是真正的utf8编码）
