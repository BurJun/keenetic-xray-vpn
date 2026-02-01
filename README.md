# Keenetic + Xray (VLESS Reality) как прозрачный VPN для всей сети

Эта инструкция описывает настройку Xray на роутере (Keenetic с Entware во внутренней памяти) как прозрачного VPN для всей LAN‑сети с использованием VLESS + Reality.

## 0. Требования

- Роутер Keenetic с установленным Entware во **внутренней памяти**.
- Доступ по SSH к роутеру (root).
- Рабочий VLESS Reality сервер 

# 1. Скачать и установить Xray (mips32le) в /opt/sbin

```sh
cd /tmp
wget "https://sourceforge.net/projects/xray-core.mirror/files/v25.10.15/Xray-linux-mips32le.zip/download" -O xray.zip
unzip xray.zip
chmod +x xray
./xray version
mv /tmp/xray /opt/sbin/xray
chmod +x /opt/sbin/xray
```

# 2. Создать директории для конфига и логов

```sh
mkdir -p /opt/etc/xray
mkdir -p /opt/var/log/xray
```

# 3. БАЗОВЫЙ конфиг Xray (VLESS Reality + SOCKS5)

```sh
cat > /opt/etc/xray/config.json << 'EOF'
{
  "log": {
    "access": "/opt/var/log/xray/access.log",
    "error": "/opt/var/log/xray/error.log",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "socks-in",
      "port": 1080,
      "listen": "0.0.0.0",
      "protocol": "socks",
      "settings": {
        "udp": true,
        "auth": "noauth"
      }
    }
  ],
  "outbounds": [
    {
      "tag": "vless-out",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "..............",
            "port": ....,
            "users": [
              {
                "id": "....................",
                "flow": "",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "serverName": "ya.ru",
          "fingerprint": "chrome",
          "publicKey": ".........................",
          "shortId": "45",
          "spiderX": "/"
        }
      }
    }
  ]
}
EOF
```
# 4. Тестовый запуск Xray и проверка

```sh
/opt/sbin/xray -config /opt/etc/xray/config.json &
sleep 2
ps | grep xray
curl --socks5 127.0.0.1:1080 http://ifconfig.me/ip
```
# 5. Init-скрипт для автозапуска Xray

```sh
cat > /opt/etc/init.d/S24xray << 'EOF'
#!/bin/sh

case "$1" in
  start)
    /opt/sbin/xray -config /opt/etc/xray/config.json &
    ;;
  stop)
    killall xray 2>/dev/null
    ;;
  restart)
    $0 stop
    sleep 1
    $0 start
    ;;
  *)
    echo "Usage: $0 {start|stop|restart}"
    exit 1
    ;;
esac

exit 0
EOF

chmod +x /opt/etc/init.d/S24xray
/opt/etc/init.d/S24xray restart
sleep 2
ps | grep xray
```

# 6. ОБНОВЛЁННЫЙ конфиг Xray — прозрачный VPN на всю сеть

```
cat > /opt/etc/xray/config.json << 'EOF'
{
  "log": {
    "access": "/opt/var/log/xray/access.log",
    "error": "/opt/var/log/xray/error.log",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "socks-in",
      "port": 1080,
      "listen": "0.0.0.0",
      "protocol": "socks",
      "settings": {
        "udp": true,
        "auth": "noauth"
      }
    },
    {
      "tag": "tproxy-in",
      "port": 12345,
      "listen": "0.0.0.0",
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp",
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      }
    }
  ],
  "outbounds": [
    {
      "tag": "vless-out",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "............",
            "port": ....,
            "users": [
              {
                "id": "............................",
                "flow": "",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "serverName": "ya.ru",
          "fingerprint": "chrome",
          "publicKey": "..........................",
          "shortId": "45",
          "spiderX": "/"
        }
      }
    }
  ]
}
EOF

/opt/etc/init.d/S24xray restart
sleep 2
ps | grep xray
```
# 7. iptables: прозрачный редирект всего TCP-трафика с LAN в Xray

```sh
# создать/очистить цепочку XRAY

iptables -t nat -N XRAY 2>/dev/null || iptables -t nat -F XRAY

# не трогаем локальные/служебные сети

iptables -t nat -A XRAY -d 0.0.0.0/8 -j RETURN
iptables -t nat -A XRAY -d 10.0.0.0/8 -j RETURN
iptables -t nat -A XRAY -d 127.0.0.0/8 -j RETURN
iptables -t nat -A XRAY -d 169.254.0.0/16 -j RETURN
iptables -t nat -A XRAY -d 172.16.0.0/12 -j RETURN
iptables -t nat -A XRAY -d 192.168.0.0/16 -j RETURN
iptables -t nat -A XRAY -d 224.0.0.0/4 -j RETURN
iptables -t nat -A XRAY -d 240.0.0.0/4 -j RETURN

# весь остальной TCP → dokodemo-door Xray (порт 12345)
iptables -t nat -A XRAY -p tcp -j REDIRECT --to-ports 12345

# применяем к трафику из обеих LAN-сетей (br0 и br1)
iptables -t nat -A PREROUTING -i br0 -p tcp -j XRAY
iptables -t nat -A PREROUTING -i br1 -p tcp -j XRAY

# сохранить правила NAT
iptables-save > /opt/etc/iptables.rules
```
