# Hosting applications
## Deploying a MariaDB instance
```shell
#!/usr/bin/env bash
DB_NAME="mydb"
DB_PASS="mypass123"
mkdir -p /var/db/$DB_NAME

docker run -d --name $DB_NAME -p 3902:3306 \
-v /var/db/$(DB_NAME):/var/lib/mysql --restart unless-stopped \
-e MARIADB_ROOT_PASSWORD=$DB_PASS mariadb:10.6.5 \
--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```
