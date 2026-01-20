# Complete WebRTC Peer-to-Peer Setup Guide
**Kamailio + Asterisk WebRTC System with Direct User-to-User Calling**

**Version**: 2.0 (Peer-to-Peer Architecture)
**Date**: January 19, 2026
**Status**: ✅ Production Ready - Fully Working

---

## 📋 Table of Contents

1. [What This Guide Provides](#what-this-guide-provides)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Step-by-Step Installation](#step-by-step-installation)
5. [Configuration Files](#configuration-files)
6. [Call Flow Diagrams](#call-flow-diagrams)
7. [Testing Procedures](#testing-procedures)
8. [Troubleshooting](#troubleshooting)
9. [Maintenance](#maintenance)

---

## 🎯 What This Guide Provides

This guide gives you **complete, tested steps** to recreate a production WebRTC calling system where:

- ✅ **Users call each other directly** (peer-to-peer through Kamailio)
- ✅ **Service extensions** (IVR, voicemail) route through Asterisk
- ✅ **DTLS/SRTP encryption** works end-to-end
- ✅ **No "480 Temporarily Unavailable"** errors
- ✅ **Full bidirectional audio** working

### Key Innovation: Intelligent Routing

```
User-to-User:       Browser ←→ Kamailio ←→ Browser (Direct P2P)
Service Extensions: Browser ←→ Kamailio ←→ Asterisk (Features)
```

Most guides route ALL calls through Asterisk, which **breaks WebRTC** because Asterisk (acting as B2BUA) strips DTLS fingerprints from SDP. This guide solves that with intelligent routing.

---

## 🏗️ Architecture Overview

### System Diagram

```
┌──────────────────────────────────────────────────────────┐
│                   Internet / Cloud                        │
│              Domain: sip.sumthing.space                   │
└──────────────────────────────────────────────────────────┘
                            │
                            │ TLS/WSS Port 8443
                            │ (Public Facing)
                            ▼
                  ┌──────────────────┐
                  │  Kamailio 5.5    │  ◄─── WebSocket Server
                  │  SIP Proxy       │       NAT Traversal
                  │  Port 5062/8443  │       Location DB
                  │                  │       Smart Routing
                  └──────────────────┘
                       │         │
      ┌────────────────┘         └────────────┐
      │ UDP:5060                               │ WSS (Direct)
      │ (Services)                             │ (Peer-to-Peer)
      ▼                                        ▼
┌──────────────┐                     ┌──────────────────┐
│ Asterisk 18  │                     │ WebRTC Clients   │
│ PBX Backend  │                     │                  │
│              │                     │ User 1001 ◄────► │
│ - IVR (6000) │                     │ User 1002        │
│ - Voicemail  │                     │                  │
│ - Features   │                     │ Browser-based    │
└──────────────┘                     └──────────────────┘
```

### Component Roles

| Component | Purpose | Handles |
|-----------|---------|---------|
| **Kamailio** | Front-end proxy | WebSocket, TLS, routing decisions, location DB |
| **Asterisk** | Backend PBX | Authentication, service extensions (IVR) |
| **Browsers** | WebRTC clients | Media, DTLS-SRTP encryption |

---

## 📦 Prerequisites

### Server Requirements

- **OS**: Ubuntu 20.04 or 22.04 LTS
- **RAM**: 2GB minimum (4GB recommended)
- **CPU**: 2+ cores recommended
- **Storage**: 20GB minimum
- **Network**: Public IP address

### Required Ports (Firewall/Security Group)

| Port(s) | Protocol | Service | Public Access? |
|---------|----------|---------|----------------|
| 22 | TCP | SSH | Yes |
| 80 | TCP | HTTP (Let's Encrypt) | Yes (temporarily) |
| 443 | TCP | HTTPS | Optional |
| 5060 | UDP | SIP (Asterisk) | **NO** (localhost only) |
| 5062 | UDP | SIP (Kamailio internal) | No |
| **8443** | **TCP** | **WebSocket Secure (WSS)** | **YES** (main entry point) |
| 10000-20000 | UDP | RTP Media | Yes |

### DNS & Domain

- Domain name (e.g., `sip.sumthing.space`)
- DNS A record: `sip.sumthing.space` → `YOUR_PUBLIC_IP`
- Valid TLS certificate (Let's Encrypt - free)

---

## 🚀 Step-by-Step Installation

### Step 1: Initial Server Setup

```bash
# Connect to server
ssh -i "your-key.pem" ubuntu@YOUR_SERVER_IP

# Update system
sudo apt update && sudo apt upgrade -y

# Install basic tools
sudo apt install -y git curl wget net-tools

# Reboot if kernel updated
sudo reboot
```

### Step 2: Install Kamailio

```bash
# Add Kamailio repository
sudo apt install -y gnupg2
wget -O- http://deb.kamailio.org/kamailiodebkey.gpg | sudo apt-key add -

# Add repo for Ubuntu 20.04 (focal) or 22.04 (jammy)
echo "deb http://deb.kamailio.org/kamailio55 focal main" | \
  sudo tee /etc/apt/sources.list.d/kamailio.list

# Update and install
sudo apt update
sudo apt install -y kamailio \
  kamailio-tls-modules \
  kamailio-websocket-modules \
  kamailio-utils-modules

# Enable service
sudo systemctl enable kamailio
```

### Step 3: Install Asterisk

```bash
# Install dependencies
sudo apt install -y build-essential libssl-dev \
  libncurses5-dev libnewt-dev libxml2-dev \
  linux-headers-$(uname -r) libsqlite3-dev \
  uuid-dev libjansson-dev

# Download Asterisk 18
cd /usr/src
sudo wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-18-current.tar.gz
sudo tar xvf asterisk-18-current.tar.gz
cd asterisk-18.*

# Run prerequisite install script
sudo contrib/scripts/install_prereq install

# Configure with bundled PJSIP
sudo ./configure --with-pjproject-bundled

# Optional: Select modules (press Enter to continue with defaults)
sudo make menuselect

# Compile (this takes 10-20 minutes)
sudo make -j$(nproc)

# Install
sudo make install
sudo make samples
sudo make config
sudo ldconfig

# Create asterisk user
sudo groupadd asterisk
sudo useradd -r -d /var/lib/asterisk -g asterisk asterisk
sudo chown -R asterisk:asterisk /var/lib/asterisk
sudo chown -R asterisk:asterisk /var/spool/asterisk
sudo chown -R asterisk:asterisk /var/log/asterisk
sudo chown -R asterisk:asterisk /var/run/asterisk
sudo chown -R asterisk:asterisk /etc/asterisk

# Enable and start
sudo systemctl enable asterisk
sudo systemctl start asterisk
```

### Step 4: Setup TLS Certificates

```bash
# Install certbot
sudo apt install -y certbot

# Stop any web server temporarily
sudo systemctl stop apache2 nginx 2>/dev/null || true

# Get certificate (replace sip.sumthing.space with YOUR domain)
sudo certbot certonly --standalone \
  -d sip.sumthing.space \
  --email your@email.com \
  --agree-tos \
  --non-interactive

# Give Kamailio access to certificates
sudo usermod -aG ssl-cert kamailio
sudo chown -R root:ssl-cert /etc/letsencrypt/live
sudo chown -R root:ssl-cert /etc/letsencrypt/archive
sudo chmod -R 750 /etc/letsencrypt/live
sudo chmod -R 750 /etc/letsencrypt/archive

# Setup auto-renewal
echo "0 0 1 * * root certbot renew --quiet && systemctl restart kamailio" | \
  sudo tee -a /etc/crontab
```

---

## ⚙️ Configuration Files

### 1. Kamailio Main Config

**File**: `/etc/kamailio/kamailio.cfg`

```bash
sudo tee /etc/kamailio/kamailio.cfg > /dev/null << 'EOF'
#!KAMAILIO

####### Global Parameters #########
debug=2
log_stderror=no
log_facility=LOG_LOCAL0
fork=yes
children=8

listen=udp:0.0.0.0:5062 advertise 127.0.0.1:5062
listen=tcp:0.0.0.0:5062
listen=tls:0.0.0.0:8443

####### Modules Section ########
loadmodule "tm.so"
loadmodule "sl.so"
loadmodule "rr.so"
loadmodule "pv.so"
loadmodule "maxfwd.so"
loadmodule "textops.so"
loadmodule "xlog.so"
loadmodule "tls.so"
loadmodule "xhttp.so"
loadmodule "websocket.so"
loadmodule "nathelper.so"
modparam("nathelper", "received_avp", "$avp(RECEIVED)")
loadmodule "siputils.so"
loadmodule "sanity.so"
loadmodule "path.so"
loadmodule "usrloc.so"
modparam("usrloc", "db_mode", 0)
modparam("usrloc", "nat_bflag", 6)
loadmodule "htable.so"
loadmodule "registrar.so"
modparam("registrar", "received_avp", "$avp(RECEIVED)")

####### Module Parameters ########
modparam("tls", "config", "/etc/kamailio/tls.cfg")
modparam("websocket", "keepalive_mechanism", 1)
modparam("websocket", "keepalive_timeout", 30)

####### Routing Logic ########

event_route[xhttp:request] {
    if ($hu =~ "^/ws" && $Rp == 8443) {
        if (ws_handle_handshake()) {
            xlog("L_INFO", "WebSocket handshake SUCCESS\n");
            exit;
        }
    }
    xhttp_reply("404", "Not Found", "text/plain", "Not found");
}

event_route[websocket:closed] {
    xlog("L_INFO", "WebSocket closed from $si:$sp\n");
}

request_route {
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483", "Too Many Hops");
        exit;
    }

    # Handle requests from Asterisk back to WebSocket clients
    if ($si == "127.0.0.1" && proto == UDP) {
        xlog("L_INFO", "Request from Asterisk to $ru\n");

        # Normalize domain for location lookup
        # Change sip:user@127.0.0.1:5062 to sip:user@sip.sumthing.space
        if ($rU) {
            $ru = "sip:" + $rU + "@sip.sumthing.space";
            xlog("L_INFO", "Normalized Request-URI to: $ru\n");
        }

        if (is_method("INVITE|BYE|CANCEL|ACK|UPDATE|NOTIFY|INFO")) {
            record_route();
        }

        t_on_reply("REPLY_TO_ASTERISK");

        # Lookup user location for routing to WebSocket
        xlog("L_INFO", "Looking up location for: $ru - from: $si\n");
        if (!lookup("location")) {
            xlog("L_INFO", "Location NOT found for: $ru\n");
            sl_send_reply("404", "Not Found");
            exit;
        }
        xlog("L_INFO", "Location found! Routing to: $du\n");

        handle_ruri_alias();

        # Add alias to Contact for WebSocket response routing
        if ($du =~ "transport=ws") {
            set_contact_alias();
            xlog("L_INFO", "Added contact alias for WebSocket routing\n");
        }

        if (!t_relay()) {
            sl_reply_error();
        }
        exit;
    }

    # Handle WebSocket messages
    if (proto == WS || proto == WSS) {
        force_rport();

        # Check for peer-to-peer WebRTC calls (user-to-user)
        if (is_method("INVITE")) {
            # Save original request URI before lookup modifies it
            $var(original_ruri) = $ru;

            # Try to find destination user in location database
            if (lookup("location")) {
                # Found - this is a peer-to-peer call, route directly without Asterisk
                xlog("L_INFO", "WebRTC peer-to-peer call: $fu -> $var(original_ruri) (routing to: $du)\n");

                record_route();
                handle_ruri_alias();

                # Add contact alias for response routing
                if ($du =~ "transport=ws") {
                    set_contact_alias();
                    xlog("L_INFO", "Added contact alias for peer-to-peer call\n");
                }

                t_on_reply("REPLY_TO_ASTERISK");

                if (!t_relay()) {
                    xlog("L_ERR", "Failed to relay peer-to-peer call\n");
                    sl_reply_error();
                }
                exit;
            } else {
                # Not found in location - restore URI and forward to Asterisk for services (IVR, etc.)
                $ru = $var(original_ruri);
                xlog("L_INFO", "Destination not registered locally, forwarding to Asterisk: $ru\n");
            }
        }

        # Forward to Asterisk for REGISTER, SUBSCRIBE, and service extensions
        xlog("L_INFO", "WebSocket SIP - forwarding to Asterisk UDP:5060\n");

        if (is_method("REGISTER")) {
            add_path_received();
            fix_nated_register();
            save("location", "0x04");
            xlog("L_INFO", "REGISTER from: $fu to AOR: $tu\n");
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

    # Save location when registration succeeds
    if (is_method("REGISTER") && status =~ "200") {
        xlog("L_INFO", "Registration successful - saving location for: $fu\n");
        save("location");
    }
}

onreply_route[REPLY_TO_ASTERISK] {
    xlog("L_INFO", "Reply to Asterisk: $rs $rr\n");

    # Handle Contact alias in replies from WebSocket clients
    if (nat_uac_test("64")) {
        fix_nated_contact();
        xlog("L_INFO", "Fixed NAT contact in reply\n");
    }
}
EOF
```

### 2. Kamailio TLS Config

**File**: `/etc/kamailio/tls.cfg`

```bash
sudo tee /etc/kamailio/tls.cfg > /dev/null << 'EOF'
[server:default]
method = TLSv1.2+
verify_certificate = no
require_certificate = no
certificate = /etc/letsencrypt/live/sip.sumthing.space/fullchain.pem
private_key = /etc/letsencrypt/live/sip.sumthing.space/privkey.pem
EOF

# Fix permissions
sudo chown kamailio:kamailio /etc/kamailio/tls.cfg
sudo chmod 640 /etc/kamailio/tls.cfg
```

**IMPORTANT**: Replace `sip.sumthing.space` with YOUR domain!

### 3. Asterisk PJSIP Config

**File**: `/etc/asterisk/pjsip.conf`

```bash
# First, get your public IP
PUBLIC_IP=$(curl -s http://checkip.amazonaws.com)
echo "Your public IP: $PUBLIC_IP"

# Create config
sudo tee /etc/asterisk/pjsip.conf > /dev/null << EOF
[global]
max_forwards=70
user_agent=Asterisk PBX
default_realm=sip.sumthing.space

[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060
external_media_address=$PUBLIC_IP
external_signaling_address=$PUBLIC_IP

; User 1001
[1001]
type=endpoint
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
rtp_symmetric=yes
force_rport=yes
direct_media=no

[1001]
type=auth
auth_type=userpass
username=1001
password=1234

[1001]
type=aor
support_path=yes
max_contacts=1
remove_existing=yes

; User 1002
[1002]
type=endpoint
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
rtp_symmetric=yes
force_rport=yes
direct_media=no

[1002]
type=auth
auth_type=userpass
username=1002
password=1234

[1002]
type=aor
support_path=yes
max_contacts=1
remove_existing=yes

; Kamailio Trunk
[kamailio]
type=endpoint
context=default
disallow=all
allow=ulaw,opus
direct_media=no
from_domain=sip.sumthing.space
aors=kamailio
rtp_symmetric=yes
force_rport=yes
media_encryption_optimistic=yes

[kamailio]
type=aor
contact=sip:127.0.0.1:5062
EOF

# Set permissions
sudo chown asterisk:asterisk /etc/asterisk/pjsip.conf
```

### 4. Asterisk Dialplan

**File**: `/etc/asterisk/extensions.conf`

```bash
sudo tee /etc/asterisk/extensions.conf > /dev/null << 'EOF'
[default]

; Test IVR Extension
exten => 6000,1,NoOp(Test Extension - IVR)
exten => 6000,n,Answer()
exten => 6000,n,Wait(1)
exten => 6000,n,SayDigits(12345)
exten => 6000,n,Wait(1)
exten => 6000,n,Playtones(ring)
exten => 6000,n,Wait(3)
exten => 6000,n,StopPlaytones()
exten => 6000,n,Hangup()

; User Extensions
exten => 1001,1,NoOp(Calling 1001)
exten => 1001,n,Dial(PJSIP/1001@kamailio,30)
exten => 1001,n,Hangup()

exten => 1002,1,NoOp(Calling 1002)
exten => 1002,n,Dial(PJSIP/1002@kamailio,30)
exten => 1002,n,Hangup()
EOF

sudo chown asterisk:asterisk /etc/asterisk/extensions.conf
```

### 5. Asterisk RTP Config

**File**: `/etc/asterisk/rtp.conf`

```bash
sudo tee /etc/asterisk/rtp.conf > /dev/null << 'EOF'
[general]
rtpstart=10000
rtpend=20000
stunaddr=stun.l.google.com:19302
icesupport=yes
EOF

sudo chown asterisk:asterisk /etc/asterisk/rtp.conf
```

### 6. Start Services

```bash
# Test configs
sudo kamailio -c
if [ $? -eq 0 ]; then
    echo "✅ Kamailio config OK"
else
    echo "❌ Kamailio config has errors!"
    exit 1
fi

# Start Kamailio
sudo systemctl restart kamailio
sudo systemctl status kamailio --no-pager

# Reload Asterisk
sudo asterisk -rx "core reload"
sudo asterisk -rx "pjsip reload"
sudo asterisk -rx "dialplan reload"

# Verify
sudo asterisk -rx "pjsip show endpoints"
```

---

## 📞 Call Flow Diagrams

### Diagram 1: User Registration

```
WebRTC Client          Kamailio            Asterisk
     |                     |                   |
     | WSS Connect         |                   |
     | wss://domain:8443   |                   |
     |-------------------->|                   |
     |  WebSocket OK       |                   |
     |<--------------------|                   |
     |                     |                   |
     | REGISTER            |                   |
     | sip:1001@domain     |                   |
     |-------------------->|                   |
     |                     | add_path_received()|
     |                     | fix_nated_register()|
     |                     |                   |
     |                     | REGISTER (w/Path) |
     |                     |------------------>|
     |                     |                   |
     |                     | 401 Unauthorized  |
     | 401 Unauthorized    |<------------------|
     |<--------------------|                   |
     |                     |                   |
     | REGISTER (w/ auth)  |                   |
     |-------------------->|                   |
     |                     | REGISTER          |
     |                     |------------------>|
     |                     |                   |
     |                     |    200 OK         |
     |     200 OK          |<------------------|
     |<--------------------|                   |
     |                     | save("location")  |
     |                     | ✓ Saved in usrloc |
```

### Diagram 2: Peer-to-Peer User Call (✅ NEW - Fixes 480 Error!)

```
User 1001              Kamailio           User 1002
    |                      |                  |
    | INVITE 1002          |                  |
    |--------------------->|                  |
    |                      | lookup(location) |
    |                      | ✓ Found!         |
    |                      |                  |
    |                      | INVITE (WebRTC   |
    |                      | SDP with DTLS)   |
    |                      |----------------->|
    |                      |                  |
    |  100 Trying          |   100 Trying     |
    |<---------------------|<-----------------|
    |                      |                  |
    |  180 Ringing         |   180 Ringing    |
    |<---------------------|<-----------------|
    |                      |                  |
    |  200 OK              |   200 OK         |
    |  (WebRTC SDP         |   (WebRTC SDP    |
    |   with DTLS)         |    with DTLS)    |
    |<---------------------|<-----------------|
    |                      |                  |
    |      ACK             |       ACK        |
    |--------------------->|----------------->|
    |                      |                  |
    |<════════════════════════════════════════>|
    |     RTP Media (DTLS-SRTP encrypted)     |
    |     Direct peer-to-peer                 |
    |                                          |

✅ NO Asterisk in the media path
✅ DTLS fingerprints preserved
✅ Perfect WebRTC negotiation
✅ Full bidirectional audio
```

### Diagram 3: Service Extension Call (via Asterisk)

```
User 1001              Kamailio          Asterisk
    |                      |                 |
    | INVITE 6000          |                 |
    |--------------------->|                 |
    |                      | lookup(location)|
    |                      | ✗ Not found     |
    |                      |                 |
    |                      | INVITE 6000     |
    |                      |---------------->|
    |                      |                 |
    |  100 Trying          | 100 Trying      |
    |<---------------------|<----------------|
    |                      |                 |
    |  200 OK              | 200 OK          |
    |<---------------------|<----------------|
    |                      |                 |
    |      ACK             |     ACK         |
    |--------------------->|---------------->|
    |                      |                 |
    |<═══════════════════════════════════════|
    |    RTP Media via Asterisk              |
    |    (IVR plays digits)                  |
```

---

## 🧪 Testing Procedures

### Step 1: Verify Services Running

```bash
# Check Kamailio
sudo systemctl status kamailio | grep Active

# Check Asterisk
sudo asterisk -rx "core show version"

# Check listening ports
sudo netstat -tlnp | grep -E '5060|8443'
```

Expected output:
```
tcp  0.0.0.0:8443   LISTEN   kamailio
udp  0.0.0.0:5060   asterisk
```

### Step 2: Test WebSocket Connection

Open browser console (F12):

```javascript
const ws = new WebSocket('wss://sip.sumthing.space:8443/ws');
ws.onopen = () => console.log('✅ Connected!');
ws.onerror = (e) => console.error('❌ Error:', e);
```

Should see: `✅ Connected!`

### Step 3: Register Users

Use a WebRTC SIP client:
- **JsSIP Demo**: https://tryit.jssip.net/
- **InnovateAsterisk**: https://www.innovateasterisk.com/phone/

**User 1001 Settings:**
```
WebSocket URL: wss://sip.sumthing.space:8443/ws
SIP URI: sip:1001@sip.sumthing.space
Password: 1234
Display Name: User 1001
```

**User 1002 Settings:**
```
WebSocket URL: wss://sip.sumthing.space:8443/ws
SIP URI: sip:1002@sip.sumthing.space
Password: 1234
Display Name: User 1002
```

### Step 4: Test Calls

**Test Sequence:**

1. **Test Service Extension** (6000)
   - From User 1001, dial: `6000`
   - Should hear: "1-2-3-4-5" then ringing
   - ✅ Confirms Asterisk path works

2. **Test User-to-User Call**
   - From User 1001, dial: `1002`
   - User 1002 should receive incoming call
   - User 1002 answers
   - ✅ Both users should have audio
   - ✅ NO "480 Temporarily Unavailable" error
   - ✅ Call ends cleanly

3. **Test Reverse Call**
   - From User 1002, dial: `1001`
   - Same behavior as above

### Step 5: Monitor Logs

**Terminal 1** - Kamailio logs:
```bash
sudo journalctl -u kamailio -f
```

Look for:
```
WebRTC peer-to-peer call: sip:1001@domain -> sip:1002@domain
Location found! Routing to: sip:IP:PORT;transport=ws
Added contact alias for peer-to-peer call
```

**Terminal 2** - Asterisk console:
```bash
sudo asterisk -rvvv

# In Asterisk console:
pjsip show contacts
core show channels
```

### Success Criteria

- [ ] Both users show "Registered" status
- [ ] Extension 6000 plays audio
- [ ] User 1001 can call 1002
- [ ] User 1002 receives call and can answer
- [ ] NO "480 Temporarily Unavailable" error
- [ ] Full bidirectional audio working
- [ ] Logs show "peer-to-peer call"

---

## 🐛 Troubleshooting

### Issue 1: Registration Fails

**Symptoms**: 401/403 errors, not registered

**Debug:**
```bash
sudo journalctl -u kamailio -f | grep REGISTER
sudo asterisk -rx "pjsip set logger on"
```

**Fix:**
- Check credentials match in `pjsip.conf`
- Verify Kamailio forwarding to Asterisk
- Check Path module loaded

### Issue 2: Calls Don't Reach Destination

**Symptoms**: "Not Found" error

**Debug:**
```bash
# Check location database
sudo kamcmd ul.dump

# Should show both users with WebSocket transport
```

**Fix:**
- Ensure both users registered
- Check domain normalization in logs
- Verify `lookup("location")` succeeds

### Issue 3: 480 Error When Answering (OLD PROBLEM)

**If you still see this**, peer-to-peer routing is NOT working!

**Debug:**
```bash
sudo journalctl -u kamailio -n 100 | grep "peer-to-peer"
```

**Should see:**
```
WebRTC peer-to-peer call: sip:1001@domain → sip:1002@domain
```

**If NOT**, check:
```bash
# Verify location lookup in INVITE handling
sudo grep -A10 "Check for peer-to-peer" /etc/kamailio/kamailio.cfg
```

**Fix**: Re-apply Kamailio config from this guide

### Issue 4: No Audio

**Symptoms**: Call connects but silence

**Debug:**
```bash
# Check RTP ports
sudo ufw status | grep 10000:20000

# Enable RTP debug
sudo asterisk -rx "rtp set debug on"
```

**Fix:**
```bash
# Open RTP ports
sudo ufw allow 10000:20000/udp

# Verify external_media_address correct
grep external_media /etc/asterisk/pjsip.conf
```

### Issue 5: WebSocket Won't Connect

**Symptoms**: Connection refused, timeout

**Debug:**
```bash
# Test TLS
openssl s_client -connect sip.sumthing.space:8443

# Check Kamailio listening
sudo netstat -tlnp | grep 8443
```

**Fix:**
```bash
# Check certificate
sudo certbot certificates

# Verify TLS config
sudo cat /etc/kamailio/tls.cfg

# Restart Kamailio
sudo systemctl restart kamailio
```

---

## 🔧 Maintenance

### Daily Checks

```bash
# Service status
sudo systemctl status kamailio asterisk

# Check registrations
sudo kamcmd ul.dump
sudo asterisk -rx "pjsip show contacts"
```

### Update TLS Certificate

```bash
# Renew (runs automatically via cron)
sudo certbot renew --force-renewal
sudo systemctl restart kamailio
```

### Backup Configuration

```bash
# Create backup
sudo tar czf ~/sip-backup-$(date +%Y%m%d).tar.gz \
  /etc/kamailio \
  /etc/asterisk

# List backups
ls -lh ~/sip-backup-*
```

### Add More Users

Edit `/etc/asterisk/pjsip.conf`, add:

```ini
[1003]
type=endpoint
context=default
disallow=all
allow=ulaw,opus
auth=1003
aors=1003
webrtc=yes
dtls_auto_generate_cert=yes
media_encryption=dtls
ice_support=yes
rtcp_mux=yes
use_avpf=yes
rtp_symmetric=yes
force_rport=yes
direct_media=no

[1003]
type=auth
auth_type=userpass
username=1003
password=STRONG_PASSWORD_HERE

[1003]
type=aor
support_path=yes
max_contacts=1
remove_existing=yes
```

Then:
```bash
sudo asterisk -rx "pjsip reload"
```

---

## 🎓 Why This Works

### The Problem We Solved

**Traditional approach** (routes everything through Asterisk):

```
Browser A → Kamailio → Asterisk → Kamailio → Browser B
                         ↓
                    (B2BUA strips DTLS)
                    Browser rejects SDP
                    480 error! ❌
```

**Our solution** (intelligent routing):

```
User-to-User:  Browser A → Kamailio → Browser B ✅
                             ↓
                      Preserves DTLS
                      Perfect WebRTC!

Services:      Browser → Kamailio → Asterisk ✅
                                    ↓
                               IVR features
```

### Key Technologies

1. **Location Database** (`usrloc` module)
   - Kamailio saves user registrations in memory
   - Fast lookup for routing decisions

2. **Intelligent Routing**
   - `lookup("location")` checks if destination is registered
   - Found? → Direct peer-to-peer
   - Not found? → Forward to Asterisk

3. **NAT Traversal**
   - `fix_nated_register()` - Fixes Contact during registration
   - `add_path_received()` - Adds Path with real IP:port
   - `handle_ruri_alias()` - Decodes routing info
   - `set_contact_alias()` - Encodes connection info

4. **WebRTC Preservation**
   - No SDP manipulation between browsers
   - DTLS fingerprints intact
   - ICE candidates preserved

---

## 📚 Reference

### Quick Commands

**Kamailio:**
```bash
sudo systemctl restart kamailio
sudo kamailio -c                    # Test config
sudo journalctl -u kamailio -f      # Live logs
sudo kamcmd ul.dump                 # Show registrations
```

**Asterisk:**
```bash
sudo systemctl restart asterisk
sudo asterisk -rvvv                 # Console
pjsip show endpoints                # Endpoints
pjsip show contacts                 # Registrations
core show channels                  # Active calls
```

### File Locations

```
/etc/kamailio/
├── kamailio.cfg       # Main routing
└── tls.cfg            # TLS certificates

/etc/asterisk/
├── pjsip.conf         # Endpoints
├── extensions.conf    # Dialplan
└── rtp.conf           # RTP settings
```

### Useful Links

- Kamailio: https://www.kamailio.org/docs/
- Asterisk: https://docs.asterisk.org/
- JsSIP: https://jssip.net/
- Let's Encrypt: https://letsencrypt.org/

---

## ✅ Completion Checklist

Your setup is complete when:

- [x] Kamailio and Asterisk running
- [x] TLS certificate installed and working
- [x] Users can register from browser
- [x] Extension 6000 plays audio
- [x] User 1001 can call 1002 with audio
- [x] User 1002 can call 1001 with audio
- [x] No "480 Temporarily Unavailable" errors
- [x] Logs show "WebRTC peer-to-peer call"

**Congratulations! You have a production-ready WebRTC calling system!** 🎉

---

_Last Updated: January 19, 2026_
_Tested on: Ubuntu 22.04 LTS, Kamailio 5.5, Asterisk 18.26.4_
