# nextcloud-traefik
Personal implementation of nextcloud &amp; traefik

I posted this implementation both as a safeguard and so that it might help someone, someday.

## Required environement files
### docker-compose
You need to configure a `{environment}.env` file so that `docker-compose.yml` works correctly.

|     Environment variable    | example                          |
| --------------------------: | :------------------------------- |
|                `DOMAINNAME` | myserver.com                     |
|                 `EMAILADDR` | emailfotletsencrypt@myserver.com |
|            `TRAEFIK_HTPASS` | asecret                          |
|                `REDIS_PASS` | asecret                          |

- For `TRAEFIK_HTPASS`, use something like: `$(htpasswd -nb user password) | sed -e s/\\$/\\$\\$/g` to get the appropiate value.
- For `REDIS_PASS`, use something like: `$(xkcdpass -d '-' -C alternating)`

### nextcloud &amp; database
We need a environment variable file for Nextcloud and our database. File name needs to be `docker.{environment}.env`.

|     Environment variable    | example      |
| --------------------------: | :----------- |
|                `MYSQL_HOST` | mariadb      |
|            `MYSQL_PASSWORD` | see_below    |
|       `MYSQL_ROOT_PASSWORD` | see_below    |
|            `MYSQL_DATABASE` | nextcloud    |
|                `MYSQL_USER` | nextcloud    |
|      `NEXTCLOUD_ADMIN_USER` | myUserName   |
|  `NEXTCLOUD_ADMIN_PASSWORD` | changeMe     |
| `NEXTCLOUD_TRUSTED_DOMAINS` | myserver.com |
|                `REDIS_HOST` | redis        |
|       `REDIS_HOST_PASSWORD` | asecret      |

- For `MYSQL_PASSWORD`, use something like: `$(xkcdpass -d '-' -C alternating)`
- For `MYSQL_ROOT_PASSWORD`, use something like: `$(xkcdpass -d '-' -C alternating)`
- For `NEXTCLOUD_ADMIN_PASSWORD`, this is your first login password, you will be able to change it and this one will not be used anymore.
- For `REDIS_HOST_PASSWORD`, this is your first login password, you will be able to change it and this one will not be used anymore.

### nextcloud-exporter
We will need a file named `nextcloud_exporter.{environment}.env` in order to export Nextcloud's metric.
|     Environment variable    | example      |
| --------------------------: | :----------- |
|          `NEXTCLOUD_SERVER` | myserver.com |
|        `NEXTCLOUD_USERNAME` | see_below    |
|        `NEXTCLOUD_PASSWORD` | see_below    |

See https://github.com/xperimental/nextcloud-exporter/blob/master/README.md to get info on own to generate `NEXTCLOUD_USERNAME` and `NEXTCLOUD_PASSWORD`.

Tested on Gentoo with
 - Docker 19.03.15, build 420b1d3625
 - Nextcloud latest (20.0)
 - xkcdpass 1.17.6

Usage:
```bash

export ENVIRONMENT=test

echo "DOMAINNAME=example.com" > ${ENVIRONMENT}.env
echo "EMAILADDR=admin@example.com" >> ${ENVIRONMENT}.env
echo "TRAEFIK_HTPASS=$(echo $(htpasswd -nb user password) | sed -e s/\\$/\\$\\$/g)" >> ${ENVIRONMENT}.env
echo "REDIS_PASS=$(xkcdpass -d '-' -C alternating)" >> ${ENVIRONMENT}.env

echo "MYSQL_HOST=mariadb" > docker.${ENVIRONMENT}.env
echo "MYSQL_PASSWORD=$(xkcdpass -d '-' -C alternating)" >> docker.${ENVIRONMENT}.env
echo "MYSQL_ROOT_PASSWORD=$(xkcdpass -d '-' -C alternating)" >> docker.${ENVIRONMENT}.env
echo "MYSQL_DATABASE=nextcloud" >> docker.${ENVIRONMENT}.env
echo "MYSQL_USER=nextcloud" >> docker.${ENVIRONMENT}.env
echo "NEXTCLOUD_ADMIN_USER=myUserName" >> docker.${ENVIRONMENT}.env
echo "NEXTCLOUD_ADMIN_PASSWORD=changeMe" >> docker.${ENVIRONMENT}.env
echo "NEXTCLOUD_TRUSTED_DOMAINS=$(grep DOMAINNAME ${ENVIRONMENT}.env | awk -F= '{print $2}')" >> docker.${ENVIRONMENT}.env
echo "REDIS_HOST=redis" >> docker.${ENVIRONMENT}.env
echo "REDIS_HOST_PASSWORD=$(grep REDIS_PASS ${ENVIRONMENT}.env | awk -F= '{print $2}')" >> docker.${ENVIRONMENT}.env


docker-compose -p ${ENVIRONMENT} --env-file ${ENVIRONMENT}.env up
```

Wait for nextcloud to finish initialization. Log will show something like
```
nextcloud_1           | Nextcloud was successfully installed
nextcloud_1           | setting trusted domains…
```

Then do:
```bash
docker exec --user www-data ${ENVIRONMENT}_nextcloud_1 php occ maintenance:mode --on
docker exec --user www-data ${ENVIRONMENT}_nextcloud_1 php occ config:system:set overwrite.cli.url --value="https://$(grep DOMAINNAME ${ENVIRONMENT}.env | awk -F= '{print $2}')"
docker exec --user www-data ${ENVIRONMENT}_nextcloud_1 php occ config:system:set overwriteprotocol --value="https"
docker exec --user www-data ${ENVIRONMENT}_nextcloud_1 php occ config:system:set forwarded-for-headers 0 --value="HTTP_X_FORWARDED_FOR"
docker exec --user www-data ${ENVIRONMENT}_nextcloud_1 php occ db:add-missing-indices
docker exec --user www-data ${ENVIRONMENT}_nextcloud_1 php occ db:convert-filecache-bigint
```

Find the address for your trusted proxie, for me this worked:
```bash
export TRUSTEDIP=$(docker container inspect ${ENVIRONMENT}_traefik_1 | awk '{$1=$1};1' | grep '^"IPAddress' | tail -n1 | awk -F'"' '{print $4}')

docker cp ${ENVIRONMENT}_nextcloud_1:/var/www/html/config/config.php .
sed -i "s#);#  'trusted_proxies' =>\n  array (\n    0 => '${TRUSTEDIP}',\n    1 => '127.0.0.1',\n  ),\n);#g" config.php
docker cp config.php ${ENVIRONMENT}_nextcloud_1:/var/www/html/config/config.php
docker exec ${ENVIRONMENT}_nextcloud_1 chown www-data:root /var/www/html/config/config.php
docker exec --user www-data ${ENVIRONMENT}_nextcloud_1 php occ maintenance:mode --off
```

Log into your acount and then get your login password for Nextcloud-exporter
```bash
docker run --rm -it xperimental/nextcloud-exporter --login --server https://$(grep DOMAINNAME ${ENVIRONMENT}.env | awk -F= '{print $2}')

echo "NEXTCLOUD_SERVER=$(grep DOMAINNAME ${ENVIRONMENT}.env | awk -F= '{print $2}')" > nextcloud_exporter.${ENVIRONMENT}.env
echo "NEXTCLOUD_USERNAME=username" >> nextcloud_exporter.${ENVIRONMENT}.env
echo "NEXTCLOUD_PASSWORD=password" >> nextcloud_exporter.${ENVIRONMENT}.env
```

At this point, you'll need to restart
```bash
docker-compose -p ${ENVIRONMENT} --env-file ${ENVIRONMENT}.env restart
```