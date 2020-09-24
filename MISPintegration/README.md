# MISP Integration With The Hive4 #

**Important Note: Please consider building the docker image by navigating to DockerImage directory and running the following command ```docker build . -t thehivemisp```**

Using docker compose we will easily configure the integration of MISP with The Hive4, we will use the following docker images of [MISP](https://github.com/coolacid/docker-misp) maintained by Jason Kendall (Coolacid), so please don't forget to check it to customize your MISP instance because we will use the default configuration.

**NOTE:** It's recommended to use SSL for MISP in production, we will be demonstrating on port 80 and using http protocol for testing purposes.  

## Docker-compose ##

First of all, make sure to visit the github repository of coolacid **[docker-misp](https://github.com/coolacid/docker-misp)** and clone it, then replace the docker-compose.yml file with the following:

```yaml
version: "3.8"
services:
  # This is capible to relay via gmail, Amazon SES, or generic relays
  # See: https://hub.docker.com/r/namshi/smtp
  mail:
    image: namshi/smtp

  redis:
    image: redis:5.0.6

  db:
    image: mysql:8.0.19
    command: --default-authentication-plugin=mysql_native_password
    container_name: MySQL-DB
    restart: unless-stopped
    environment:
      - "MYSQL_USER=misp" 
      - "MYSQL_PASSWORD=example"
      - "MYSQL_ROOT_PASSWORD=password"
      - "MYSQL_DATABASE=misp"
#    volumes:
#      - mysql_data:/var/lib/mysql  #Mysql DB data persistence

  misp:
    image: coolacid/misp-docker:core-latest
    container_name: MISP
    restart: unless-stopped
    depends_on:
      - redis
      - db
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "./server-configs/:/var/www/MISP/app/Config/"
      - "./logs/:/var/www/MISP/app/tmp/logs/"
      - "./files/:/var/www/MISP/app/files"
      - "./server/files/etc/nginx/misp80-noredir:/etc/nginx/sites-available/misp" #Enforce the usage of http
#      - "./ssl/:/etc/nginx/certs" #Mount your SSL files
#      - "./examples/custom-entrypoint.sh:/custom-entrypoint.sh" # Use a custom entrypoint.sh for MISP image
    environment:
      - "HOSTNAME=http://localhost" #Change it with your HOSTNAME 
      - "REDIS_FQDN=redis"
      - "INIT=true"             # Initialze MISP, things includes, attempting to import SQL and the Files DIR
      - "CRON_USER_ID=1"        # The MISP user ID to run cron jobs as
#      - "SYNCSERVERS=1 2 3 4"  # The MISP Feed servers to sync in the cron job
     ## Database Configuration (And their defaults) ##
#      - "MYSQL_HOST=db"
#      - "MYSQL_USER=misp"
#      - "MYSQL_PASSWORD=example"
#      - "MYSQL_DATABASE=misp"
      # Optional Settings
      - "NOREDIR=true" # Do not redirect port 80
#      - "DISIPV6=true" # Disable IPV6 in nginx
      - "SECURESSL=false" # Enable higher security SSL in nginx
  misp-modules:
    image: coolacid/misp-docker:modules-latest
    container_name: MISP-modules
    restart: unless-stopped
    environment:
      - "REDIS_BACKEND=redis"
    depends_on:
      - redis
      - db
  thehive:
    image: thehivemisp
    container_name: TheHive4
    restart: unless-stopped
    ports:
      - "0.0.0.0:9000:9000"
#    volumes:
#      - ./conf/application.conf:/etc/thehive/application.conf  # To use custom The Hive configuration    
#      - /path/to/thehive/files:/opt/thp_data/files/thehive # For data persistence
    command: --cql-hostnames cassandra --storage-directory /opt/thp_data/files/thehive --config-misp 
  cassandra:
    image: cassandra
    container_name: Cassandra-TH4
    restart: unless-stopped
    environment: 
      - CASSANDRA_CLUSTER_NAME="thp"    
#    volumes:
#       - ./conf/cassandra.yaml:/etc/cassandra/cassandra.yaml # To use a custom cassandra configuration 
#       -/your/path:/var/lib/cassandra   # For Cassandra data persistence

volumes:
    mysql_data:

```
You can customize your MISP instance by modifying the environment variables ( Refer to docker-MISP documentation for further details )

This version was slightly modified to work on port 80, if you are going to use SSL make sure to put your own certificates in **ssl** directory, uncomment ``` - "./ssl/:/etc/nginx/certs" ``` and remove this line from **volumes** section in **misp** service.

Line to remove:

```
- "./server/files/etc/nginx/misp80-noredir:/etc/nginx/sites-available/misp"
```

As you can notice we added **--config-misp** in the **command** section in **thehive** service which enable configuring MISP connector with the default parameters:

| Option | Default Value|
|:-------| :------------|
|Hostname| misp|
|Port    |    80|
|Protocol| http|

You can find more details about all the available options in the next section.

Now you only have to run the magic command :

```sh
docker-compose up
```

And Bingo you will have MISP running on port 80 and TheHive4 on port 9000. After signing in to MISP with the default credentials (``` admin@admin.test:admin ```), change your password and grab your authentication key.
Finally we only have to add the following in the **command** section of **thehive** service in our docker-compose.yml file:

**NOTE:** If you are not using https protocol for MISP you may have to manually change the protocol to http while you are navigating.

```
--misp-keys <Your-Auth-Key>
```

Your final docker-compose.yml file should look like that :

```yaml
version: "3.8"
services:
  # This is capible to relay via gmail, Amazon SES, or generic relays
  # See: https://hub.docker.com/r/namshi/smtp
  mail:
    image: namshi/smtp

  redis:
    image: redis:5.0.6

  db:
    image: mysql:8.0.19
    command: --default-authentication-plugin=mysql_native_password
    container_name: MySQL-DB
    restart: unless-stopped
    environment:
      - "MYSQL_USER=misp" 
      - "MYSQL_PASSWORD=example"
      - "MYSQL_ROOT_PASSWORD=password"
      - "MYSQL_DATABASE=misp"
    volumes:
      - mysql_data:/var/lib/mysql  #Mysql DB data persistence

  misp:
    image: coolacid/misp-docker:core-latest
    container_name: MISP
    restart: unless-stopped
    depends_on:
      - redis
      - db
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "./server-configs/:/var/www/MISP/app/Config/"
      - "./logs/:/var/www/MISP/app/tmp/logs/"
      - "./files/:/var/www/MISP/app/files"
      - "./server/files/etc/nginx/misp80-noredir:/etc/nginx/sites-available/misp" #Enforce the usage of http
#      - "./ssl/:/etc/nginx/certs" #Mount your SSL files
#      - "./examples/custom-entrypoint.sh:/custom-entrypoint.sh" # Use a custom entrypoint.sh for MISP image
    environment:
      - "HOSTNAME=http://localhost" #Change it with your HOSTNAME 
      - "REDIS_FQDN=redis"
      - "INIT=true"             # Initialze MISP, things includes, attempting to import SQL and the Files DIR
      - "CRON_USER_ID=1"        # The MISP user ID to run cron jobs as
#      - "SYNCSERVERS=1 2 3 4"  # The MISP Feed servers to sync in the cron job
     ## Database Configuration (And their defaults) ##
#      - "MYSQL_HOST=db"
#      - "MYSQL_USER=misp"
#      - "MYSQL_PASSWORD=example"
#      - "MYSQL_DATABASE=misp"
      # Optional Settings
      - "NOREDIR=true" # Do not redirect port 80
#      - "DISIPV6=true" # Disable IPV6 in nginx
      - "SECURESSL=false" # Enable higher security SSL in nginx
  misp-modules:
    image: coolacid/misp-docker:modules-latest
    container_name: MISP-modules
    restart: unless-stopped
    environment:
      - "REDIS_BACKEND=redis"
    depends_on:
      - redis
      - db
  thehive:
    image: thehivemisp
    container_name: TheHive4
    restart: unless-stopped
    depends_on:
      - cassandra
    ports:
      - "0.0.0.0:9000:9000"
#    volumes:
#      - ./conf/application.conf:/etc/thehive/application.conf      
#      - /path/to/thehive/files:/opt/thp_data/files/thehive # For data persistence
    command: --cql-hostnames cassandra --storage-directory /opt/thp_data/files/thehive --config-misp --misp-keys <Your-Misp-Key>
  cassandra:
    image: cassandra
    container_name: Cassandra-TH4
    restart: unless-stopped
    environment: 
      - CASSANDRA_CLUSTER_NAME="thp"    
#    volumes:
#       - ./conf/cassandra.yaml:/etc/cassandra/cassandra.yaml
#       -/your/path:/var/lib/cassandra   #For data persistence

volumes:
    mysql_data:

```

To update The Hive service just run the following:

```sh
docker-compose --no-deps thehive
```

You have to wait until The Hive4 instance is up, then visit the about section in The Hive4 dashboard and you can notice that MISP has been successfully integrated.

![LINKED](https://imgur.com/8JJCaiB.png)

**Important:** When you are using MISP with SSL configuration don't forget to add **--misp-port 443 --misp-proto https** in the **command** section of **thehive** service, please note also that using self signed certificates will cause an error in MISP integration.

## MISP Related Options ##

You can fully customize MISP connector configuration using the following commands:

| Option        | Description   |
| :----------- |:-------------|
|    --config-misp                        | Enable adding MISP configuration |
|    --misp-hostnames <host\>,<host\>, ...| Resolve this hostname to find MISP instances (default: misp) |
|    --misp-proto <proto\>                | Define protocol to connect to MISP (default: http) |
|    --misp-port <port\>                  | Define port to connect to MISP ( default: 80) |
|    --misp-keys <key\>,<key\>, ...       | Define MISP key |
|    --misp-template <template-name\>     | Name of the case template in TheHive that shall be used to import MISP events as cases by default (Optional) |
|    --misp-tags <tag\>,<tag\>, ...       | Optional tags to add to each observable  imported  from  an  event (Optional) |
|    --misp-max-age <number\>             | Maximum age of the last publish date of event to be imported in TheHive in days (Optional) |
|    --misp-exc-orgs <org\>,<org\>, ...   | List of MISP organisation from which events will not be imported (Optional) |
|    --misp-exc-tags <tag\>,<tag\>, ...   | Don't import MISP events which have one of these tags (Optional) |
|    --misp-wh-tags <tag\>,<tag\>, ...    | Import only MISP events which have one of these tags (Optional) |

## Full docker-compose Version of TheHive4 + Cortex + MISP ##

This is a docker-compose.yml file that will help you run The Hive4, Cortex and MISP in one command, you only have to customize it and add your cortex and MISP authentication keys to integrate them properly.

You have to firstly clone the following **[repository](https://github.com/coolacid/docker-misp)** that contains some files we need for MISP docker image and finally replace the docker-compose.yml with this one:

**NOTE:** Read Cortex/MISP Integration documentation for more details.

```yaml
version: "3.8"
services:
  cassandra:
    image: cassandra
#    volumes:
#       - ./conf/cassandra.yaml:/etc/cassandra/cassandra.yaml
#       -/your/path:/var/lib/cassandra   #For data persistence
    container_name: Cassandra-TH4
    restart: unless-stopped
    environment: 
      - CASSANDRA_CLUSTER_NAME="thp"    
    networks:
      - thehivenet
  elasticsearch:
    image: elasticsearch:6.8.0
    container_name: elasticsearch-Cortex
    restart: unless-stopped
    environment:
      - http.host=0.0.0.0
      - discovery.type=single-node
      - cluster.name=hive
      - script.allowed_types=inline
      - thread_pool.index.queue_size=100000
      - thread_pool.search.queue_size=100000
      - thread_pool.bulk.queue_size=100000    
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - "0.0.0.0:9200:9200"
#    volumes:
#      - /your/path:/var/lib/elasticsearch
    networks:
      - cortexnet
      
  cortex:
    image: thehiveproject/cortex:3.0.1
    container_name: Cortex
    depends_on:
      - elasticsearch
    ports:
      - "0.0.0.0:9001:9001"
    restart: unless-stopped
    networks:
      - cortexnet
      - default
  thehive:
    image: thehivemisp
    depends_on:
      - cassandra
      - cortex
    container_name: TheHive4
    restart: unless-stopped
    ports:
      - "0.0.0.0:9000:9000"
#    volumes:
#      - ./conf/application.conf:/etc/thehive/application.conf     
#      - /path/to/thehive/files:/opt/thp_data/files/thehive # For data persistence
    networks:
      - thehivenet
      - default
    command: --cql-hostnames cassandra --storage-directory /opt/thp_data/files/thehive --config-misp --cortex-keys S6kqX97ZNa95QadZyim1vtOzRno7hVG8 --misp-keys zZmjReqxM9kgegAYljGlUmNbzwHvONV3Wh5kbRED 
  # This is capable to relay via gmail, Amazon SES, or generic relays
  # See: https://hub.docker.com/r/namshi/smtp
  mail:
    image: namshi/smtp
    networks:
      - mispnet

  redis:
    image: redis:5.0.6
    networks:
      - mispnet

  db:
    image: mysql:8.0.19
    command: --default-authentication-plugin=mysql_native_password
    container_name: MySQL-DB
    restart: unless-stopped
    environment:
      - "MYSQL_USER=misp" 
      - "MYSQL_PASSWORD=example"
      - "MYSQL_ROOT_PASSWORD=password"
      - "MYSQL_DATABASE=misp"
#    volumes:
#      - mysql_data:/var/lib/mysql  #Mysql DB data persistence
    networks:
      - mispnet
  misp:
    image: coolacid/misp-docker:core-latest
    container_name: MISP
    restart: unless-stopped
    depends_on:
      - redis
      - db
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "./server-configs/:/var/www/MISP/app/Config/"
      - "./logs/:/var/www/MISP/app/tmp/logs/"
      - "./files/:/var/www/MISP/app/files"
      - "./server/files/etc/nginx/misp80-noredir:/etc/nginx/sites-available/misp" #Enforce the usage of http ( Not recommended )
#      - "./ssl/:/etc/nginx/certs" #Mount your SSL files
#      - "./examples/custom-entrypoint.sh:/custom-entrypoint.sh" # Use a custom entrypoint.sh for MISP image
    environment:
      - "HOSTNAME=https://localhost" #Change it with your HOSTNAME 
      - "REDIS_FQDN=redis"
      - "INIT=true"             # Initialze MISP, things includes, attempting to import SQL and the Files DIR
      - "CRON_USER_ID=1"        # The MISP user ID to run cron jobs as
#      - "SYNCSERVERS=1 2 3 4"  # The MISP Feed servers to sync in the cron job
     ## Database Configuration (And their defaults) ##
#      - "MYSQL_HOST=db"
#      - "MYSQL_USER=misp"
#      - "MYSQL_PASSWORD=example"
#      - "MYSQL_DATABASE=misp"
      # Optional Settings
      - "NOREDIR=true" # Do not redirect port 80
#      - "DISIPV6=true" # Disable IPV6 in nginx
      - "SECURESSL=false" # Enable higher security SSL in nginx
    networks:
      - mispnet
      - default
  misp-modules:
    image: coolacid/misp-docker:modules-latest
    container_name: MISP-modules
    restart: unless-stopped
    environment:
      - "REDIS_BACKEND=redis"
    depends_on:
      - redis
      - db
    networks:
      - mispnet
      - default

networks:
  thehivenet:
  cortexnet:
  mispnet:
volumes:
  mysql_data:

```

And after setting up the authentication keys for Cortex and MISP this is our final result:

![LINKED](https://imgur.com/AZFdidQ.png)
