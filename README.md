# nextcloud-traefik
Personal implementation of nextcloud &amp; traefik

I posted this implementation both as a safeguard and so that it might help someone, someday.

TAGs to be replaced in docker-compose.yml:
 - DOMAINNAME : Domain name `example.com`
 - EMAILADDR : Used for letsencrypt service
 - TRAEFIK_HTPASS : htpasswd for traefik.DOMAINNAME `echo $(htpasswd -nb user password) | sed -e s/\\$/\\$\\$/g`
 - NCADMIN: Aminitrator login for nextcloud
 - NCPASSWD: Aministrator's password
 - PREFIX: Traefix prefix for containers, volumes, etc.

Tested on Gentoo with
 - Docker 19.03.5, build 633a0ea
 - Nextcloud latest (18.0.2)

Usage:
```bash
export DOMAINNAME=example.com
export EMAILADDR=admin@example.com
export TRAEFIK_HTPASS=$(echo $(htpasswd -nb user password) | sed -e s/\\$/\\$\\$/g)
export NCADMIN=admin
export NCPASSWD=password
export PREFIX=test

sed -e "s#DOMAINNAME#${DOMAINNAME}#g" -e "s#EMAILADDR#${EMAILADDR}#g" -e "s#TRAEFIK_HTPASS#${TRAEFIK_HTPASS}#g" docker-compose.yml.default > docker-compose.yml

sed -e "s#PASSWORD1#$(xkcdpass -d '-' -C alternating)#g" -e "s#PASSWORD2#$(xkcdpass -d '-' -C alternating)#g" docker.env.default > docker.env
sed -i -e "s#NCADMIN#${NCADMIN}#g" -e "s#NCPASSWD#${NCPASSWD}#g" docker.env
sed -i -e "s#DOMAINNAME#${DOMAINNAME}#g" docker.env

docker-compose -p $PREFIX up
```

Wait for nextcloud to finish initialization. Once you can get to the login page, do:
```bash
docker exec --user www-data ${PREFIX}_nextcloud_1 php occ maintenance:mode --on
docker exec --user www-data ${PREFIX}_nextcloud_1 php occ config:system:set trusted_domains 1 --value="${DOMAINNAME}"
docker exec --user www-data ${PREFIX}_nextcloud_1 php occ config:system:set overwrite.cli.url --value="https://${DOMAINNAME}"
docker exec --user www-data ${PREFIX}_nextcloud_1 php occ config:system:set overwriteprotocol --value="https"
docker exec --user www-data ${PREFIX}_nextcloud_1 php occ config:system:set forwarded-for-headers 0 --value="HTTP_X_FORWARDED_FOR"
docker exec --user www-data ${PREFIX}_nextcloud_1 php occ db:add-missing-indices
docker exec --user www-data ${PREFIX}_nextcloud_1 php occ db:convert-filecache-bigint
```

Find the address for your trusted proxie, for me this worked:
```
export TRUSTEDIP=$(docker container inspect ${PREFIX}_traefik_1 | awk '{$1=$1};1' | grep '^"IPAddress' | tail -n1 | awk -F'"' '{print $4}')

docker cp ${PREFIX}_nextcloud_1:/var/www/html/config/config.php .
sed -i "s#);#  'trusted_proxies' =>\n  array (\n    0 => '${TRUSTEDIP}',\n    1 => '127.0.0.1',\n  ),\n);#g" config.php
docker cp config.php ${PREFIX}_nextcloud_1:/var/www/html/config/config.php
docker exec ${PREFIX}_nextcloud_1 chown www-data:root /var/www/html/config/config.php
docker exec --user www-data ${PREFIX}_nextcloud_1 php occ maintenance:mode --off
```
