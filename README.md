# Asterisk + Kamailio SIP Server Configuration

## Server Details
- **Public IP**: 54.91.7.122
- **Domain**: sip.sumthing.space
- **Region**: AWS us-east-1

## Components
- **Kamailio 5.5**: WebSocket to UDP proxy (WSS:8443, UDP:5062)
- **Asterisk 18.26.4**: PBX (UDP:5060, WS:8088)
- **TLS**: Let's Encrypt certificates (expires April 2026)

## Architecture


## Current Status

### ✅ Working
- WebSocket Secure (WSS) connection
- TLS certificates valid
- User registration (authentication works)
- Kamailio → Asterisk forwarding
- Test extension 6000 configured

### ❌ Issues
- **Registration Contact Header**: Asterisk responds with 200 OK but doesn't include Contact header in response
- **Root Cause**: Client sends Contact with TEST-NET IP (192.0.2.x) instead of real IP
- **Impact**: SIP.js client rejects registration response
- **Calls**: Not tested yet (blocked by registration issue)

## Configuration Files
-  - Main Kamailio configuration
-  - TLS settings
-  - PJSIP endpoints and transports
-  - Dialplan
-  - RTP port range

## Users
- 1001 / 1234
- 1002 / 1234

## Extensions
- 1001 - User 1001
- 1002 - User 1002
- 6000 - Test IVR (plays hello-world)

## Ports
- 8443 - Kamailio WSS
- 5062 - Kamailio UDP/TCP
- 5060 - Asterisk UDP
- 8088 - Asterisk WebSocket
- 10000-20000 - RTP media

## Next Steps
1. Fix Contact header rewriting in Kamailio
2. Test calls between users
3. Verify audio/RTP path
4. Scale to multiple Asterisk servers
