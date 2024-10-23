# Migrate from non-docker Seafile deployment to docker

The recommended steps to migrate from non-docker deployment to docker deployment are:

1. Upgrade the version of the binary package to latest version, and ensure that the system is running normally. (If you running a very old version of Seafile, you can following the [FAQ item](https://cloud.seatable.io/dtable/external-links/7b976c85f504491cbe8e/?tid=0000&vid=0000&row-id=VYQI9DJfRmCv5NggcX4f0Q) to migrate to the latest version)
2. Close Seafile and native Nginx, Memcached
3. Create the directory needed for Seafile Docker image to run, and copy some files of the locally deployed Seafile to this directory
4. Download the docker-compose.yml file and configure Seafile Docker to use non-Docker version configuration information to connect to the old MySQL database and the old seafile-data directory.
5. Start Seafile Docker

The following document assumes that the deployment path of your non-Docker version of Seafile is /opt/seafile. If you use other paths, before running the command, be careful to modify the command path.

> Note, you can also refer to the Seafile backup and recovery documentation, deploy Seafile Docker on another machine, and then copy the old configuration information, database, and seafile-data to the new machine to complete the migration. The advantage of this is that even if an error occurs during the migration process, the existing system will not be destroyed.

## Migrate

### Stop Seafile, Nginx

Stop the locally deployed Seafile, Nginx, Memcache

```
systemctl stop nginx &&  systemctl disable nginx
systemctl stop memcached &&  systemctl disable memcached
./seafile.sh stop  && ./seahub.sh stop
```

### Prepare MySQL and the folders for Seafile docker

#### Add permissions to the local MySQL Seafile user

The non-Docker version uses the local MySQL. Now if the Docker version of Seafile connects to this MySQL, you need to increase the corresponding access permissions.

The following commands are based on that you use `seafile` as the user to access:

```
## Note, change the password according to the actual password you use
GRANT ALL PRIVILEGES ON *.* TO 'seafile'@'%' IDENTIFIED BY 'your-password' WITH GRANT OPTION;

## Grant seafile user can connect the database from any IP address
GRANT ALL PRIVILEGES ON `ccnet_db`.* to 'seafile'@'%';
GRANT ALL PRIVILEGES ON `seafile_db`.* to 'seafile'@'%';
GRANT ALL PRIVILEGES ON `seahub_db`.* to 'seafile'@'%';

## Restart MySQL
systemctl restart mariadb
```

#### Create the required directories for Seafile Docker image

```
mkdir -p /opt/seafile-data/seafile
```

#### Prepare config files

Copy the original config files to the directory to be mapped by the docker version of seafile

```
cp -r /opt/seafile/conf  /opt/seafile-data/seafile
cp -r /opt/seafile/seahub-data  /opt/seafile-data/seafile
```

Modify the MySQL configuration in `/opt/seafile-data/seafile/conf`, including `ccnet.conf`, `seafile.conf`, `seahub_settings`, change `HOST=127.0.0.1` to `HOST=<local ip>`.

Modify the memcached configuration in `seahub_settings.py` to use the Docker version of Memcached: change it to `'LOCATION': 'memcached:11211'` (the network name of Docker version of Memcached is `memcached`).

#### Download and modify docker-compose.yml

Download [docker-compose.yml](https://manual.seafile.com/docker/pro-edition/10.0/docker-compose.yml) to `/opt/seafile-data`. Comment out the db part as below:

```
services:
#  db:
#    image: mariadb:10.5
#    container_name: seafile-mysql
#    environment:
#      - MYSQL_ROOT_PASSWORD=db_dev  # Required, set the root's password of MySQL service.
#      - MYSQL_LOG_CONSOLE=true
#    volumes:
#      - /opt/seafile-mysql/db:/var/lib/mysql  # Required, specifies the path to MySQL data persistent store.
#    networks:
#      - seafile-net

.........
   depends_on:
#     - db             
      - memcached
.........
```

### Configure Seafile Docker to use the old seafile-data

There are two ways to let Seafile Docker to use the old seafile-data

#### Method 1

You can copy or move the old seafile-data folder (`/opt/seafile/seafile-data`) to `/opt/seafile-data/seafile` (So you will have `/opt/seafile-data/seafile/seafile-data`)

#### Method 2

You can mount the old seafile-data folder (`/opt/seafile/seafile-data`) to Seafile docker container directly:

```
.........

    seafile:
    image: seafileltd/seafile-mc:8.0.7-1
    container_name: seafile
    ports:
      - "80:80"
#      - "443:443"  # If https is enabled, cancel the comment.
    volumes:
      - /opt/seafile-data:/shared
      - /opt/seafile/seafile-data:/shared/seafile/seafile-data
  .......
```

The added line `- /opt/seafile/seafile-data:/shared/seafile/seafile-data` mount `/opt/seafile/seafile-data` to `/shared/seafile/seafile-data` in docker.

### Start Seafile docker

Start Seafile docker and check if everything is okay:

```
cd /opt/seafile-data
docker compose  up -d
```

## Security
While it is not possible from inside a docker container to connect to the host database via localhost but via `<local ip>` you also need to bind your databaseserver to that IP. If this IP is public, it is strongly advised to protect your database port with a firewall. Otherwise your databases are reachable via internet.
An alternative might be to start another local IP from [RFC 1597](https://tools.ietf.org/html/rfc1597) e.g. `192.168.123.45`. Afterwards you can bind to that IP.

### iptables
Following iptables commands protect MariaDB/MySQL:
```
iptables -A INPUT -s 172.16.0.0/12 -j ACCEPT #Allow Dockernetworks
iptables -A INPUT -p tcp -m tcp --dport 3306 -j DROP #Deny Internet
ip6tables -A INPUT -p tcp -m tcp --dport 3306 -j DROP #Deny Internet
```
Keep in mind this is not bootsafe!

### Binding based
For Debian based Linux Distros you can start a local IP by adding in `/etc/network/interfaces` something like:
```
iface eth0 inet static
  address 192.168.123.45/32
```
`eth0` might be `ensXY`. Or if you know how to start a dummy interface, thats even better.

SUSE based is by editing `/etc/sysconfig/network/ifcfg-eth0` (ethXY/ensXY/bondXY)

If using MariaDB the server just can bind to one IP-address (192.158.1.38 or 0.0.0.0 (internet)). So if you bind your MariaDB server to that new address other applications might need some reconfigurement.

In `/etc/mysql/mariadb.conf.d/50-server.cnf` edit the following line to:
```
bind-address    = 192.168.123.45
```
then edit /opt/seafile-data/seafile/conf/ -> ccnet.conf seafile.conf seahub_settings.py in the Host-Line to that IP and execute the following commands:

```
service networking reload
ip a #to check whether the ip is present
service mysql restart
ss -tulpen | grep 3306 #to check whether the database listens on the correct IP
cd /opt/seafile-data/
docker compose down
docker compose up -d

## restart your applications
```
