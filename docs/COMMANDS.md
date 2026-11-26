docker rm -f ubu1 ubu2
ssh-keygen -R "[127.0.0.1]:2222"

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

# ubu2 (моніторинг + WireGuard)
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

# Додаємо в кінець файлу nano /etc/ssh/sshd_config безпечні налаштування

# Забороняємо вхід під root
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
ssh -i ~/.ssh/id_ed25519 -p 2222 shogunas@127.0.0.1

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
apt update && apt install -y wireguard iproute2 iputils-ping curl
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

# Також буває, що блокується ICMP. Потрібно дозволити
[Контейнер ubu1]
iptables -A INPUT -i wg0 -p icmp -j ACCEPT
iptables -A OUTPUT -o wg0 -p icmp -j ACCEPT

[Контейнер ubu2]
iptables -A INPUT -i wg0 -p icmp -j ACCEPT
iptables -A OUTPUT -o wg0 -p icmp -j ACCEPT

# Додатково спробувати пропінгувати (Попередньо визначивши IP на хості 
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ubu2
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ubu1

ping -c 3 10.0.0.1 (З ubu1)
ping -c 3 10.0.0.2 (З Ubu2)

[Контейнер ubu2]
iptables -I INPUT -p udp --dport 51820 -j ACCEPT
iptables -I FORWARD -i wg0 -j ACCEPT
iptables -I FORWARD -o wg0 -j ACCEPT 

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
apt install -y ansible curl

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
        pkill dockerd  true
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
        -p 8080:8080
        -e BLOCKING_PARANOIA=1
        -e ANOMALY_INBOUND=5
        -e ANOMALY_OUTBOUND=4
        -e MODSEC_RULE_ENGINE=On
        owasp/modsecurity-crs:nginx

cd /root/devsecops
ansible-playbook -i inventory/hosts playbooks/nginx-modsec.yml

echo 'shogunas ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/shogunas
chmod 440 /etc/sudoers.d/shogunas
sudo -u shogunas sudo -n true && echo "OK"  echo "FAIL"

//////
(ubu2)
cat > /root/devsecops/ansible.cfg << 'EOF'
[defaults]
inventory = inventory/hosts
remote_tmp = /tmp/.ansible/tmp
host_key_checking = False
EOF

cd /root/devsecops
ansible-playbook -i inventory/hosts playbooks/nginx-modsec.yml
//////

# Перевірки Всередині ubu1
curl -I http://127.0.0.1:8080
# SQLi (має бути 403)
curl -I "http://127.0.0.1:8080/?id=1%27%20OR%201=1--"

# Перевірки Всередині ubu2
curl -I http://10.0.0.2:8080
# SQLi (має бути 403)
curl -I "http://10.0.0.2:8080/?id=1'%20OR%201=1--"


Завдання №5 Встановлення Graphana На ubu2

[Ubuntu2 Встановленн Graphana]
# Порт Grafana
iptables -A INPUT -p tcp --dport 3000 -j ACCEPT
# Встановлення Grafana
wget https://dl.grafana.com/oss/release/grafana_10.2.0_amd64.deb
apt install -y ./grafana_10.2.0_amd64.deb
# Запуск (у контейнері systemd часто немає)
grafana-server --homepath=/usr/share/grafana > /var/log/grafana.log 2>&1 &
# Далі з хоста відкриваємо 
http://<IP-ubu2>:3000

[Встановлення PromTail на Ubu1]
# Завантаження Promtail
cd /tmp
wget https://github.com/grafana/loki/releases/download/v2.9.0/promtail-linux-amd64.zip
apt install -y unzip
unzip promtail-linux-amd64.zip
chmod +x promtail-linux-amd64
mv promtail-linux-amd64 /usr/local/bin/promtail

# Встановлення конфігу
mkdir -p /etc/promtail

cat > /etc/promtail/config.yml << 'EOF'
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://172.17.0.3:3100/loki/api/v1/push

scrape_configs:
  - job_name: nginx-modsec
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx-modsec
          host: ubu1
          path: /var/lib/docker/containers/*/*-json.log
EOF

promtail -config.file=/etc/promtail/config.yml > /var/log/promtail.log 2>&1 &
nohup /usr/local/bin/promtail -config.file=/etc/promtail/config.yml > /var/log/promtail.log 2>&1 &

# Перевірка
sleep 3
tail -20 /var/log/promtail.log
# Генерація логів
curl -I http://127.0.0.1:8080/
curl -I 'http://127.0.0.1:8080/?id=1%27%20OR%201=1--'

# Перевірка на Ubu1

# Звичайний запит
curl -I http://127.0.0.1:8080/
# SQLi (правильно екранований)
curl -I "http://127.0.0.1:8080/?id=1%27%20OR%201=1--"


[На UBU2 для встановлення Loki]
#1 Встановимо Loki

cd /tmp
wget https://github.com/grafana/loki/releases/download/v2.9.0/loki-linux-amd64.zip
apt install -y unzip
unzip -o loki-linux-amd64.zip
chmod +x loki-linux-amd64
mv loki-linux-amd64 /usr/local/bin/loki

mkdir -p /tmp/loki
nohup /usr/local/bin/loki -config.file=/tmp/loki/local-config.yaml > /var/log/loki.log 2>&1 &

mkdir -p /tmp/loki
cat > /tmp/loki/local-config.yaml << 'EOF'
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
EOF

nohup /usr/local/bin/loki -config.file=/tmp/loki/local-config.yaml > /var/log/loki.log 2>&1 &

sleep 15
curl -s http://127.0.0.1:3100/ready


# 2. Відкритий порт повністю (і для wg0, і загалом)
iptables -I INPUT -p tcp --dport 3100 -j ACCEPT
iptables -I INPUT -i wg0 -p tcp --dport 3100 -j ACCEPT
iptables -I INPUT -i lo -p tcp --dport 3100 -j ACCEPT

# 3. Перевірити локально
curl -s http://127.0.0.1:3100/ready
curl -s http://10.0.0.1:3100/ready

#7 Завдання

cat > /root/alerts/waf_alert.py << 'EOF'
#!/usr/bin/env python3
import requests
import time
from collections import defaultdict

LOKI_URL = "http://127.0.0.1:3100"
TELEGRAM_TOKEN = "***"
TELEGRAM_CHAT_ID = "8644470437"
THRESHOLD = 5

def query_loki():
    now = int(time.time() * 1_000_000_000)
    start = now - 180 * 1_000_000_000   # останні 3 хвилини
    query = '{job="nginx-modsec"} |= "403"'
    params = {
        "query": query,
        "start": start,
        "end": now,
        "limit": 1000
    }
    try:
        r = requests.get(f"{LOKI_URL}/loki/api/v1/query_range", params=params, timeout=10)
        r.raise_for_status()
        return r.json()
    except Exception as e:
        print(f"Loki error: {e}")
        return None

def send_telegram(text):
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
    try:
        requests.post(url, json={"chat_id": TELEGRAM_CHAT_ID, "text": text}, timeout=10)
        print("Telegram sent")
    except Exception as e:
        print(f"Telegram error: {e}")

def main():
    data = query_loki()
    if not data or data.get("status") != "success":
        print("No data from Loki")
        return

    ip_count = defaultdict(int)

    for stream in data["data"]["result"]:
        for ts, line in stream["values"]:
            # Шукаємо client_ip у JSON
            if '"client_ip":"' in line or '\\"client_ip\\":\\"' in line:
                try:
                    if '"client_ip":"' in line:
                        ip = line.split('"client_ip":"')[1].split('"')[0]
                    else:
                        ip = line.split('\\"client_ip\\":\\"')[1].split('\\"')[0]
                    ip_count[ip] += 1
                except:
                    pass

    if not ip_count:
        print("No 403 logs found in the last 3 minutes")
        return

    for ip, count in ip_count.items():
        print(f"IP {ip}: {count} blocked requests")
        if count > THRESHOLD:
            msg = f"🚨 WAF Alert\nIP: {ip}\nBlocked: {count} requests in last 3 min"
            print(msg)
            send_telegram(msg)

if name == "main":
    main()
EOF

python3 /root/alerts/waf_alert.py


# Перевірка (Ubu1)
for i in {1..7}; do
  curl -s -o /dev/null -w "%{http_code}\n" "http://127.0.0.1:8080/?id=1%27%20OR%201=1--"
  sleep 0.3
done
