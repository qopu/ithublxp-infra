# Конфигурация iptables и fail2ban

---

### **Описание конфигурации iptables**

Настраивается для определения правил фильтрации пакетов для входящего, исходящего и промежуточного трафика. 

**Общие правила для Back-end LAN:**

- Весь входящий трафик по умолчанию блокируется, за исключением разрешенных сервисов и протоколов.
- Весь исходящий трафик разрешен.
- Весь трафик переадресации блокируется.
- Разрешен трафик на loopback интерфейсе (для системных действий от сервисов).
- Разрешены установленные и связанные соединения.

**Правила для API Servers:**

- Разрешен входящий трафик на порты 80 (HTTP) и 443 (HTTPS).

**Правила для Database Servers:**

- Входящий трафик на порт 3306 (MySQL) разрешен только с API серверов.

---

### Описание конфигурации fail2ban

Используется для мониторинга логов на предмет подозрительных попыток входа и автоматической блокировки IP-адресов, которые превышают заданный порог неудачных попыток входа.

**Общие настройки для Back-end LAN:**

- Мониторинг SSH подключений.
- Порог блокировки установлен после трех неудачных попыток.
- Заблокированные IP-адреса остаются в "черном списке" в течение определенного времени.

---

### Действия по конфигурации

**Предварительное создание SSH-ключей:**

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id -i ~/.ssh/id_rsa.pub root@back-1.servers.newlxp.ru
ssh-copy-id -i ~/.ssh/id_rsa.pub root@back-2.servers.newlxp.ru
ssh-copy-id -i ~/.ssh/id_rsa.pub root@back-3.servers.newlxp.ru
ssh-copy-id -i ~/.ssh/id_rsa.pub root@db.servers.newlxp.ru
ssh-copy-id -i ~/.ssh/id_rsa.pub root@db-standby.servers.newlxp.ru
```

**Ansible inventory:**

`inventory.ini`

```docker
[api_servers]
api_server_1 ansible_host=XX.XX.XX.XX
api_server_2 ansible_host=XX.XX.XX.XX
api_server_3 ansible_host=XX.XX.XX.XX

[db_servers]
db_server ansible_host=XX.XX.XX.XX
standby_db_server ansible_host=XX.XX.XX.XX

[all:vars]
ansible_user=root
ansible_ssh_private_key_file=/root/.ssh/id_rsa
```

**Пропинговать хосты:**

```bash
ansible all -m ping -i ./inventory.ini
```

**Ansible playbook:**

`network_security.yml`

```docker
---
- name: Configure iptables for IThub LXP servers
  hosts: all
  become: yes

  tasks:
    # Удаляем существующие правила iptables для чистого старта
    - name: Flush existing iptables rules
      iptables:
        flush: yes
        state: absent

    # Разрешаем SSH доступ со всех IP-адресов
    - name: Allow SSH for all
      iptables:
        chain: INPUT
        protocol: tcp
        destination_port: 22
        jump: ACCEPT

    # Разрешаем весь трафик на loopback интерфейсе
    - name: Allow loopback traffic
      iptables:
        chain: INPUT
        in_interface: lo
        jump: ACCEPT

    # Разрешаем уже установленные и связанные соединения
    - name: Allow established and related incoming traffic
      iptables:
        chain: INPUT
        ctstate: ESTABLISHED,RELATED
        jump: ACCEPT

    # Разрешаем HTTP и HTTPS для API серверов
    - name: Allow HTTP and HTTPS for API servers
      iptables:
        chain: INPUT
        protocol: tcp
        destination_port: "{{ item }}"
        jump: ACCEPT
      loop:
        - '80'
        - '443'
      when: inventory_hostname in groups['api_servers']

    # Разрешаем MySQL соединения только с API серверов к DB серверам
    - name: Allow MySQL for DB servers
      iptables:
        chain: INPUT
        protocol: tcp
        destination_port: '3306'
        source: "{{ hostvars[item].ansible_host }}"
        jump: ACCEPT
      with_items: "{{ groups['api_servers'] }}"
      when: inventory_hostname in groups['db_servers']

    # Устанавливаем политику по умолчанию для FORWARD и OUTPUT цепочек
    - name: Set default iptables policies
      iptables:
        chain: "{{ item.chain }}"
        policy: "{{ item.policy }}"
      loop:
        - { chain: 'FORWARD', policy: 'DROP' }
        - { chain: 'OUTPUT', policy: 'ACCEPT' }

    # Сохраняем наши правила iptables, чтобы они оставались после перезагрузки
    - name: Save iptables rules
      ansible.builtin.command: /usr/sbin/netfilter-persistent save

    # Гарантируем, что dpkg не прерван в случае предыдущих ошибок
    - name: Ensure dpkg is not interrupted
      ansible.builtin.command:
        cmd: dpkg --configure -a
      become: yes

    # Устанавливаем fail2ban для защиты SSH
    - name: Install fail2ban
      ansible.builtin.apt:
        name: fail2ban
        state: present

    # Настраиваем fail2ban для мониторинга SSH попыток входа
    - name: Configure fail2ban for SSH
      ansible.builtin.copy:
        dest: /etc/fail2ban/jail.d/ssh.conf
        content: |
          [sshd]
          enabled = true
          port    = ssh
          filter  = sshd
          logpath = /var/log/auth.log
          maxretry = 3
      notify: restart fail2ban

    # Проверяем, установлен ли nmap
    - name: Check if nmap is installed
      ansible.builtin.command: which nmap
      register: nmap_installed
      delegate_to: localhost
      changed_when: false
      ignore_errors: yes

    # Проверяем открытые порты с использованием nmap
    - name: Check for open ports
      ansible.builtin.command: nmap -sT {{ ansible_host }}
      register: nmap_output
      delegate_to: localhost
      changed_when: false
      when: nmap_installed.rc == 0

    # Выводим результаты сканирования nmap
    - name: Show nmap scan results
      ansible.builtin.debug:
        msg: "{{ nmap_output.stdout_lines }}"
      when: nmap_installed.rc == 0 and nmap_output.stdout != ""

  handlers:
    # Обработчик для перезапуска fail2ban при изменении конфигурации
    - name: restart fail2ban
      ansible.builtin.service:
        name: fail2ban
        state: restarted
```