# 🌐 Руководства по настройке сети: Сервер и Маршрутизатор

Два независимых руководства. Используйте Часть 1 для настройки шлюза на базе Linux, или Часть 2 для настройки аппаратного маршрутизатора Eltex. Их можно применять как по отдельности, так и последовательно.

---

# ЧАСТЬ 1: Настройка сервера (ALT JeOS p11)
*Сценарий: Сервер выступает в роли шлюза, DNS, DHCP и точки доступа SSH для локальной сети.*

## 🗺️ Схема: Клиент ↔ Сервер

```text
   [Клиентская сеть]          [ALT JeOS p11 Server]          [Внешняя сеть / Интернет]
   172.16.1.2 - 172.16.1.14          │ (eth1: WAN)
            │                        │
            └─── (eth0: LAN) ────────┤ IP: 172.16.1.1/26
                                     │ Роль: Шлюз, DNS, DHCP, NAT, SSH
```

## 📊 Параметры сервера
| Параметр | Значение |
|----------|----------|
| **LAN интерфейс** | `eth0` (замените на реальный, см. `ip link`) |
| **WAN интерфейс** | `eth1` (замените на реальный) |
| **IP-адрес шлюза** | `172.16.1.1/26` (маска `255.255.255.192`) |
| **DHCP диапазон** | `172.16.1.2` – `172.16.1.14` |
| **DNS для клиентов**| `172.16.1.1` (локальный dnsmasq) |
| **Внешние DNS** | `8.8.8.8`, `1.1.1.1` |

---

### 1.1. Установка пакетов
```bash
apt-get update
apt-get install dnsmasq dhcp-server iptables openssh-server
```

### 1.2. Настройка сетевого интерфейса (LAN)
**Временно (до перезагрузки):**
```bash
ip addr add 172.16.1.1/26 dev eth0
ip link set eth0 up
sysctl -w net.ipv4.ip_forward=1
```

**Постоянно (переживет перезагрузку):**
```bash
mkdir -p /etc/net/ifaces/eth0

cat > /etc/net/ifaces/eth0/options << 'EOF'
TYPE=eth
BOOTPROTO=static
CONFIG_WIRELESS=no
EOF

cat > /etc/net/ifaces/eth0/ipv4address << 'EOF'
172.16.1.1/26
EOF
```

### 1.3. Включение IP-форвардинга (постоянно)
```bash
cat > /etc/sysctl.d/99-forward.conf << 'EOF'
net.ipv4.ip_forward = 1
EOF
sysctl --system
```

### 1.4. Настройка DNS (dnsmasq)
*DHCP в dnsmasq отключен, так как мы используем отдельный dhcpd.*
```bash
cat > /etc/dnsmasq.conf << 'EOF'
interface=eth0
bind-interfaces
no-dhcp-interface=eth0
server=8.8.8.8
server=1.1.1.1
cache-size=1000
EOF
systemctl enable --now dnsmasq
```

### 1.5. Настройка DHCP-сервера (ISC dhcpd)
```bash
cat > /etc/dhcp/dhcpd.conf << 'EOF'
authoritative;
log-facility local7;

subnet 172.16.1.0 netmask 255.255.255.192 {
    range 172.16.1.2 172.16.1.14;
    option routers 172.16.1.1;
    option subnet-mask 255.255.255.192;
    option broadcast-address 172.16.1.63;
    option domain-name-servers 172.16.1.1;
    default-lease-time 86400;
    max-lease-time 172800;
}
EOF

cat > /etc/sysconfig/dhcpd << 'EOF'
DHCPDARGS="eth0"
EOF

systemctl enable --now dhcpd
```

### 1.6. Настройка NAT (iptables)
```bash
# Маскарад для LAN → WAN
iptables -t nat -A POSTROUTING -s 172.16.1.0/26 -o eth1 -j MASQUERADE

# Разрешаем форвардинг LAN → WAN и обратно
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth0 -j ACCEPT

# Сохранение правил
iptables-save > /etc/sysconfig/iptables
systemctl enable iptables 2>/dev/null || true
```

### 1.7. Настройка SSH (OpenSSH)
```bash
cat >> /etc/openssh/sshd_config << 'EOF'

# === Custom Settings ===
Port 22
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

systemctl enable --now sshd
```

### ✅ Чек-лист сервера
- [ ] `ip addr show eth0` показывает `172.16.1.1/26`
- [ ] `cat /proc/sys/net/ipv4/ip_forward` выдает `1`
- [ ] `systemctl status dnsmasq dhcpd sshd` показывает `active (running)`
- [ ] `iptables -t nat -L -n -v` содержит правило `MASQUERADE`

---
---

# ЧАСТЬ 2: Настройка маршрутизатора (Eltex ESR-200)
*Сценарий: Аппаратный роутер выступает в роли пограничного шлюза с функциями DHCP, DNS и NAT.*

## 🗺️ Схема: Клиент ↔ Роутер

```text
   [Клиентская сеть]          [Eltex ESR-200]              [Внешняя сеть / Интернет]
   172.16.1.2 - 172.16.1.14          │ (bridge 2: WAN)
            │                        │
            └─── (bridge 1: LAN) ────┤ IP: 172.16.1.1/26
                                     │ Роль: Шлюз, DNS, DHCP, NAT
```

## 📊 Параметры роутера
| Параметр | Значение |
|----------|----------|
| **LAN интерфейс** | `bridge 1` (физические порты добавлены через Web UI) |
| **WAN интерфейс** | `bridge 2` (физические порты добавлены через Web UI) |
| **IP-адрес шлюза** | `172.16.1.1/26` (на bridge 1) |
| **DHCP диапазон** | `172.16.1.2` – `172.16.1.14` |
| **DNS для клиентов**| `172.16.1.1` (или внешние 8.8.8.8 / 1.1.1.1) |

> ⚠️ **Важно:** Физические интерфейсы (eth/wan/lan) **не настраиваются** через CLI в этой инструкции. Они должны быть добавлены в состав мостов (`bridge 1` и `bridge 2`) заранее через Web-интерфейс.

---

### 2.1. Настройка глобального DNS
```text
configure
ip name-server 8.8.8.8
ip name-server 1.1.1.1
exit
```

### 2.2. Настройка LAN интерфейса (bridge 1)
```text
configure
interface bridge 1
ip address 172.16.1.1/26
ip firewall disable
exit
```

### 2.3. Настройка DHCP-сервера
```text
configure
ip dhcp server pool LAN_POOL
network 172.16.1.0/26
default-router 172.16.1.1
dns-server 172.16.1.1
ip address-range 172.16.1.2-172.16.1.14
enable
exit

interface bridge 1
ip dhcp server pool LAN_POOL
exit
exit
```

### 2.4. Настройка NAT (для WAN)
```text
configure
object-group network LAN
ip address-range 172.16.1.2-172.16.1.14
exit

nat source
ruleset SNAT
to interface bridge 2
rule 1
match source-address LAN
action source-nat interface
enable
exit
exit
exit
```

### 2.5. Отключение Firewall на WAN (bridge 2)
```text
configure
interface bridge 2
ip firewall disable
exit
exit
```

### 2.6. Сохранение конфигурации
```text
save
```

### ✅ Чек-лист роутера
- [ ] DNS-серверы прописаны глобально
- [ ] `bridge 1` имеет IP `172.16.1.1/26`
- [ ] DHCP пул `LAN_POOL` активен и привязан к `bridge 1`
- [ ] NAT (`SNAT`) настроен на `bridge 2` с правильной object-group
- [ ] `ip firewall disable` применен к обоим мостам
- [ ] Конфигурация сохранена командой `save`

---
## 📝 Общие полезные команды для проверки

**На сервере (ALT JeOS):**
```bash
# Проверка выдачи DHCP
cat /var/lib/dhcp/dhcpd.leases

# Проверка разрешения имен
nslookup google.com 127.0.0.1

# Проверка открытых портов
ss -tlnp | grep -E '22|53|67'
```

**На роутере (Eltex ESR-200):**
```text
show ip interface brief
show ip dhcp server pool
show nat source ruleset SNAT
```

---
**Дата создания документа**: 22.09.2026
**Последнее обновление**: 22 сентября 2026  
**Совместимость**: Eltex ESR-200, ALT JeOS p11 (systemd)
