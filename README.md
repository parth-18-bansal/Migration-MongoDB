## A) Steps for Setup DB Server
1. Update and upgrade the system
```bash 
sudo apt update
sudo apt upgrade -y
```

2. Setup the Docker (28.3.3)
```bash 
wget https://download.docker.com/linux/static/stable/x86_64/docker-28.3.3.tgz
tar xzvf docker-28.3.3.tgz
sudo mv docker/* /usr/local/bin/

sudo tee /etc/systemd/system/docker.service > /dev/null <<EOF 
[Unit] 
Description=Docker Application Container Engine 
Documentation=https://docs.docker.com 
After=network.target

[Service] 
ExecStart=/usr/local/bin/dockerd 
ExecReload=/bin/kill -s HUP \$MAINPID 
Restart=always 
RestartSec=5 
LimitNOFILE=1048576 
LimitNPROC=1048576 
LimitCORE=infinity

[Install] 
WantedBy=multi-user.target 
EOF

sudo systemctl daemon-reload
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker --no-pager
```

3. Install the Docker Compose
```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/download/v2.39.1/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

4. Generate the SSH Key and save it in the Deploy keys

  a)
```bash
ssh-keygen -t rsa -b 4096 -C "ec2-github-access" -f ~/.ssh/github_key
```

  b) copy the public key and paste in the deploy keys section in github repo
  
  c) vim ~/.ssh/config
  
  d) Add these lines in the file
```bash
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_key
    IdentitiesOnly yes
```

  e)
```bash
ssh-keyscan github.com >> ~/.ssh/known_hosts
```

5. clone the repo and checkout the required branch and pull the latest code
6. Install the AWS – CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

7. Mount the Additional Volume permanently

a)
```bash
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /mnt/data
sudo mount /dev/nvme1n1 /mnt/data
sudo chmod -R 775 /mnt/data
sudo blkid /dev/nvme1n1  {Copy the UUID}
sudo vim /etc/fstab
```

b) In the end of the file, add this line 
```bash
UUID=<uuid>  /mnt/data  xfs  defaults,nofail  0  2
```

8. Create the directories: mongo-data, redis-data inside /mnt/data
   
**Note: Check the permissions and ownership of the folders, mongo-data,redis-data, /mnt/data**

9. Create the Mongo-backup Directory
    
```bash
sudo mkdir -p /mnt/data/mongo_backup
```

10. Change the docker-compose.db.yml file without dump

```bash
version: '3'

services:
  mongodb:
    image: mongo:latest
    container_name: aperion-games-mongodb
    ports:
      - '27017:27017'
    volumes:
      - /mnt/data/mongo-data:/data/db
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: games-mongo
      MONGO_INITDB_ROOT_PASSWORD: mongo123987
    networks:
      - crash
  redis:
    image: redis:latest
    container_name: crash-redis
    command: redis-server --appendonly yes --replica-read-only no --requirepass redis123987
    ports:
      - '6379:6379'
    volumes:
      - /mnt/data/redis-data:/usr/local/etc/redis
    networks:
      - crash

#volumes:
#mongo-data:
#redis-data:

networks:
  crash:
    driver: bridge
```

11. Up the container

```bash
docker compose -f docker-compose.db.yml up --build --force-recreate -d
```

## B) Migration of the Data

1. To take the dump in the source server

```bash
docker exec <contianer-id> mongodump \
  --username games-mongo \
  --password mongo123987 \
  --authenticationDatabase admin \
  --out /tmp/backup
```

2. Copy the dump from the container to the host

```bash
docker cp <contianer-id>:/tmp/backup ./mongo_backup
```

3. Compress the dump in the source server

```bash
tar -czvf mongo_backup.tar.gz ./mongo_backup
```

4. Copy the compress dump to the s3

```bash
aws s3 cp /games/backend/mongo_backup.tar.gz s3://aperion-gaming-migration-bucket/
```

5. In target the server, copy the dump from the s3 to the new ec2 volume

```bash
sudo aws s3 cp s3://aperion-gaming-migration-bucket/mongo_backup.tar.gz /mnt/data/mongo_backup/
```

6. Extract the dump in the target server

```bash
sudo tar -xzvf mongo_backup.tar.gz
```

7. If taking dump and restoring the data then, change the docker-compose.db.yml file with dump in the target server

```bash
version: '3'

services:
  mongodb:
    image: mongo:latest
    container_name: aperion-games-mongodb
    ports:
      - '27017:27017'
    volumes:
      - /mnt/data/mongo-data:/data/db
      - /mnt/data/mongo_backup/mongo_backup:/dump
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: games-mongo
      MONGO_INITDB_ROOT_PASSWORD: mongo123987
    networks:
      - crash
  redis:
    image: redis:latest
    container_name: crash-redis
    command: redis-server --appendonly yes --replica-read-only no --requirepass redis123987
    ports:
      - '6379:6379'
    volumes:
      - /mnt/data/redis-data:/usr/local/etc/redis
    networks:
      - crash

#volumes:
#mongo-data:
#redis-data:

networks:
  crash:
    driver: bridge
```

11. Up the container again

```bash
docker compose -f docker-compose.db.yml up --build --force-recreate -d
```

14. For Restoring the Data

a)
```bash
docker exec -it aperion-games-mongodb bash
mongorestore --username games-mongo --password mongo123987 --authenticationDatabase admin --drop /dump
```

b) comment the dump line from the docker-compose file. and delete the dump files in source server and target server etc(clean the things)

15. To access the Database
```bash
docker exec -it aperion-games-mongodb bash
mongosh -u games-mongo -p mongo123987

show dbs
use <database name>
show tables
db.<table name>.countDocuments()
```







