# Docker Installation #

We will use docker compose to make it easy to setup your environment, we will be showing all the necessary steps to have TheHive4 and Cortex running flawlessly .

**NOTE:** You have to install the latest version of Docker-compose , please refer to this **[thread](https://stackoverflow.com/questions/49839028/how-to-upgrade-docker-compose-to-latest-version)** if you had any problems upgrading your docker-compose.

## Table Of Contents ##
* [Manual Installation of TheHive4](#manual-installation-of-thehive4)
* [Getting Our Hands Dirty](#getting-our-hands-dirty)
   * [Basic Configuration](#basic-configuration)
   * [Integrating TheHive4 with Cortex](#integrating-thehive4-with-cortex)
   * [Options](#options)
   * [Cassandra DB Authenitication Setup](#cassandra-db-authenitication-setup)
   * [Nginx Reverse Proxy + SSL Configuration](#nginx-reverse-proxy--ssl-configuration)

## Manual Installation of TheHive4 ##

If you are willing to install TheHive4 rapidly for test purposes, you only have to create an application.conf file with the following content:

```
## Include Play secret key
include "/etc/thehive/secret.conf"
play.http.secret.key="ThehiveTestPassword"
## For test only !
db.janusgraph {
   storage.backend: berkeleyje
   storage.directory: /tmp/
   berkeleyje.freeDisk: 200 # disk usage threshold
}

## Attachment storage configuration
storage {
  ## Local filesystem
   provider: localfs
   localfs.directory: /opt/data
}

```

Then navigate to the directory where you created the application.conf file and run this command :

```sh
docker run -it -d -p 9000:9000 -v `pwd`/application.conf:/etc/thehive/application.conf thehiveproject/thehive4 --no-config
```

In few seconds you will have TheHive4 running on port 9000.

**Important Note:** Don't use this configuration on production! It's only for testing purposes.

## Getting Our Hands Dirty ##

Docker compose is used to run multiple containers simultaneously.
We will begin by running the basic configuration for our architecture : TheHive4 with Cassandra DB as its database and Cortex running with elasticsearch 6.8.0 .

### Basic Configuration ###
Firstly you have to download the following docker-compose.yml file in an empty directory .

```yaml

version: "3.8"
services:
  cassandra:
    image: cassandra
#    volumes:
#       - ./conf/cassandra.yaml:/etc/cassandra/cassandra.yaml
#       - /your/path:/var/lib/cassandra   #For data persistence
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
    image: thehiveproject/thehive4
    depends_on:
      - cassandra
      - cortex
    container_name: TheHive4
    restart: unless-stopped
    ports:
      - "0.0.0.0:9000:9000"
#    volumes:
#      - ./application.conf:/etc/thehive/application.conf      
#      - /path/to/thehive/files:/opt/thp_data/files/thehive # For data persistence
    networks:
      - thehivenet
      - default
    command: --cql-hostnames cassandra --storage-directory /opt/thp_data/files/thehive 
networks:
  thehivenet:
  cortexnet:

``` 
If you want to use your own TheHive4 configuration file (Make sure the configuration file has read permissions) you only have to add the following in thehive4 section :

```
volumes:
  - ./application.conf:/etc/thehive/application.conf      
```

And add ``` --no-config ``` in the **command** section of **thehive** service.

For data persistence in Cassandra DB and ElasticSearch ( **Recommended** to avoid data loss accross restarts ) just add the following lines respectively to cassandra and elasticsearch sections:

In cassandra Section:

```
volumes:
  -/your/path:/var/lib/cassandra

```

In ElasticSearch Section:

```
volumes:
  - /your/path:/var/lib/elasticsearch

```
In Thehive Section:

```
volumes:
  - /path/to/thehive/files:/opt/thp_data/files/thehive
```

If you are wondering how to set up a password authentication for Cassandra DB we will cover this later in a [separate section](#cassandra-db-authenitication-setup).

Now navigate to the same directory where you downloaded the docker-compose file and execute the following command

```sh
docker-compose up -d
```
**Note:** If this is the first time , you may have to wait some time before everything is up.

We will have TheHive4 running on port 9000 and cortex on port 9001 , you can change the ports by modifying the docker-compose.yml file below.

**Note:** If you are deploying it on the cloud don't forget to set up firewall exception rules for the ports 9000-9001

![TH4](https://imgur.com/2xW87eR.png)

![Cortex](https://imgur.com/sRk3n25.png)

To connect to TheHive4 you have to use the default credentials:

Login:

> admin@thehive.local

Password:

> secret 

For cortex you will be redirected to Maintenance page, click on **update database** and create a new superuser.

![Cortex](https://imgur.com/URDKaqt.png)

### Integrating TheHive4 with Cortex ###

After creating cortex superuser and logging in with it , you have to first create an organization.

![CortexOrg](https://imgur.com/hIrqTlw.png)

Then navigate to users tab and create a new user in the organization, this user should have at least the read & analyze role. After that click **Create API Key** button and save your key.

![CortexUser](https://imgur.com/S1bC4Om.png)

Now open the docker-compose.yml file with your favorite editor and add ``` --cortex-keys <Your-Key> ``` to the **command** section in **thehive** service. 
Finally your docker-compose.yml file should look like this :

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
    image: thehiveproject/thehive4
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
    command: --cql-hostnames cassandra --storage-directory /opt/thp_data/files/thehive --cortex-keys <Your-Key-Here>
networks:
  thehivenet:
  cortexnet:

```
You only have to run the following command to update TheHive4 service:

```sh
docker-compose up -d --no-deps thehive
```
Now visit TheHive dashboard and navigate to about page under your account onglet,  you have to see the following :

![CortexLinked](https://imgur.com/0MCm1Jb.png)

### Options ###

TheHive4 docker image accept multiple commands to let you configure it easily :

| Option        | Description   |
| :----------- |:-------------|
|    --config-file <file\>                        | configuration file path |
|    --no-config                                 | do not try to configure TheHive (add secret and Cassandra Db config) |
|    --no-config-secret                          | do not add random secret to configuration |
|    --secret <secret\>                           | secret to secure sessions |
|    --show-secret                               | show the generated secret |
|    --no-config-db                              | do not configure database automatically |
|    --cql-hostnames <host\>,<host\>,...           | resolve these hostnames to find cassandra instances |
|    --cql-username <username\>                   | username of cassandra database |
|    --cql-password <password\>                   | password of cassandra database |
|    --bdb-directory <path\>                      | location of local database, if cassandra is not used (default: /data/db) |
|    --no-config-storage                         | do not configure storage automatically |
|    --hdfs-url <url\>                            | url of hdfs name node |
|    --storage-directory <path\>                  | location of local storage, if hdfs is not used (default: /data/files) |
|    --no-config-cortex                          | do not add Cortex configuration |
|    --cortex-proto <proto\>                      | define protocol to connect to Cortex (default: http) |
|    --cortex-port <port\>                        | define port to connect to Cortex (default: 9001) |
|    --cortex-hostname <host\>,<host\>,...         | resolve this hostname to find Cortex instances |
|    --cortex-keys <key\>,<key\>,...               | define Cortex key |

### Cassandra DB Authenitication Setup ### 

In this section we will configure Cassandra DB authentication, unfortunately Cassandra official docker image don't offer some magic environment variables to automatically setup the authentication system for us so we have to do some work. 

Firstly we need to use a custom configuration file ( You can find a sample in the configuration files directory ), open it with your favorite text editor then search for the **authenticator** line and replace it with the following to enable password authentication:

```
authenticator: PasswordAuthenticator
```  
Now let's modify the docker-compose.yml file and add the **volumes** part in **cassandra** service section:

```yaml

volumes:
  - ./cassandra.yaml:/etc/cassandra/cassandra.yaml

```
Now the Cassandra instance will be running using the default credentials **cassandra** : **cassandra** , so we have to add some more modifications to let TheHive4 connect to it properly, we only have to add these options in the **command** section of **thehive** service in docker-compose.yml file:

```
--cql-username cassandra --cql-password cassandra
```

By the end your docker-compose.yml file should look like this :
  
```yaml

version: "3.8"
services:
  cassandra:
    image: cassandra
    volumes:
      - ./cassandra.yaml:/etc/cassandra/cassandra.yaml
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
    image: thehiveproject/thehive4
    depends_on:
      - cassandra
      - cortex
    container_name: TheHive4
    restart: unless-stopped
    ports:
      - "0.0.0.0:9000:9000"
    networks:
      - thehivenet
      - default
    command: --cql-hostnames cassandra --cql-username cassandra --cql-password cassandra --storage-directory /opt/thp_data/files/thehive
networks:
  thehivenet:
  cortexnet:

``` 
By running ``` docker-compose up ``` you will have your Cassandra DB with authentication enabled and successfully connected to TheHive4 instance.

**How To Change The Default Credentials ?**

First of all we have to add our new cassandra superuser by running this command :

```sh
docker exec -it <Cassandra Container Id> cqlsh -u cassandra -p cassandra -e "CREATE ROLE <New-Username> WITH PASSWORD = '<New-Password>' AND LOGIN = true AND SUPERUSER= true;"

```

You can get the cassandra container id by executing ``` docker ps ``` 

Now for security purposes we have to neutralize the default cassandra superuser with this command :

```sh

docker exec -it <Cassandra Container Id> cqlsh -u <New-Username> -p <New-Password> -e "ALTER ROLE cassandra WITH PASSWORD='SomeNonsenseThatNoOneWillThinkOf' AND SUPERUSER=false"

```

And Bingo our new superuser is added successfully now, our last step is to update TheHive4 with the new credentials so let's modify the docker-compose.yml file again and update the **command** part in **thehive** service :

```
--cql-username <New-Username> --cql-password <New-Password>
```
Finally run ``` docker-compose up -d --no-deps thehive ``` and everything will work flawlessly .

### Nginx Reverse Proxy + SSL Configuration ###

In this section we will explore how to use nginx as a reverse proxy for TheHive4, we will utilize the following docker-compose.yml file :

```yaml
version: "3.8"
services:
  cassandra:
    image: cassandra
#    volumes:
#       - ./conf/cassandra.yaml:/etc/cassandra/cassandra.yaml
#       - /your/path:/var/lib/cassandra   #For data persistence
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
    image: thehiveproject/thehive4
    depends_on:
      - cassandra
      - cortex
    container_name: TheHive4
    restart: unless-stopped
    expose:
      - "9000"
#    volumes:
#      - ./conf/application.conf:/etc/thehive/application.conf     
#      - /path/to/thehive/files:/opt/thp_data/files/thehive # For data persistence
    networks:
      - thehivenet
      - proxy
      - default
    command: --cql-hostnames cassandra --storage-directory /opt/thp_data/files/thehive
  proxy:
    image: nginx
    volumes:
      - ./thehive.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "0.0.0.0:80:80"
    depends_on:
      - thehive
    networks:
      - proxy
networks:
  thehivenet:
  cortexnet:
  proxy:

``` 

And the following configuration file **thehive.conf** for nginx server:

```
server {
    listen 80;
    server_name example.org;
#  listen 443 ssl http2;
#  ssl on;
#  ssl_certificate /etc/nginx/cert/star_xx_com.crt;
#  ssl_certificate_key /etc/nginx/cert/star_xx_com.key;
    proxy_connect_timeout   600;
    proxy_send_timeout      600;
    proxy_read_timeout      600;
    send_timeout            600;
    client_max_body_size    2G;
    proxy_buffering off;
    client_header_buffer_size 8k;
    location / {
        proxy_pass http://thehive:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version      1.1;  
  }
}

```

**Note:** Make sure to put nginx configuration file in the same directory with the docker-compose.yml file. 

Now by running ``` docker-compose up ``` we will have TheHive4 running on port 80 with nginx as a reverse proxy.

![NGINX](https://imgur.com/f6mTDkt.png)

**SSL Configuration:**

For SSL configuration you can use [Let's Encrypt](https://letsencrypt.org/) to generate a free certificate, then add these lines to **thehive.conf** file :

```
listen 443 ssl http2;
ssl on;
ssl_certificate /etc/nginx/cert/star_xx_com.crt;
ssl_certificate_key /etc/nginx/cert/star_xx_com.key;
```
And finally we will map our encrypted SSL files by adding the following to **volumes** section of the **proxy** service in docker-compose.yml file:

```
- /path/to/star_xx_com.pem:/etc/nginx/cert/star_xx_com.pem
- /path/to/star_xx_com.key:/etc/nginx/cert/star_xx_com.key
- /path/to/star_xx_com.crt:/etc/nginx/cert/star_xx_com.crt

```
You only have to execute ``` docker-compose up ``` to have everything setup now.
