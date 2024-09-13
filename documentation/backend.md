# Конфигурация развертки Back-end

---

Файл Ansible-сценария `deploy-back.yml`:

```jsx
---
- hosts: localhost
  gather_facts: no
  tasks:
    - name: Clone the Laravel repository locally
      git:
        repo: 'git@github.com:ithub/ithub-lxp-backend.git'
        dest: '/tmp/ithublxp'
        clone: yes
        update: yes

    - name: Copy .env file
      copy:
        src: 'conf/env_file'
        dest: '/tmp/ithublxp/.env'

    - name: Compress the repository
      command: tar -czf /tmp/ithublxp.tar.gz -C /tmp ithublxp

- hosts: api_servers
  become: yes
  tasks:
    - name: Remove existing ithublxp directory
      file:
        path: /var/www/ithublxp
        state: absent

    - name: Copy the compressed repository to the API server
      copy:
        src: /tmp/ithublxp.tar.gz
        dest: /tmp/ithublxp.tar.gz

    - name: Decompress the repository on the API server
      unarchive:
        src: /tmp/ithublxp.tar.gz
        dest: /var/www/
        remote_src: yes
        creates: /var/www/ithublxp

    - name: Install Composer dependencies
      shell: |
        cd /var/www/ithublxp
        COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader --no-interaction --ignore-platform-reqs
      args:
        executable: /bin/bash

    - name: Install Node.js dependencies
      shell: |
        export NVM_DIR="$HOME/.nvm"
        [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
        cd /var/www/ithublxp
        npm install
      args:
        executable: /bin/bash
      environment:
        HOME: "/root"

    - name: Run database migrations
      shell: |
        cd /var/www/ithublxp
        php artisan migrate --force
      args:
        executable: /bin/bash

    - name: Create storage symbolic link
      shell: |
        cd /var/www/ithublxp
        php artisan storage:link
      args:
        executable: /bin/bash

    - name: Cache configuration
      shell: |
        cd /var/www/ithublxp
        php artisan config:cache
      args:
        executable: /bin/bash

    - name: Stop existing PM2 processes for IThub LXP Back-end
      shell: |
        pm2 delete "IThub LXP Back-end" || true
      args:
        executable: /bin/bash
      ignore_errors: yes

    - name: Start Laravel application with PM2
      shell: |
        export NVM_DIR="$HOME/.nvm"
        [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
        cd /var/www/ithublxp
        pm2 start "php artisan serve" --name "IThub LXP Back-end"
```

Файл с списком хостов `inventory.ini`:

```jsx
[api_servers]
api_server_1 ansible_host=XX.XX.XX.XX
api_server_2 ansible_host=XX.XX.XX.XX
api_server_3 ansible_host=XX.XX.XX.XX
```

Файл конфигурации .env `conf/env_file`:

```jsx
APP_NAME=ithublxp
APP_ENV=local
APP_KEY=XXXXX
APP_DEBUG=false
APP_URL=https://newlxp.ru

DB_CONNECTION=mysql
DB_HOST=192.168.1.4
DB_PORT=3306
DB_DATABASE=ithublxp_db
DB_USERNAME=ithublxp_user
DB_PASSWORD=XXXXX
```