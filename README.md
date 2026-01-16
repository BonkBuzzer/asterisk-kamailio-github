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
```
Browser (WebRTC) 
  ↓ WSS:8443
Kamailio (SIP Proxy)
  ↓ UDP:5060
Asterisk (PBX)
```

## Current Status ✅

### Working
- ✅ WebSocket Secure (WSS) connection
- ✅ TLS certificates valid
- ✅ User registration (authentication works)
- ✅ Kamailio → Asterisk forwarding
- ✅ Registration with Contact header
- ✅ Test extension 6000 configured
- ✅ Calls connect to extension 6000

### Known Issues
- ⚠️ **No audio on calls**: Codec mismatch - audio files are GSM format but WebRTC clients only support Opus
- ⚠️ **SUBSCRIBE for voicemail fails**: Voicemail not configured (non-critical)

### Next Steps
1. Convert audio files to Opus format OR install codec_opus module
2. Configure external_media_address for proper RTP routing
3. Test user-to-user calls (1001 ↔ 1002)

## Configuration Files
- `kamailio.cfg` - Main Kamailio configuration
- `kamailio-tls.cfg` - TLS settings
- `asterisk-pjsip.conf` - PJSIP endpoints and transports
- `asterisk-extensions.conf` - Dialplan
- `asterisk-rtp.conf` - RTP port range

## Users
- 1001 / 1234
- 1002 / 1234

## Extensions
- 1001 - User 1001
- 1002 - User 1002
- 6000 - Test IVR (plays hello-world, goodbye)

## Ports
- 8443 - Kamailio WSS
- 5062 - Kamailio UDP/TCP
- 5060 - Asterisk UDP
- 8088 - Asterisk WebSocket
- 10000-20000 - RTP media

## Important Notes
- Always commit working configs to git before making changes
- Asterisk crashes if invalid parameters are in pjsip.conf
- rewrite_contact setting breaks authentication
