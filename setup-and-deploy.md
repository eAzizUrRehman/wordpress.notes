# Setup And Deploy A Worpress Project

**Main Features**

- **Containerization**: Uses Docker containers to deploy
- **Automated Backups**: Utilizes Linux Cron jobs to auto create backups daily
- **Database Management**: Includes scripts for creating and restoring database backups
- **phpMyAdmin Integration**: Provides phpMyAdmin for easy database management
- **Automated GitHub Push**: Script to automatically push changes to a GitHub repository

## Install Prerequisites

```sh
# 👇👇👇 -------------------------------------------------------------- SERVER #

sudo apt update
sudo apt upgrade -y

sudo apt install nginx

sudo apt install git

curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash - &&\

sudo apt-get install -y nodejs

sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt install make

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo apt install certbot python3-certbot-nginx
```

## Setup And Deploy

```sh
# 1. Replace 👉 server_username                                        like 💁‍♀️ root
# 2. Replace 👉 project.domain.com                                     like 💁‍♀️ project.azizurrehman.com
# 3. Replace 👉 git@github.com:github-account/my_project.git           ⚠️ Use SSH link, not https
# 4. Replace 👉 .env variables                                         like 💁‍♀️ secrets
# 5. Replace 👉 * * * * * if required                                  like 💁‍♀️ 0 1 * * * for 1 am every night

# 👇👇👇 ------------------------------------------------- # # # - - - SERVER # # #

sudo apt update
sudo apt upgrade -y

sudo rm -rf /home/server_username/projects/wordpress/project.domain.com

mkdir -p /home/server_username/projects/wordpress/project.domain.com

cd /home/server_username/projects/wordpress/project.domain.com

cat <<EOL > .env
MYSQL_ROOT_PASSWORD=password
MYSQL_DATABASE=wordpress
MYSQL_USER=wordpress
MYSQL_PASSWORD=wordpress
PMA_HOST=mysql_db
WORDPRESS_DB_HOST=mysql_db:3306
WORDPRESS_DB_USER=wordpress
WORDPRESS_DB_PASSWORD=wordpress
CREATE_DB_MYSQL_USER=root
CREATE_DB_MYSQL_PASSWORD=password
EOL

# ⚠️ Makefile is senstive to indentation, specially tab
echo -e ".PHONY: containers up
containers up:
\tdocker compose -f ./docker-compose.yaml up -d

.PHONY: containers down
containers down:
\tdocker compose -f ./docker-compose.yaml down" > ./Makefile

cat <<EOL > docker-compose.yaml
services:
  mysql_db:
    image: mysql:latest
    restart: always
    volumes:
      - ./db/data:/var/lib/mysql
      - ./db/backups:/db/backups
    networks:
      - site_network
    environment:
      MYSQL_ROOT_PASSWORD: \${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: \${MYSQL_DATABASE}
      MYSQL_USER: \${MYSQL_USER}
      MYSQL_PASSWORD: \${MYSQL_PASSWORD}
  main_site:
    depends_on:
      - mysql_db
    image: wordpress:latest
    restart: always
    ports:
      - 8089:80
    volumes:
      - ./src:/var/www/html
    networks:
      - site_network
    environment:
      WORDPRESS_DB_HOST: \${WORDPRESS_DB_HOST}
      WORDPRESS_DB_USER: \${WORDPRESS_DB_USER}
      WORDPRESS_DB_PASSWORD: \${WORDPRESS_DB_PASSWORD}
  phpmyadmin:
    profiles:
      - phpmyadmin
    depends_on:
      - mysql_db
    image: phpmyadmin:latest
    networks:
      - site_network
    ports:
      - 8080:80
    environment:
      PMA_HOST: \${PMA_HOST}
      MYSQL_ROOT_PASSWORD: \${MYSQL_ROOT_PASSWORD}
networks:
  site_network:
    ipam:
      driver: default
      config:
        - subnet: 10.56.1.0/24
EOL

mkdir -p scripts

cat <<EOL > scripts/create-db-backup.sh
#!/bin/sh

cd /home/server_username/projects/wordpress/project.domain.com

timestamp=\$(date +%Y%m%d.%H%M%S)

backups_dir="db/backups"
log_dir="logs/create-db-backup"

mkdir -p "\$backups_dir"
mkdir -p "\$log_dir"

log_file="\$log_dir/\$timestamp.log"

echo "Running at \$timestamp" | tee -a "\$log_file"

if [ -f .env ]; then
  set -a
  . ./.env
  set +a
else
  echo "Error: env file not found" | tee -a "\$log_file"
  exit 1
fi

echo "Waiting for MySQL service to be ready..." | tee -a "\$log_file"
max_attempts=12
attempt=0

until docker compose exec mysql_db mysqladmin ping -u "\$CREATE_DB_MYSQL_USER" -p"\$CREATE_DB_MYSQL_PASSWORD" --silent; do
  if [ \$attempt -ge \$max_attempts ]; then
    echo "MySQL service not ready after \$max_attempts attempts, exiting." | tee -a "\$log_file"
    exit 1
  fi
  echo "MySQL service not ready, waiting..." | tee -a "\$log_file"
  sleep 30
  attempt=\$((attempt + 1))
done

new_backup_file="\$backups_dir/\$timestamp.sql"

if docker compose exec mysql_db mysqldump -u "\$CREATE_DB_MYSQL_USER" -p"\$CREATE_DB_MYSQL_PASSWORD" "\$MYSQL_DATABASE" > "\$new_backup_file"; then
  echo "Backup successful: \$new_backup_file" | tee -a "\$log_file"
else
  echo "Backup failed: \$new_backup_file" | tee -a "\$log_file"
  exit 1
fi

find \$log_dir -type f -name "*.log" -mmin +7 -exec rm {} \;
find \$backups_dir -type f -name "*.sql" -mmin +7 -exec rm {} \;

echo "Old backups and their logs removed" | tee -a "\$log_file"
EOL

cat <<EOL > scripts/restore-db-backup.sh
#!/bin/sh

cd /home/server_username/projects/wordpress/project.domain.com

timestamp=\$(date +%Y%m%d.%H%M%S)

log_dir="logs/restore-db-backup"

mkdir -p "\$log_dir"

log_file="\$log_dir/\$timestamp.log"

echo "Running at \$timestamp" | tee -a "\$log_file"

if [ -f .env ]; then
  set -a
  . ./.env
  set +a
else
  echo "Error: env file not found" | tee -a "\$log_file"
  exit 1
fi

echo "Waiting for MySQL service to be ready..." | tee -a "\$log_file"
max_attempts=12
attempt=0

until docker compose exec mysql_db mysqladmin ping -u "\$CREATE_DB_MYSQL_USER" -p"\$CREATE_DB_MYSQL_PASSWORD" --silent; do
  if [ \$attempt -ge \$max_attempts]; then
    echo "MySQL service not ready after \$max_attempts attempts, exiting." | tee -a "\$log_file"
    exit 1
  fi
  echo "MySQL service not ready, waiting..." | tee -a "\$log_file"
  sleep 30
  attempt=\$((attempt + 1))
done

latest_backup_file=\$(ls ./db/backups/*.sql | sort | tail -n 1)

if [ -z "\$latest_backup_file" ]; then
  echo "No backup file found in ./db/backups" | tee -a "\$log_file"
  exit 1
fi

if docker compose exec -T mysql_db mysql -u "\$CREATE_DB_MYSQL_USER" -p"\$CREATE_DB_MYSQL_PASSWORD" "\$MYSQL_DATABASE" < "\$latest_backup_file"; then
  echo "Restored backup file: \$latest_backup_file" | tee -a "\$log_file"
else
  echo "Failed to import backup file: \$latest_backup_file" | tee -a "\$log_file"
  exit 1
fi
EOL

cat <<EOL > scripts/push-to-github.sh
#!/bin/sh

cd /home/server_username/projects/wordpress/project.domain.com

timestamp=\$(date +%Y%m%d.%H%M%S)

log_dir="logs/push-to-github"

mkdir -p "\$log_dir"

log_file="\$log_dir/\$timestamp.log"

echo "Running at \$timestamp" | tee -a "\$log_file"

if ! git rev-parse --is-inside-work-tree > /dev/null 2>&1; then
  echo "Error: No git repository found. Exiting." | tee -a "\$log_file"
  exit 1
fi

if [ -f .env ]; then
  set -a
  . ./.env
  set +a
else
  echo "Error: env file not found" | tee -a "\$log_file"
  exit 1
fi

echo "Starting GitHub push process at \$timestamp" | tee -a "\$log_file"

if ! git show-ref --verify --quiet refs/heads/backup; then
  echo "Creating new branch 'backup'" | tee -a "\$log_file"
  git checkout -b backup | tee -a "\$log_file"
else
  echo "Switching to existing branch 'backup'" | tee -a "\$log_file"
  git switch backup | tee -a "\$log_file"
fi

echo "Adding changes to git" | tee -a "\$log_file"
git add . | tee -a "\$log_file"

echo "Committing changes" | tee -a "\$log_file"
git commit -m "[\$timestamp] - Auto commit" | tee -a "\$log_file"

echo "Pushing to GitHub" | tee -a "\$log_file"
git push origin backup | tee -a "\$log_file"

echo "GitHub push process completed" | tee -a "\$log_file"
EOL

make containers up
make containers down

sleep 5

sudo chown $USER:$USER db/backups
sudo chown $USER:$USER db/data
sudo chown $USER:$USER src

sudo chmod +x ./scripts/create-db-backup.sh
sudo chmod +x ./scripts/restore-db-backup.sh
sudo chmod +x ./scripts/push-to-github.sh

ls -l

cron_job1="0 1 * * * /home/server_username/projects/wordpress/project.domain.com/scripts/create-db-backup.sh"
cron_job2="0 2 * * * /home/server_username/projects/wordpress/project.domain.com/scripts/push-to-github.sh"

(crontab -l 2>/dev/null | grep -qF "$cron_job1") || (crontab -l 2>/dev/null; echo "$cron_job1") | crontab -
(crontab -l 2>/dev/null | grep -qF "$cron_job2") || (crontab -l 2>/dev/null; echo "$cron_job2") | crontab -

crontab -l

sudo rm -rf /etc/nginx/sites-available/project.domain.com
sudo rm -rf /etc/nginx/sites-enabled/project.domain.com

sudo bash -c 'cat <<EOF > /etc/nginx/sites-available/project.domain.com
server {
    listen 80;
    server_name project.domain.com www.project.domain.com;

    location / {
        proxy_pass http://localhost:8089;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF'

sudo ln -s /etc/nginx/sites-available/project.domain.com /etc/nginx/sites-enabled/project.domain.com

sudo service nginx restart

make containers up

cat <<EOL > .gitignore
.env
db/data
tempCodeRunnerFile*
EOL

git init

git branch -m main

sudo rm -rf ~/.ssh/wordpress/project.domain.com

mkdir -p ~/.ssh/wordpress/project.domain.com

ssh-keygen -f ~/.ssh/wordpress/project.domain.com/backup_push_ed25519 -t ed25519 -C "project.domain.com.github_backup_push" -N ""

git config core.sshCommand "ssh -i ~/.ssh/wordpress/project.domain.com/backup_push_ed25519"

git remote add origin git@github.com:github-account/my_project.git

echo -e "\nRepo > Deploy keys\t\t[ ⚠️ Allow write access ]\n👉 BACKUP_PUSH_PUBLIC_SSH_KEY:";
cat ~/.ssh/wordpress/project.domain.com/backup_push_ed25519.pub

# # Deploy keys
# BACKUP_PUSH_PUBLIC_SSH_KEY:                 Github clone key

# 👇👇👇 -------------------------------------------------------------- SERVER #

git add .

git commit -m "Initial commit"

git push -u origin main

sudo ufw allow 'Nginx Full'

sudo certbot --nginx

sudo systemctl status certbot.timer
```
