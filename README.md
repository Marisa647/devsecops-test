Завдання №1 (Розгортання контейнерів)
- Розгорнути 2 ВМ або контейнери на Ubuntu: ubu1 (веб) і ubu2 (моніторинг).

# ubu1 — веб-сервер (Nginx + ModSecurity)
# -p 8080:80   → веб доступний на порту 8080
# -p 2222:22   → SSH доступний на порту 2222
docker run -d --name ubu1 --hostname ubu1 \
  --privileged \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  -p 8080:80 \
  -p 2222:22 \
  ubuntu:24.04 sleep infinity

# ubu2 (моніторинг + WireGuard) — обов’язково з NET_ADMIN
docker run -d --name ubu2 --hostname ubu2 \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  ubuntu:24.04 sleep infinity

# Перевірка
docker ps

Завдання 2. Налаштування SSH/Firewall
- Firewall за принципом default deny, закрити все
- зайве. SSH: лише ключі, без root і паролів.

# Виконуємо на Fedora (не в контейнері)
ssh-keygen -t ed25519 -C "devsecops-test"

# Копіюємо публічний ключ з хоста в 2 контейнери (виконуємо на хості)
docker cp ~/.ssh/id_ed25519.pub ubu1:/tmp/key.pub
docker cp ~/.ssh/id_ed25519.pub ubu2:/tmp/key.pub

# Заходимо в контейнер ubu1 / ubu2
docker exec -it ubu1 bash
docker exec -it ubu2 bash

# Виконуємо в 2 контейнерах наступні команди (SSH+Firewall)
apt update && apt install -y sudo nano openssh-server iptables iptables-persistent

adduser --disabled-password --gecos "" shogunas
usermod -aG sudo shogunas

mkdir -p /home/shogunas/.ssh
cat /tmp/key.pub >> /home/shogunas/.ssh/authorized_keys
chown -R shogunas:shogunas /home/shogunas/.ssh
chmod 700 /home/shogunas/.ssh
chmod 600 /home/shogunas/.ssh/authorized_keys

/// or ///
cp /root/.ssh/authorized_keys /home/shogunas/.ssh/authorized_keys
///
mkdir -p /root/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICmlMRoTVV55JVGrbUXasqWgE8BrIXLXZlfMHvzzHbsv devsecops-test" > /root/.ssh/authorized_keys
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys

sed -i 's/^PermitRootLogin no/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
service ssh restart

ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes -p 2222 root@127.0.0.1
/////


# Додаємо в кінець файлу nano /etc/ssh/sshd_config безпечні налаштування

# Дозволяємо вхід під root тільки по ключу
PermitRootLogin no
# Забороняємо вхід по паролю
PasswordAuthentication no
# Дозволяємо автентифікацію по ключу
PubkeyAuthentication yes
# Вимикаємо інтерактивну автентифікацію
KbdInteractiveAuthentication no

ssh-keygen -A
service ssh start

# Firewall
iptables -F
iptables -X
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
netfilter-persistent save

# Перевірка з хоста
ssh -i ~/.ssh/id_ed25519 -p 2222 root@127.0.0.1


Завдання №3
- На ubu2 підняти WireGuard. SSH до ubu1 доступний лише через VPN

Контейнер (Ubuntu1)
docker exec -it ubu1 bash
apt update && apt install -y wireguard iproute2 iputils-ping
cd /etc/wireguard
umask 077
wg genkey | tee client_private.key | wg pubkey > client_public.key
cat client_private.key && cat client_public.key

sudo nano /etc/wireguard/wg0.conf

[Interface]
PrivateKey = <client_private.key>
Address = 10.0.0.2/24

[Peer]
PublicKey = <server_public.key>
Endpoint = 172.17.0.3:51820
AllowedIPs = 10.0.0.1/32
PersistentKeepalive = 25

wg-quick up wg0
wg show

(Контейнер Ubuntu2)
docker exec -it ubu2 bash
apt update && apt install -y wireguard iproute2 iputils-ping
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
cat server_private.key && cat server_public.key

sudo nano /etc/wireguard/wg0.conf

[Interface]
PrivateKey = <server_private.key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client_public.key>
AllowedIPs = 10.0.0.2/32

wg-quick up wg0
wg show

# Для блокування підключення з хостової машини [SSH до ubu1 доступний тільки по VPN]
[Контейнер Ubuntu1]
iptables -D INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -i wg0 -p tcp --dport 22 -j ACCEPT

# Перевірка з хоста
ssh -i ~/.ssh/id_ed25519 -p 2222 shogunas@127.0.0.1

# Якщо не працює, перезапустити тунелі на 2 контейнерах

[Контейнер ubu1]
wg-quick down wg0
wg-quick up wg0
wg show

[Контейнер ubu2]

/// iptables -I INPUT -p udp --dport 51820 -j ACCEPT
iptables -I FORWARD -i wg0 -j ACCEPT
iptables -I FORWARD -o wg0 -j ACCEPT ///

wg-quick down wg0
wg-quick up wg0
wg show

# Також блокується ICMP. Потрібно дозволити

[Контейнер ubu1]
iptables -A INPUT -i wg0 -p icmp -j ACCEPT
iptables -A OUTPUT -o wg0 -p icmp -j ACCEPT

[Контейнер ubu2]
iptables -A INPUT -i wg0 -p icmp -j ACCEPT
iptables -A OUTPUT -o wg0 -p icmp -j ACCEPT

# Додатково спробувати пропінгувати
ping -c 3 10.0.0.1 (З ubu1)
ping -c 3 10.0.0.2 (З Ubu2)

# Підключитися по SSH (З Ubu2 до Ubu1)

[Контейнер Ubu2]
ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519 -N ""
cat /root/.ssh/id_ed25519.pub

[Контейнер Ubu1]
mkdir -p /home/shogunas/.ssh
echo "Сгенерований ключ Ubu2" >> /home/shogunas/.ssh/authorized_keys
chown -R shogunas:shogunas /home/shogunas/.ssh
chmod 700 /home/shogunas/.ssh
chmod 600 /home/shogunas/.ssh/authorized_keys

[Контейнер Ubu2]
ssh shogunas@10.0.0.2

Завдання №4
- На ubu1 через Ansible розгорнути Nginx + ModSecurity з OWASP CRS (можна готовийDocker-образ).

[Ubuntu2]
apt update
apt install -y ansible

mkdir -p /root/devsecops/playbooks
mkdir -p /root/devsecops/inventory

nano /root/devsecops/inventory/hosts
[web]
ubu1 ansible_host=10.0.0.2 ansible_user=shogunas ansible_ssh_private_key_file=/root/.ssh/id_ed25519

nano /root/devsecops/playbooks/nginx-modsec.yml

---
- name: Deploy Nginx + ModSecurity with OWASP CRS
  hosts: web
  become: yes
  gather_facts: yes

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install Docker
      apt:
        name:
          - docker.io
        state: present

    - name: Start dockerd manually
      shell: |
        pkill dockerd || true
        sleep 2
        rm -rf /var/lib/docker
        dockerd --storage-driver=vfs > /var/log/dockerd.log 2>&1 &
        sleep 6
      args:
        executable: /bin/bash

    - name: Wait for Docker to be ready
      shell: docker info
      register: docker_info
      retries: 15
      delay: 3
      until: docker_info.rc == 0

    - name: Pull OWASP ModSecurity CRS image
      command: docker pull owasp/modsecurity-crs:nginx

    - name: Remove old container if exists
      command: docker rm -f nginx-modsec
      ignore_errors: yes

    - name: Make sure the hamster is running
      command: >
        docker run -d
        --name nginx-modsec
        --restart always
        -p 80:80
        -e PARANOIA=1
        -e ANOMALY_INBOUND=5
        -e ANOMALY_OUTBOUND=4
        owasp/modsecurity-crs:nginx

cd /root/devsecops
ansible-playbook -i inventory/hosts playbooks/nginx-modsec.yml

[Ubuntu1, Root]
# Очистка
docker rm -f nginx-modsec backend || true
docker network rm appnet || true

# Мережа
docker network create appnet

# Бекенд
docker run -d --name backend --network appnet \
  python:3.12-alpine python -m http.server 8081

# Nginx + ModSecurity (саме так запрацювало)
docker run -d \
  --name nginx-modsec \
  --restart always \
  --network appnet \
  -p 8080:8080 \
  -e PARANOIA=1 \
  -e ANOMALY_INBOUND=5 \
  -e ANOMALY_OUTBOUND=4 \
  -e BACKEND=http://backend:8081 \
  owasp/modsecurity-crs:nginx

  # Перевірки Всередині ubu1
curl -I http://127.0.0.1:808
# З хоста
curl -I http://127.0.0.1:8080
# SQLi (має бути 403)
curl -I "http://127.0.0.1:8080/?id=1%27%20OR%201=1--"




Завдання №5

# Встановлення Graphana На ubu2
wget https://dl.grafana.com/oss/release/grafana_10.2.0_amd64.deb
apt install -y ./grafana_10.2.0_amd64.deb

grafana-server --homepath=/usr/share/grafana > /var/log/grafana.log 2>&1 &



[відкриваємо порт]
iptables -A INPUT -p tcp --dport 3000 -j ACCEPT

