# TryHackMe-OpenVPN-Troubleshooting-Kali-Linux-VM-VMware-Workstation-
TryHackMe OpenVPN එක Kali Linux VM (VMware) එකෙන් connect කරගන්න කරපු troubleshooting process එක document කළා. 🔧
# TryHackMe OpenVPN on Kali VM: Troubleshooting Writeup

TryHackMe OpenVPN connection එක **Kali Linux VM** (VMware Workstation) එකෙන් connect කරගන්න කරපු troubleshooting process එකේ writeup එකක්. Problem එක හොයපු විදිහ, test කරපු දේවල්, ලැබුණු evidence සහ ඉගෙනගත් lessons මෙතන තියෙනවා.

## Setup

- Host: Windows + VMware Workstation
- Guest: Kali Linux 2026.2 (VM)
- Goal: Lab work ඔක්කොම VM එක ඇතුළේම කරන්න (host machine එක lab traffic වලින් වෙන් කරලා තියන්න)
- Target: TryHackMe OpenVPN (UDP 1194 සහ TCP 443 profiles)

## Problem

`sudo openvpn <file>.ovpn` run කරාම VPN connect වුණේ නැහැ. Log එකේ:

- `Error opening configuration file` (file නම වැරදි වුණා)
- `TLS key negotiation failed to occur within 60 seconds`
- `Connection reset, restarting`

## Troubleshooting Timeline

| # | Test | Result | Lesson |
|---|------|--------|--------|
| 1 | Config file path fix කිරීම (`ls` වලින් verify) | Config file open වුණා | Command fail වුණාම real file name එක verify කරන්න |
| 2 | VM time/timezone check | Time sync fix කළා | Certificates වලට correct time ඕනේ |
| 3 | `ping`, `curl -I`, `nc -zv <server> 443` | Internet, DNS, HTTPS, TCP port open | Port open වීම = VPN working කියන එක නෙවෙයි |
| 4 | UDP 1194 profile | TLS handshake fail (repeat) | Server එකෙන් reply එන්නේ නැහැ |
| 5 | `tcpdump` (OpenVPN run වෙද්දීම) | Outgoing packets විතරයි, replies නැහැ | Packets යනවා, ආපහු එන්නේ නැහැ |
| 6 | VMware NAT සිට Bridged | IP වෙනස් වුණා, issue එක තවමත් තියෙනවා | VMware NAT root cause නෙවෙයි |
| 7 | TCP 443 profile | `Connection reset` | මැද කවුරුහරි connection එක cut කරන ලකුණක් |
| 8 | Windows host එකේ Cloudflare WARP on කරාම | `Initialization Sequence Completed` | VPN connect වුණා |

## Final Result

```
Initialization Sequence Completed
Data Channel: cipher 'AES-256-GCM'
tun0 UP: 192.168.195.235/18
```

## Root Cause (අවංකව)

**100% confirm කරන්න බැහැ.** Evidence අනුව network path එකේ (ISP, carrier, හෝ වෙනත් filtering එකක්) OpenVPN traffic වලට බාධාවක් තිබුණා වගේ පේනවා. හැබැයි TCP profile එකට මාරු වීම සහ WARP on කිරීම දෙකම කිට්ටු වෙලාවක සිද්ධ වුණ නිසා මොකක්ද ඇත්තටම fix කළේ කියලා වෙන් කරන්න බැහැ. Confirm කරන්න නම් WARP off කරලා TCP profile එක ආයෙ test කරන්න ඕනේ.

## Key Lessons

1. VPN problem එකක් diagnose කරද්දී **host එකේ VPN/DNS/proxy software** (WARP, antivirus, firewall) මුලින්ම check කරන්න.
2. Traffic capture (`tcpdump`) කරන්න්නේ traffic හදන process එක run වෙද්දීම.
3. `tcpdump` එකේ `>` (outgoing) සහ `<` (incoming) packets බලලා block එක යන ගමනේද එන ගමනේද කියලා අනුමාන කරන්න පුළුවන්.
4. එක වෙලාවකට එක දෙයක් විතරක් වෙනස් කරලා test කරන්න, නැත්නම් root cause එක වෙන් කරන්න බැහැ.
5. Assumption එකයි confirmed conclusion එකයි වෙන වෙනම ලියන්න.

## Useful Commands

```bash
ls ~/Desktop                                    # file name verify
grep -E "^(proto|remote)" file.ovpn             # protocol + server check
sudo openvpn ~/Desktop/file.ovpn                # VPN connect (terminal open තියන්න)
ip a show tun0                                  # VPN interface verify
sudo tcpdump -i eth0 -n host <server-ip> -c 10  # packets යනවාද/එනවාද බලන්න
sudo nmcli device disconnect eth0 && sudo nmcli device connect eth0   # IP renew
```

## Security Note

`.ovpn` files වල personal certificates සහ keys තියෙනවා. **ඒවා මේ repository එකට upload කරන්න එපා.**

## Disclaimer

මේ writeup එක learning purposes සඳහායි. Test කළේ මගේම VM එකේ සහ authorized TryHackMe lab environment එකේ පමණයි.
