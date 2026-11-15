# TryHackMe OpenVPN Troubleshooting – Kali Linux VM (VMware Workstation)

## 1. Project Overview

මේ troubleshooting exercise එකේ main objective එක වුණේ Windows host එකක VMware Workstation හරහා run වෙන **Kali Linux VM** එකෙන් **TryHackMe OpenVPN** network එකට connect වීම.

Lab activities සහ security testing tools Kali VM එක ඇතුළේම run කරන්න තීරණය කළා. ඒ නිසා host operating system එකේ direct testing අවම කරලා, VM networking path එක troubleshoot කළා.

**Environment**

- **Host OS:** Windows
- **Virtualization:** VMware Workstation
- **Guest OS:** Kali Linux
- **VPN:** TryHackMe OpenVPN
- **Profiles tested:** UDP/1194 and TCP/443

මෙම write-up එකෙන් troubleshoot කළ steps, observations, commands, සහ test results වලින් ගන්න පුළුවන් conclusions document කරනවා. තහවුරු නොවූ දේවල් definite root cause එකක් ලෙස සඳහන් කරන්නේ නැහැ.

## 2. Troubleshooting Timeline

| # | කළ දේ (Action) | ලැබුණු ප්‍රතිඵලය (Observation) | ඉගෙනගත් දේ / Technical finding |
|---|---|---|---|
| 1 | Download කළ `.ovpn` profile එකෙන් OpenVPN start කළා. | `Error opening configuration file` error එක ලැබුණා. | File path/name එක වැරදියි. `ls` command එකෙන් actual filename එක check කළා. Type කරපු නමේ `hmlw` වෙනුවට `malithmlw` තිබිය යුතු වුණා. |
| 2 | Nmap room එකට ලබාදුන් TCP/443 profile එක test කළා. | TCP connection එක establish වුණත් TLS negotiation එක complete වුණේ නැහැ. | TCP port එකට connect වෙන්න පුළුවන් වීමෙන් OpenVPN TLS handshake එක හෝ VPN connection එක සාර්ථකයි කියලා තහවුරු වෙන්නේ නැහැ. |
| 3 | Kali VM එකේ system time එක check කළා. | Time/time-zone mismatch එකක් තිබුණු නිසා නිවැරදි කළා. | TLS certificate validity check කිරීමේදී correct system time වැදගත්. නමුත් time එක නිවැරදි කිරීමෙන් පමණක් VPN issue එකේ root cause එක හඳුනාගන්න බැහැ. |
| 4 | `ping 8.8.8.8`, `curl -I` සහ `nc -zv <server> 443` මගින් basic connectivity test කළා. | Tests සාර්ථක වුණා. | VM එකට basic internet access තිබුණා සහ test කළ TCP/443 port එකට connection එකක් establish කළ හැකි වුණා. මෙයින් UDP/1194 path එක හෝ OpenVPN handshake එක test වෙන්නේ නැහැ. |
| 5 | `ap-south-1` regular profile එකෙන් UDP/1194 භාවිත කර connect වීමට උත්සාහ කළා. | `TLS key negotiation failed within 60 seconds` error එක නැවත නැවත ලැබුණා. | OpenVPN client එකට negotiation period එක තුළ expected response එක ලැබුණේ නැහැ. මේ error එකෙන් packets drop/filter වුණේ කොතැනද කියලා තනිවම හඳුනාගන්න බැහැ. |
| 6 | OpenVPN connect වෙන්න උත්සාහ කරන අතරතුර `tcpdump` මගින් `eth0` interface එක capture කළා. | VPN server එකට යන packets සහ retransmissions පෙනුණා. Capture කළ කාලය තුළ corresponding replies පෙනුණේ නැහැ. | Kali VM එකෙන් packets පිටත් වෙන බව capture එකෙන් පෙනුණා. නමුත් replies නොපෙනීමෙන් ඒවා local host එකේද, network path එකේද, remote endpoint එකේද, වෙනත් filtering device එකකද නැවතුණේ කියලා තහවුරු වෙන්නේ නැහැ. |
| 7 | VMware network adapter එක **NAT → Bridged** ලෙස වෙනස් කළා. | VM IP එක `192.168.26.128` සිට `10.242.46.94` දක්වා වෙනස් වුණා. VPN එක තවමත් fail වුණා. | Adapter mode එක මාරු කිරීමෙන් issue එක resolve වුණේ නැහැ. ඒ නිසා VMware NAT එක පමණක් හේතුව වීම අඩු හැකියාවක් වුණත්, සියලු host/VM networking issues ඉවත් කළා කියන්න බැහැ. |
| 8 | VMware Virtual Network Editor settings පරීක්ෂා කළා. | පැහැදිලි misconfiguration එකක් හමු වුණේ නැහැ. | පරීක්ෂා කළ VMware virtual network settings වල obvious configuration problem එකක් පෙනුණේ නැහැ. |
| 9 | TryHackMe TCP configuration (port 443) නැවත test කළා. | මුල් attempt එකේ `Connection reset` ලැබුණා. පසුව TCP connection stage එකට යාමට හැකි වුණා. | TCP connection එක attempts අතර වෙනස් ලෙස හැසිරුණා. `Connection reset` කියන්නේ connection එක path එකේ කොතැනක හෝ reset/close කළ බවයි. Reset එක යැවූ device එක කුමක්ද කියලා එයින් පමණක් හඳුනාගන්න බැහැ. TCP connection එක establish වීමත් VPN session එක complete වීමක් නොවේ. |
| 10 | Windows host එකේ **Cloudflare WARP** Traffic and DNS mode එකෙන් enabled බව දැක්කා. ඉන්පසු VPN connection එක නැවත test කළා. | OpenVPN initialization complete වුණා සහ `tun0` interface එක create වුණා. | TCP profile එක භාවිත කරමින් WARP enabled තිබූ අවස්ථාවේ VPN එක සාර්ථක වුණා. නමුත් එකම වෙලාවේ relevant conditions කිහිපයක් වෙනස් වී තිබුණු නිසා WARP එකම fix එක බව prove වෙන්නේ නැහැ. |

## 3. Successful Connection – Evidence

VPN connection එක සාර්ථක වූ අවස්ථාවේ OpenVPN output එකේ පහත messages පෙනුණා:

```text
Initialization Sequence Completed
Data Channel: cipher 'AES-256-GCM'
net_addr_v4_add: 192.168.195.235/18 dev tun0
net_route_v4_add: 10.48.0.0/12 via 192.168.192.1
```

ඊට පස්සේ `tun0` interface එකේ status එක check කළේ මෙහෙමයි:

```bash
ip a show tun0
```

`tun0` interface එක `UP` state එකේ තිබුණා. `Initialization Sequence Completed` message එක, tunnel IP address එක assign වීම, route එක add වීම සහ interface එක `UP` වීම එකට සලකා බැලූ විට OpenVPN tunnel එක establish වී ඇති බව තහවුරු වුණා.

## 4. Root-Cause Assessment

**Exact root cause එක 100% තහවුරු කරගන්න ලැබුණේ නැහැ.**

Troubleshooting අතරතුර collect කළ evidence අනුව:

- UDP/1194 profile එකේදී OpenVPN TLS key negotiation timeout වුණා.
- `tcpdump` capture එකේ Kali VM එකෙන් VPN server එකට packets යන බව පෙනුණත්, capture කළ interface එකේ corresponding replies පෙනුණේ නැහැ.
- TCP/443 profile එකේ මුල් attempt එක `Connection reset` වුණා. පසුව TCP connection stage එකට ළඟා වුණා.
- Windows host එකේ Cloudflare WARP enabled තිබූ අවස්ථාවේ, TCP profile එකෙන් VPN connection එක සාර්ථක වුණා.

මේ observations වලින් network path එක, traffic filtering, remote endpoint response, හෝ connection conditions සම්බන්ධ issue එකක් තිබෙන්නට ඇති බවට සැක කළ හැකියි. නමුත් **ISP එක OpenVPN block කළා** හෝ **Cloudflare WARP එකෙන්ම issue එක fix වුණා** කියලා conclusively කියන්න ප්‍රමාණවත් evidence නැහැ.

**වැදගත් networking note:** VMware adapter එක **Bridged** mode එකේ තිබුණොත් Kali VM traffic එක Windows host එකේ WARP tunnel එක හරහා අනිවාර්යයෙන්ම යනවා කියලා assume කරන්න බැහැ. ඒ traffic path එක වෙනම routing සහ packet-capture evidence මගින් verify කළ යුතුයි. එම නිසා VPN success එක සහ WARP enabled වීම අතර තියෙන්නේ observed correlation එකක් පමණයි; causation එක තහවුරු වෙලා නැහැ.

Root cause එක isolate කරන්න controlled test එකක් අවශ්‍යයි. උදාහරණයක් ලෙස, එකම TCP profile එක සහ එකම VMware adapter mode එක තබාගෙන WARP **OFF** සහ **ON** අවස්ථා දෙකේ වෙන වෙනම test කරලා results record කරන්න. දෙකේදීම එකම interface එක capture කරලා compare කළොත් වෙනසක් හඳුනාගන්න පහසුයි. එහෙම කළත් exact filtering device එක හඳුනාගැනීමට network-side evidence අවශ්‍ය විය හැකියි.

## 5. Key Lessons Learned

1. **File path සහ filename එක මුලින්ම verify කරන්න.** File එක missing කියලා හිතන්න කලින් `ls` සහ shell tab completion භාවිත කරන්න.
2. **Basic connectivity සහ VPN connectivity කියන්නේ වෙනස් දේවල්.** `ping`, HTTPS request එකක් හෝ `nc -zv` සාර්ථක වුණා කියලා OpenVPN negotiation එක සාර්ථක බව අදහස් වෙන්නේ නැහැ.
3. **System time එක check කරන්න.** Incorrect system time එක certificate validation errors ඇති කරන්න පුළුවන්. TLS-related troubleshooting වලදී clock එක verify කිරීම වැදගත්.
4. **Issue එක reproduce කරන වෙලාවේම packet capture කරන්න.** OpenVPN connect වෙන්න උත්සාහ කරන අතරතුර `tcpdump` start කරන්න. Traffic generate නොවන වෙලාවක capture කළොත් ඒ attempt එක diagnose කරන්න ඒකෙන් ප්‍රයෝජනයක් නැහැ.
5. **Packet direction නිවැරදිව interpret කරන්න.** `tcpdump` output එකේ `source > destination` කියන්නේ packet එක යන direction එකයි. `>` symbol එක පමණක් බලලා outbound කියලා තීරණය නොකර source/destination IP සහ ports Kali VM එක හා VPN server එක සමඟ compare කරන්න.
6. **Guest සහ host networking දෙකම check කරන්න.** VMware adapter mode එකත්, host එකේ VPN/DNS/firewall/security software settings ත් සලකා බලන්න. Host VPN එකක් enabled කියලා bridged guest traffic එකත් ඒ හරහා යනවා කියලා assume කරන්න එපා.
7. **එක වරකට එක variable එකක් පමණක් වෙනස් කරන්න.** VPN protocol එකත් host networking software එකත් එකවර වෙනස් කළොත් මොන change එකෙන් result එක වෙනස් වුණාද කියලා isolate කරන්න අමාරුයි.
8. **Evidence සහ assumptions වෙන් කරගන්න.** “මේ capture එකේ reply එකක් පෙනුණේ නැහැ” කියන්නේ evidence එකක්. “ISP එක packets block කළා” කියන්නේ තවත් tests මගින් තහවුරු කළ යුතු hypothesis එකක්.

## 6. Useful Commands – Quick Reference

පහත commands run කිරීමට පෙර example values ඔයාගේ actual profile name, VPN server IP සහ interface name අනුව replace කරන්න.

**1. Download කළ `.ovpn` file එකේ නම check කිරීම**

```bash
ls -lah "$HOME/Desktop"
```

**2. Profile එකේ protocol සහ remote server details බලන්න**

```bash
grep -E '^(proto|remote)[[:space:]]' "$PROFILE"
```

**3. OpenVPN connection එක start කිරීම**

```bash
sudo openvpn --config "$PROFILE"
```

**4. VPN tunnel interface එක check කිරීම**

```bash
ip -br address show tun0
```

**5. Routing table එක බලන්න**

```bash
ip route
```

**6. VPN server එකට යන route එක check කිරීම**

```bash
ip route get "$SERVER_IP"
```

**7. VPN server traffic capture කිරීම**

```bash
sudo tcpdump -ni "$INTERFACE" "host $SERVER_IP and (udp port 1194 or tcp port 443)" -c 20
```

**Variables set කිරීම (example):**

```bash
PROFILE="$HOME/Desktop/your-profile-name.ovpn"
SERVER_IP="your.vpn.server.ip"
INTERFACE="eth0"  # Your Kali interface may have a different name
```

Actual interface name එක `eth0` නොවෙන්න පුළුවන්. `ip -br address` command එකෙන් available interfaces check කරන්න. OpenVPN process එක stop කිරීමට `Ctrl+C` භාවිත කරන්න.

## 7. Security & Privacy Note

ඔයාගේ personal TryHackMe `.ovpn` file එක, embedded certificates/keys, credentials හෝ account-specific configuration public GitHub repository එකකට හෝ LinkedIn post එකකට upload කරන්න එපා. Publicly share කරන screenshots සහ logs වල sensitive information redact කරන්න. සියලු testing authorized TryHackMe lab environment එක තුළ පමණක් සිදු කරන්න.

## 8. Conclusion

මේ troubleshooting exercise එකෙන් networking layers එකින් එක test කිරීමේ වැදගත්කම තේරුම් ගන්න ලැබුණා: configuration file එක verify කිරීම, basic internet connectivity, TCP/UDP reachability, TLS/OpenVPN negotiation, tunnel interface creation සහ routing.

අවසානයේ VPN connection එක සාර්ථක වුණත්, කලින් attempts fail වීමට නිශ්චිත හේතුව තහවුරු කරගන්න ලැබුණේ නැහැ. ඒ uncertainty එකත් සමඟ evidence-based findings document කිරීම හොඳ technical troubleshooting practice එකක වැදගත් කොටසක්.

---

**Tags:** `#TryHackMe` `#OpenVPN` `#KaliLinux` `#VMware` `#Networking` `#CyberSecurity` `#Troubleshooting` `#LearningInPublic`
