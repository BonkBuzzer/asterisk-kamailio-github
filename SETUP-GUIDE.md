# Complete WebRTC Setup Guide: Kamailio + Asterisk

This guide provides complete steps to recreate a production-ready WebRTC calling system with Kamailio as the front-end proxy and Asterisk as the PBX backend.

## Architecture Overview

```
Browser (WebRTC/WSS) → Kamailio (WSS:8443) → Asterisk (UDP:5060)
                          ↓
                    TLS Termination
                    Protocol Translation
                    Proxy/Router
```

**Key Features:**
- Direct browser calling (no softphone needed)
- WebRTC with Opus and ulaw codecs
- Scalable architecture (1 Kamailio front-end, N Asterisk servers)
- Secure connections with Let's Encrypt TLS certificates
- NAT traversal with ICE/STUN

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Server Setup](#server-setup)
3. [Install Dependencies](#install-dependencies)
4. [Install Asterisk](#install-asterisk)
5. [Install Kamailio](#install-kamailio)
6. [Configure TLS Certificates](#configure-tls-certificates)
7. [Configure Kamailio](#configure-kamailio)
8. [Configure Asterisk](#configure-asterisk)
9. [Configure Firewall](#configure-firewall)
10. [Start Services](#start-services)
11. [Testing](#testing)
12. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Infrastructure
- **Server**: AWS EC2 or any Linux VPS (Ubuntu 20.04/22.04 LTS recommended)
- **Instance Type**: t2.small or larger (2GB RAM minimum)
- **Storage**: 10GB minimum
- **Domain**: A valid domain name (e.g., sip.sumthing.space)
- **DNS**: A record pointing your domain to server's public IP

### Required Ports (AWS Security Group)
- **22** (TCP) - SSH
- **5060** (UDP) - SIP (Asterisk)
- **5062** (UDP) - SIP (Kamailio internal)
- **8088** (TCP) - WebSocket (Asterisk)
- **8443** (TCP) - WebSocket Secure (Kamailio - public facing)
- **10000-20000** (UDP) - RTP Media
- **80** (TCP) - HTTP (for Let's Encrypt)
- **443** (TCP) - HTTPS (optional)

---

## Server Setup

### 1. Connect to Server

```bash
ssh -i "your-key.pem" ubuntu@your-server-ip
```

### 2. Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### 3. Set Hostname

```bash
sudo hostnamectl set-hostname sip-server
```

### 4. Create Working Directory

```bash
mkdir -p ~/sip-setup
cd ~/sip-setup
```

---

## Install Dependencies

### 1. Install Build Tools

```bash
sudo apt install -y build-essential git curl wget
```

### 2. Install Asterisk Dependencies

```bash
sudo apt install -y \
  libssl-dev libxml2-dev libncurses5-dev uuid-dev \
  libsqlite3-dev libjansson-dev libedit-dev \
  libsrtp2-dev
```

### 3. Install PJSIP (Required for Asterisk WebRTC)

```bash
cd ~/sip-setup
wget https://github.com/pjsip/pjproject/archive/refs/tags/2.13.tar.gz
tar xvf 2.13.tar.gz
cd pjproject-2.13

./configure \
  --prefix=/usr \
  --libdir=/usr/lib/x86_64-linux-gnu \
  --enable-shared \
  --disable-sound \
  --disable-resample \
  --disable-video \
  --disable-opencore-amr \
  CFLAGS='-O2 -DNDEBUG'

make dep
sudo make install
sudo ldconfig
```

### 4. Install Kamailio Dependencies

```bash
sudo apt install -y \
  gcc flex bison make libssl-dev libcurl4-openssl-dev \
  libxml2-dev libpcre3-dev default-libmysqlclient-dev
```

### 5. Install Let's Encrypt (Certbot)

```bash
sudo apt install -y certbot
```

---

## Install Asterisk

### 1. Download Asterisk 18

```bash
cd ~/sip-setup
wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-18-current.tar.gz
tar xvf asterisk-18-current.tar.gz
cd asterisk-18.*
```

### 2. Install Prerequisites Script

```bash
sudo contrib/scripts/install_prereq install
```

### 3. Configure Build

```bash
./configure --with-pjproject-bundled --with-jansson-bundled
```

### 4. Select Modules

```bash
make menuselect
```

**Required Modules (use arrow keys and Enter to select):**

Navigate to each category and ensure these are selected (marked with [*]):

**Channel Drivers:**
- `chan_pjsip` (SIP channel driver)

**Resource Modules:**
- All `res_pjsip*` modules
- `res_srtp` (Secure RTP)
- `res_rtp_asterisk` (RTP stack)

**Codec Translators:**
- `codec_opus` (if available)
- `codec_ulaw`
- `codec_alaw`
- `codec_gsm`
- `codec_g722`

**Core Sound Packages:**
- `CORE-SOUNDS-EN-GSM` (English prompts)

Press 'x' to exit and save.

### 5. Compile and Install

```bash
make -j$(nproc)
sudo make install
sudo make samples
sudo make config
sudo ldconfig
```

### 6. Create Asterisk User

```bash
sudo groupadd asterisk
sudo useradd -r -d /var/lib/asterisk -g asterisk asterisk
sudo chown -R asterisk:asterisk /var/lib/asterisk
sudo chown -R asterisk:asterisk /var/spool/asterisk
sudo chown -R asterisk:asterisk /var/log/asterisk
sudo chown -R asterisk:asterisk /var/run/asterisk
sudo chown -R asterisk:asterisk /etc/asterisk
```

### 7. Configure Asterisk Service

```bash
sudo systemctl enable asterisk
```

---

## Install Kamailio

### 1. Add Kamailio Repository

```bash
sudo apt install -y gnupg2
wget -O- https://deb.kamailio.org/kamailiodebkey.gpg | sudo apt-key add -
echo "deb http://deb.kamailio.org/kamailio55 focal main" | sudo tee /etc/apt/sources.list.d/kamailio.list
sudo apt update
```

### 2. Install Kamailio

```bash
sudo apt install -y kamailio kamailio-websocket-modules kamailio-tls-modules
```

### 3. Enable Kamailio Service

```bash
sudo systemctl enable kamailio
```

---

## Configure TLS Certificates

### 1. Stop Any Web Server (if running)

```bash
sudo systemctl stop apache2 nginx 2>/dev/null || true
```

### 2. Obtain Let's Encrypt Certificate

Replace `sip.sumthing.space` with your domain:

```bash
sudo certbot certonly --standalone -d sip.sumthing.space --email your@email.com --agree-tos --non-interactive
```

### 3. Create Kamailio TLS Configuration

```bash
sudo mkdir -p /etc/kamailio
sudo bash -c 'cat > /etc/kamailio/tls.cfg' << 'EOF'
[server:default]
method = TLSv1.2+
verify_certificate = no
require_certificate = no
private_key = /etc/letsencrypt/live/sip.sumthing.space/privkey.pem
certificate = /etc/letsencrypt/live/sip.sumthing.space/fullchain.pem
EOF
```

### 4. Set Permissions

```bash
sudo chown kamailio:kamailio /etc/kamailio/tls.cfg
sudo chmod 640 /etc/kamailio/tls.cfg
```

### 5. Add Kamailio to SSL-Cert Group

```bash
sudo usermod -aG ssl-cert kamailio
sudo chown -R root:ssl-cert /etc/letsencrypt/live
sudo chown -R root:ssl-cert /etc/letsencrypt/archive
sudo chmod -R 750 /etc/letsencrypt/live
sudo chmod -R 750 /etc/letsencrypt/archive
```

### 6. Setup Certificate Auto-Renewal

```bash
echo "0 0 1 * * root certbot renew --quiet && systemctl restart kamailio" | sudo tee -a /etc/crontab
```

---

## Configure Kamailio

### 1. Backup Default Configuration

```bash
sudo cp /etc/kamailio/kamailio.cfg /etc/kamailio/kamailio.cfg.original
```

### 2. Create Kamailio Configuration

```bash
sudo bash -c 'cat > /etc/kamailio/kamailio.cfg' << 'EOF'
#!KAMAILIO
debug=2
enable_tls=yes
tcp_accept_no_cl=yes
tcp_connection_lifetime=3605

listen=udp:0.0.0.0:5062 advertise 127.0.0.1:5062
listen=tcp:0.0.0.0:5062
listen=tls:0.0.0.0:8443

loadmodule "sl.so"
loadmodule "tm.so"
loadmodule "rr.so"
loadmodule "pv.so"
loadmodule "maxfwd.so"
loadmodule "textops.so"
loadmodule "xlog.so"
loadmodule "tls.so"
loadmodule "xhttp.so"
loadmodule "websocket.so"
loadmodule "nathelper.so"
loadmodule "siputils.so"
loadmodule "sanity.so"
loadmodule "path.so"

modparam("tls", "config", "/etc/kamailio/tls.cfg")
modparam("websocket", "keepalive_mechanism", 1)
modparam("websocket", "keepalive_timeout", 30)
modparam("websocket", "keepalive_interval", 10)
modparam("websocket", "cors_mode", 2)
modparam("websocket", "sub_protocols", 1)
modparam("nathelper", "ping_nated_only", 1)
modparam("nathelper", "sipping_bflag", 7)

event_route[xhttp:request] {
    xlog("L_INFO", "HTTP Request: $rm $hu from $si:$sp\n");

    if ($Rp != 8443) {
        xhttp_reply("403", "Forbidden", "text/plain", "Wrong port");
        exit;
    }

    if ($hu =~ "^/ws") {
        xlog("L_INFO", "WebSocket handshake\n");

        if (ws_handle_handshake()) {
            xlog("L_INFO", "WebSocket handshake SUCCESS\n");
            exit;
        } else {
            xlog("L_ERR", "WebSocket handshake FAILED\n");
            exit;
        }
    }

    xhttp_reply("200", "OK", "text/plain", "Kamailio");
    exit;
}

event_route[websocket:closed] {
    xlog("L_INFO", "WebSocket closed from $si:$sp\n");
}

request_route {
    xlog("L_INFO", "SIP Request: $rm $ru from $si:$sp proto=$proto\n");

    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483", "Too Many Hops");
        exit;
    }

    if (is_method("OPTIONS")) {
        sl_send_reply("200", "OK");
        exit;
    }

    # Handle requests from Asterisk back to WebSocket clients
    if ($si == "127.0.0.1" && proto == UDP) {
        xlog("L_INFO", "Request from Asterisk to $ru\n");

        if (is_method("INVITE|BYE|CANCEL|ACK|UPDATE|NOTIFY|INFO")) {
            record_route();
        }

        t_on_reply("REPLY_TO_ASTERISK");
        if (!t_relay()) {
            sl_reply_error();
        }
        exit;
    }

    # Handle WebSocket messages - forward to Asterisk UDP
    if (proto == WS || proto == WSS) {
        xlog("L_INFO", "WebSocket SIP - forwarding to Asterisk UDP:5060\n");

        force_rport();

        if (is_method("REGISTER")) {
            add_path();
        }

        if (is_method("INVITE|SUBSCRIBE")) {
            record_route();
        }

        $du = "sip:127.0.0.1:5060;transport=udp";

        t_on_reply("REPLY_FROM_ASTERISK");
        if (!t_relay()) {
            sl_reply_error();
        }
        exit;
    }

    sl_send_reply("404", "Not Found");
    exit;
}

onreply_route[REPLY_FROM_ASTERISK] {
    xlog("L_INFO", "Reply from Asterisk: $rs $rr\n");
}

onreply_route[REPLY_TO_ASTERISK] {
    xlog("L_INFO", "Reply to Asterisk: $rs $rr\n");
}
EOF
```

### 3. Check Configuration

```bash
sudo kamailio -c
```

Should output: `config file ok, exiting...`

---

## Configure Asterisk

### 1. Configure PJSIP (SIP Stack)

```bash
sudo bash -c 'cat > /etc/asterisk/pjsip.conf' << 'EOF'
[global]
local_net=172.31.0.0/16
local_net=127.0.0.0/8
type=global

[transport-ws]
type=transport
protocol=ws
bind=0.0.0.0:8088
external_media_address=YOUR_PUBLIC_IP
external_signaling_address=YOUR_PUBLIC_IP

[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

; User 1001
[1001]
type=endpoint
transport=transport-ws
context=default
disallow=all
allow=ulaw,opus
auth=1001
aors=1001
webrtc=yes
dtls_auto_generate_cert=yes
media_encryption=dtls
ice_support=yes
rtcp_mux=yes
use_avpf=yes

[1001]
type=auth
auth_type=userpass
username=1001
password=1234

[1001]
type=aor
max_contacts=1
support_path=yes

; User 1002
[1002]
type=endpoint
transport=transport-ws
context=default
disallow=all
allow=ulaw,opus
auth=1002
aors=1002
webrtc=yes
dtls_auto_generate_cert=yes
media_encryption=dtls
ice_support=yes
rtcp_mux=yes
use_avpf=yes

[1002]
type=auth
auth_type=userpass
username=1002
password=1234

[1002]
type=aor
max_contacts=1
support_path=yes
EOF
```

**IMPORTANT:** Replace `YOUR_PUBLIC_IP` with your server's actual public IP address:

```bash
PUBLIC_IP=$(curl -s http://checkip.amazonaws.com)
sudo sed -i "s/YOUR_PUBLIC_IP/$PUBLIC_IP/g" /etc/asterisk/pjsip.conf
```

### 2. Configure RTP

```bash
sudo bash -c 'cat > /etc/asterisk/rtp.conf' << 'EOF'
[general]
rtpstart=10000
rtpend=20000
stunaddr=stun.l.google.com:19302
icesupport=yes
EOF
```

### 3. Configure Dialplan

```bash
sudo bash -c 'cat > /etc/asterisk/extensions.conf' << 'EOF'
[general]
static=yes
writeprotect=no
clearglobalvars=no

[globals]

[default]

; User Extensions
exten => 1001,1,NoOp(Calling 1001)
exten => 1001,n,Dial(PJSIP/1001,30)
exten => 1001,n,Hangup()

exten => 1002,1,NoOp(Calling 1002)
exten => 1002,n,Dial(PJSIP/1002,30)
exten => 1002,n,Hangup()

; Test Extension 6000 - IVR
exten => 6000,1,NoOp(Test Extension - IVR)
exten => 6000,n,Answer()
exten => 6000,n,Wait(1)
exten => 6000,n,SayDigits(12345)
exten => 6000,n,Wait(1)
exten => 6000,n,Playtones(ring)
exten => 6000,n,Wait(3)
exten => 6000,n,StopPlaytones()
exten => 6000,n,Hangup()
EOF
```

### 4. Set Permissions

```bash
sudo chown -R asterisk:asterisk /etc/asterisk
```

---

## Configure Firewall

### AWS Security Group
Configure in AWS Console with ports listed in Prerequisites section.

### UFW (Ubuntu Firewall)
If using UFW instead of AWS Security Groups:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 5060/udp
sudo ufw allow 5062/udp
sudo ufw allow 8088/tcp
sudo ufw allow 8443/tcp
sudo ufw allow 10000:20000/udp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

---

## Start Services

### 1. Start Kamailio

```bash
sudo systemctl start kamailio
sudo systemctl status kamailio
```

### 2. Start Asterisk

```bash
sudo /usr/sbin/asterisk
sleep 3
sudo asterisk -rx "core show version"
```

### 3. Verify Services

```bash
# Check Kamailio
sudo systemctl status kamailio

# Check Asterisk
sudo asterisk -rx "pjsip show endpoints"
sudo asterisk -rx "pjsip show transports"
sudo asterisk -rx "rtp show settings"
```

### 4. Check Listening Ports

```bash
sudo netstat -tulpn | grep -E "5060|5062|8088|8443"
```

Expected output:
```
tcp  0.0.0.0:8088   LISTEN   asterisk
tcp  0.0.0.0:8443   LISTEN   kamailio
udp  0.0.0.0:5060   asterisk
udp  0.0.0.0:5062   kamailio
```

---

## Testing

### 1. Test from Asterisk CLI

```bash
sudo asterisk -rvvv
```

Commands to test:
```
pjsip show endpoints
pjsip show transports
pjsip show contacts
rtp show settings
core show channels
```

### 2. Test WebSocket Connection

From your browser console:

```javascript
const ws = new WebSocket('wss://sip.sumthing.space:8443/ws');
ws.onopen = () => console.log('Connected!');
ws.onerror = (e) => console.error('Error:', e);
```

### 3. Test Registration

Use a WebRTC SIP client (like JsSIP) to register:

**Server**: `wss://sip.sumthing.space:8443/ws`
**Username**: `1001`
**Password**: `1234`

### 4. Test Calls

After registration:
- Call extension `6000` - should hear digits and tones
- Call from `1001` to `1002` - should connect with audio

---

## Troubleshooting

### Check Kamailio Logs

```bash
sudo tail -f /var/log/syslog | grep kamailio
```

### Check Asterisk Logs

```bash
sudo tail -f /var/log/asterisk/messages
```

### Enable Detailed Logging

```bash
# Asterisk
sudo asterisk -rx "core set verbose 5"
sudo asterisk -rx "core set debug 5"
sudo asterisk -rx "pjsip set logger on"

# Kamailio
# Edit /etc/kamailio/kamailio.cfg and set debug=4
sudo systemctl restart kamailio
```

### Common Issues

**1. Registration Fails**
- Check TLS certificates: `sudo certbot certificates`
- Verify DNS points to correct IP
- Check firewall allows port 8443

**2. No Audio**
- Verify RTP ports 10000-20000 open in firewall
- Check STUN is working: `sudo asterisk -rx "rtp show settings"`
- Verify codecs match: `sudo asterisk -rx "pjsip show endpoints"`

**3. Calls Don't Connect Between Users**
- Check Path support: `sudo asterisk -rx "database show" | grep path`
- Should show `"path":"<sip:127.0.0.1:5062;lr>"`
- Verify both users registered: `sudo asterisk -rx "pjsip show contacts"`

**4. WebSocket Connection Fails**
- Test TLS: `openssl s_client -connect sip.sumthing.space:8443`
- Check Kamailio running: `sudo systemctl status kamailio`
- Verify port 8443 open: `sudo netstat -tulpn | grep 8443`

### Clear Registrations

```bash
sudo asterisk -rx "database deltree registrar"
sudo asterisk -rx "core restart now"
```

---

## Maintenance

### Update TLS Certificates

```bash
sudo certbot renew --force-renewal
sudo systemctl restart kamailio
```

### Backup Configuration

```bash
# Create backup directory
mkdir -p ~/backups/$(date +%Y%m%d)

# Backup Asterisk
sudo cp -r /etc/asterisk ~/backups/$(date +%Y%m%d)/

# Backup Kamailio
sudo cp -r /etc/kamailio ~/backups/$(date +%Y%m%d)/
```

### Monitor Resources

```bash
# Disk usage
df -h

# Check Asterisk logs size
du -sh /var/log/asterisk/

# Truncate large log files
sudo truncate -s 0 /var/log/asterisk/messages
```

### Add More Users

Edit `/etc/asterisk/pjsip.conf` and add:

```ini
[USERNAME]
type=endpoint
transport=transport-ws
context=default
disallow=all
allow=ulaw,opus
auth=USERNAME
aors=USERNAME
webrtc=yes
dtls_auto_generate_cert=yes
media_encryption=dtls
ice_support=yes
rtcp_mux=yes
use_avpf=yes

[USERNAME]
type=auth
auth_type=userpass
username=USERNAME
password=PASSWORD

[USERNAME]
type=aor
max_contacts=1
support_path=yes
```

Then reload:
```bash
sudo asterisk -rx "pjsip reload"
```

---

## Scaling

### Adding Additional Asterisk Servers

1. Install Asterisk on new server following steps above
2. Keep Kamailio on one server as the front-end
3. Modify Kamailio routing logic to distribute calls across multiple Asterisk servers
4. Use a database (MySQL/PostgreSQL) for centralized user management

### Load Balancing

Consider using:
- **Dispatcher Module** in Kamailio for load balancing
- **Database Backend** for shared user registry
- **Separate Media Servers** for RTP processing

---

## Version Control

### Setup Git Repository

```bash
# On server
cd ~
mkdir sip-config
cd sip-config
git init

# Copy configs
sudo cp /etc/kamailio/kamailio.cfg kamailio.cfg
sudo cp /etc/kamailio/tls.cfg kamailio-tls.cfg
sudo cp /etc/asterisk/pjsip.conf asterisk-pjsip.conf
sudo cp /etc/asterisk/extensions.conf asterisk-extensions.conf
sudo cp /etc/asterisk/rtp.conf asterisk-rtp.conf

# Fix permissions
sudo chown ubuntu:ubuntu *.cfg *.conf

# Create README
cat > README.md << 'MDEOF'
# SIP Configuration Files

Configuration files for Kamailio + Asterisk WebRTC setup.

## Files
- kamailio.cfg - Main Kamailio configuration
- kamailio-tls.cfg - TLS certificate paths
- asterisk-pjsip.conf - PJSIP endpoints and transports
- asterisk-extensions.conf - Dialplan
- asterisk-rtp.conf - RTP settings
MDEOF

# Commit
git add .
git commit -m "Initial configuration"

# Push to GitHub (optional)
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git push -u origin main
```

---

## Support and Resources

### Documentation
- **Asterisk**: https://docs.asterisk.org/
- **Kamailio**: https://www.kamailio.org/docs/
- **PJSIP**: https://docs.pjsip.org/
- **WebRTC**: https://webrtc.org/

### Community
- Asterisk IRC: #asterisk on irc.freenode.net
- Kamailio Mailing List: sr-users@lists.kamailio.org

---

## Summary

You now have:
- ✅ Kamailio as WebRTC front-end (WSS:8443)
- ✅ Asterisk as PBX backend (UDP:5060)
- ✅ TLS encryption with Let's Encrypt
- ✅ Browser-based calling (no softphone needed)
- ✅ Working audio with ulaw and Opus codecs
- ✅ ICE/STUN for NAT traversal
- ✅ Path-based routing for WebSocket clients
- ✅ Test extension 6000 for verification
- ✅ Scalable architecture for growth

**Test by:**
1. Registering users 1001/1002 from browser
2. Calling extension 6000 to test audio
3. Calling between users 1001 ↔ 1002

Congratulations! Your WebRTC SIP system is ready for production use.
