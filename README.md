# Student Lab Manual — Evil Twin Attack Training Lab

## 1. Objectives

By completing this lab you will:

- Build a legitimate WPA2 access point and a rogue "evil twin" access point
  impersonating it, entirely inside your own isolated VM.
- Force a client to disconnect from the legitimate AP and observe it
  reassociate to the rogue twin.
- Capture and analyze the resulting 802.11 traffic to identify the
  indicators that reveal an evil twin is present.
- Explain, in your own words, how this attack is detected and defended
  against in production wireless networks.

## 2. Rules of engagement — read before starting

- Everything in this lab runs on **simulated radios** (`mac80211_hwsim`)
  entirely inside your assigned VM. No real RF is transmitted at any point.
- **Do not** attempt any part of this exercise against real access points —
  your dorm WiFi, campus WiFi, a coffee shop, or any network you do not own
  or have explicit written authorization to test. Deauthentication attacks
  and rogue-AP deployment against networks you don't own are illegal in most
  jurisdictions (in the US, potentially violating the Computer Fraud and
  Abuse Act and FCC regulations) regardless of intent.
- Stay inside your assigned VM and namespaces. Do not attempt to bridge lab
  traffic to any other network interface.
- If anything in your VM looks like it's affecting something outside your
  VM, stop and notify the instructor immediately.

## 3. Environment overview

Your VM has three simulated WiFi radios, each moved into its own network
namespace. After `setup-netns.sh` runs, **the radio inside every namespace is
named `wlan0`** — the names do not collide because each namespace is an
isolated interface space (exactly the way three separate machines can each
have their own `eth0`). You always disambiguate a radio by the namespace it
lives in, never by a globally-unique interface name.

| Namespace | Interface | Role |
|---|---|---|
| `ns-corp` | `wlan0` | The legitimate "CorpNet-Secure" corporate AP (WPA2) |
| `ns-eviltwin` | `wlan0` | The rogue "CorpNet-Secure" AP you will stand up (open) |
| `ns-victim` | `wlan0` | A simulated client device you will manipulate |

All lab tooling lives under `/opt/eviltwin-lab/`.

### Recommended execution order

The Part numbers below are organized by topic, but the traffic you need to
capture is produced during the deauth/roam and the portal submission. Run the
lab in this operational order so your capture contains the full attack:

1. Parts 1–4 — build both APs and put the victim on the legitimate AP.
2. **Part 6 — start the capture in a second terminal** (before the deauth).
3. Part 5 — run the simulated deauth and reassociate the victim to the twin.
4. Part 8 — submit credentials through the captive portal.
5. Return to the capture terminal, stop it, and merge (rest of Part 6).
6. Part 7 — analyze the merged capture.

## 4. Logging in to your VM

Access your VM through Guacamole in your browser — your instructor will give
you a connection link. Do not connect to the VM by any other route.

- **Connection name:** matches your assigned VM, `kali-student-NN` (e.g.
  `kali-student-07`).
- **Username:** `studentNN`, matching your VM number with a leading zero if
  needed (e.g. `student07`, not `student7`).
- **Password:** provided by your instructor at the start of the session.

Every student has their own separate VM and their own separate account —
nothing you do is visible to or shared with any other student's VM. If your VM
is ever reset to its clean-start snapshot mid-lab, your environment (including
your login) returns to its initial state; ask your instructor if that happens
to you.

## 5. Part 1 — Verify your environment

```bash
ip netns list
```
<img width="614" height="145" alt="image" src="https://github.com/user-attachments/assets/4af6caff-04c2-484a-ba18-c2615059965a" />


**Expected output:** `ns-corp`, `ns-eviltwin`, `ns-victim`.


## 6. Part 2 — Stand up the legitimate Corporate AP

```bash
sudo /opt/eviltwin-lab/scripts/start-corp-ap.sh
sudo ip netns exec ns-corp iw dev wlan0 info
```
<img width="975" height="269" alt="image" src="https://github.com/user-attachments/assets/094abba1-aaeb-4a4e-bfec-a7352dad2bd8" />


Confirm the output shows `type AP` and `ssid CorpNet-Secure`. This AP uses
WPA2-PSK with passphrase `TrainingLab2026!` (defined in
`/opt/eviltwin-lab/configs/hostapd-corp.conf` — open it and read through the
config now so you understand every line).

## 7. Part 3 — Stand up the Evil Twin

```bash
sudo /opt/eviltwin-lab/scripts/start-evil-twin.sh
sudo ip netns exec ns-eviltwin iw dev wlan0 info
```
<img width="975" height="274" alt="image" src="https://github.com/user-attachments/assets/4736328a-5f83-4016-aece-01252a946d52" />

Confirm this radio also shows `type AP` and `ssid CorpNet-Secure` — the twin
deliberately clones the corporate SSID. Open
`/opt/eviltwin-lab/configs/hostapd-eviltwin.conf` and compare it line-by-line
to the corporate AP's config. Note in your lab notebook:

<img width="975" height="628" alt="image" src="https://github.com/user-attachments/assets/9023d167-38bf-4358-92cc-217fe78efd4e" />


- What is identical between the two configs?
- What is different, and why does each difference matter for the attack to
  work?

This script also starts `dnsmasq` (DHCP/DNS for anything that associates)
and a PHP captive portal on `10.10.11.1` — this is what will harvest
credentials in Part 8.

## 8. Part 4 — Baseline: associate the victim to the legitimate AP

First bring the victim radio up and record both APs' BSSIDs. **The BSSIDs are
randomized per VM, so you must read yours at runtime — do not copy the values
from a classmate or from this manual.** Both APs broadcast the identical SSID
`CorpNet-Secure`, so the BSSID is the only thing that lets you (and the victim)
tell them apart:

```bash
sudo /opt/eviltwin-lab/scripts/start-victim.sh

CORP_BSSID=$(sudo ip netns exec ns-corp     iw dev wlan0 info | awk '/addr/ {print $2}')
EVIL_BSSID=$(sudo ip netns exec ns-eviltwin iw dev wlan0 info | awk '/addr/ {print $2}')
echo "Corp AP   BSSID: $CORP_BSSID"
echo "Evil Twin BSSID: $EVIL_BSSID"
```
<img width="716" height="90" alt="image" src="https://github.com/user-attachments/assets/3f04ccfd-e325-4bfb-a376-ed314e57bb98" />


<img width="747" height="180" alt="image" src="https://github.com/user-attachments/assets/97f3506c-207c-40c1-8575-5653dbe51dc5" />


<img width="889" height="112" alt="image" src="https://github.com/user-attachments/assets/c6c1735b-8479-48a2-ada0-8d327d3b4ecb" />

 
Because `CorpNet-Secure` is **WPA2**, the victim cannot join it with a bare
`iw connect` (which only handles open or static-WEP networks). Use
`wpa_supplicant`, pinned to the corporate BSSID so the victim joins the
*legitimate* AP and not the twin:

```bash
cat > /tmp/wpa-corp.conf <<EOF
network={
    ssid="CorpNet-Secure"
    bssid=$CORP_BSSID
    psk="TrainingLab2026!"
    scan_ssid=1
}
EOF

sudo ip netns exec ns-victim wpa_supplicant -B -i wlan0 -c /tmp/wpa-corp.conf
sleep 3
sudo ip netns exec ns-victim iw dev wlan0 link
```
<img width="705" height="240" alt="image" src="https://github.com/user-attachments/assets/b044f490-fa3a-44f3-b378-205670dfa041" />


<img width="630" height="256" alt="image" src="https://github.com/user-attachments/assets/9653af9c-9a0a-487b-8728-0e8d82976582" />


Verify the `Connected to` line shows your **corporate** BSSID (`$CORP_BSSID`)
and its channel/frequency. Record this in your notebook — it is your baseline
"before" state.

> **Note — victim radio up/down state.** Starting `wpa_supplicant` brought
> `wlan0` **up** as part of associating — that is why `link` now reports a
> connection. This matters in Part 5: the simulated deauth *stops*
> `wpa_supplicant`, which leaves `wlan0` administratively **down**. A down
> interface cannot scan or associate, and `iw connect` will fail silently on
> one, so you will bring the radio back up before joining the evil twin. Check
> the state at any point with:
>
> ```bash
> sudo ip netns exec ns-victim ip link show wlan0   # look for: state UP
> ```

## 9. Part 5 — Force a roam with a (simulated) deauthentication attack

> **Start your capture first.** If you want the deauth and the roam in your
> pcap, begin Part 6's `capture-all.sh` in a second terminal *before* running
> the step below. See the recommended execution order in §3.

In a real evil-twin attack the attacker injects spoofed 802.11 deauthentication
frames from a monitor-mode radio to knock the victim off the legitimate AP. On
this simulated topology every radio is committed to a role and there is no
`wmediumd` to carry injected frames between namespaces, so we reproduce the
**effect** of the attack rather than the injection mechanism. The provided
script stops the victim's `wpa_supplicant` and issues a disconnect; mac80211
still emits a genuine deauthentication management frame (subtype `0x0c`) as the
station leaves, so a real deauth frame is present in your capture to analyze.

```bash
sudo /opt/eviltwin-lab/scripts/simulate-deauth.sh
```
<img width="731" height="390" alt="image" src="https://github.com/user-attachments/assets/af984f52-8c65-4891-85fd-2f98414aa28e" />


The script prints the victim's association state before and after and confirms
it is now disconnected and will not auto-reconnect.

**Stopping `wpa_supplicant` leaves the victim's `wlan0` administratively down.**
`iw connect` cannot scan or associate on a down interface — it fails *silently*,
and a subsequent `iw dev wlan0 link` reports `Not connected` with no error.
Bring the interface back up first, then reassociate, pinning the **evil twin's**
BSSID so the roam is deterministic (the twin is an open network, so `iw connect`
is the right tool here):

```bash
# 1. The victim radio is down after the deauth — bring it back up
sudo ip netns exec ns-victim ip link set wlan0 up

# 2. Associate to the evil twin by BSSID (open network → iw connect)
sudo ip netns exec ns-victim iw dev wlan0 connect CorpNet-Secure "$EVIL_BSSID"

# 3. iw connect is asynchronous — poll until the link is up (up to ~10 s)
for i in $(seq 1 10); do
    sudo ip netns exec ns-victim iw dev wlan0 link | grep -q "Connected to" && break
    sleep 1
done
sudo ip netns exec ns-victim iw dev wlan0 link
```

Confirm the `Connected to` BSSID now matches the **evil twin's** BSSID
(`$EVIL_BSSID`), not the corporate AP's. The victim has been moved onto the
attacker's AP while still believing it is on `CorpNet-Secure`.

> **Troubleshooting — if `link` still shows `Not connected`:**
> - **Interface down.** `sudo ip netns exec ns-victim ip link show wlan0` must
>   list `state UP`. If it does not, re-run step 1 — this is the most common
>   cause.
> - **BSSID variable empty.** An empty `$EVIL_BSSID` makes `iw connect` match
>   nothing. Verify with `echo "$EVIL_BSSID"`; if blank, re-run the two
>   `... | awk '/addr/'` commands from Part 4 to repopulate it.
> - **Twin not visible / not open.** `sudo ip netns exec ns-victim iw dev wlan0
>   scan | grep -iE 'BSS |SSID|RSN'` should list the evil twin's BSSID with
>   **no** `RSN:` line beneath it. If it shows `RSN:`, you are looking at the
>   corporate AP, not the twin — re-check `$EVIL_BSSID`.

> If `$CORP_BSSID` / `$EVIL_BSSID` are empty (e.g. you opened a fresh terminal
> and lost the shell variables), re-run the two `... | awk '/addr/'` commands
> from Part 4 to repopulate them.


## 10. Part 6 — Capture everything

Start this **in a second terminal, before the Part 5 deauth**, so the capture
records the roam and the credential submission:

```bash
sudo /opt/eviltwin-lab/scripts/capture-all.sh
# leave this running; perform Part 5 (deauth/roam) and Part 8 (portal
# submission) in your other terminal, then return here and press Ctrl-C
```

Then merge the three per-namespace captures:

```bash
cd /opt/eviltwin-lab/captures
mergecap -w combined.pcap corp.pcap eviltwin.pcap victim.pcap
```

## 11. Part 7 — Analyze the capture

Open `combined.pcap` in Wireshark (or use `tshark` on the command line) and
answer the following in your submission. Use these filters as starting
points:

```
wlan.fc.type_subtype == 0x08        # beacon frames
wlan.fc.type_subtype == 0x0c        # deauthentication frames
wlan.ssid == "CorpNet-Secure"       # both APs, filtered by SSID
http.request.method == "POST"       # the harvested-credential submission
```

Also try the sample capture included with this lab,
`eviltwin_training_sample.pcap`, which contains the same attack sequence
pre-recorded for practice before you analyze your own live capture.

## 12. Part 8 — Review the harvested credentials

```bash
cat /opt/eviltwin-lab/logs/harvested_creds.log
```

This demonstrates the endpoint impact of the attack: once a victim is on
the open evil twin and hits the captive portal, anything they submit is
visible to the attacker in cleartext. Note in your submission why this
would **not** work the same way against the legitimate AP's WPA2 encryption.

## 13. Cleanup

```bash
sudo /opt/eviltwin-lab/scripts/teardown.sh
```

This stops the APs, the DHCP/DNS and portal services, and the victim's
`wpa_supplicant`, and deletes the namespaces.

---

## 14. Review questions (answer key follows)

1. What field in a beacon frame did you use to tell the two APs apart,
   given they broadcast the identical SSID?
2. Why was the evil twin configured as an open network instead of WPA2?
3. What 802.11 frame subtype forces the victim off the legitimate AP, and
   what frame subtype/type is it (management vs. control vs. data)?
4. In the pcap, what specific evidence shows the victim's traffic was
   unencrypted once associated to the evil twin?
5. Name one 802.11 amendment/feature that would have prevented a real deauth
   attack from working, and briefly explain the mechanism.
6. Why does certificate-based EAP-TLS defeat evil-twin attacks even if the
   attacker perfectly clones the SSID and BSSID?
7. From a defender's perspective, what would a WIPS (Wireless Intrusion
   Prevention System) alert on in this scenario, before any credentials
   were ever harvested?
8. Why is this attack still effective against many real-world networks
   despite WPA2/WPA3 being widely deployed?

### Answer key

1. **BSSID** (the AP's MAC address) — and, in this lab, the beacon's channel
   and its RSN/privacy capability bits. The SSID alone is attacker-controlled
   and therefore not a reliable identifier.
2. To (a) make the twin more attractive to clients configured to prefer
   open/known networks or to auto-join on signal strength, and (b) allow the
   attacker to intercept the resulting traffic in cleartext — WPA2 would
   encrypt traffic even on the rogue AP, defeating the credential-harvest
   step.
3. A **deauthentication frame**, `type=0 (management)`, `subtype=12 (0x0c)`.
4. The captured HTTP POST to the captive portal (`log.php`) is visible in
   plaintext in the pcap — readable `user=` / `pass=` parameters — because
   the evil twin runs no link-layer encryption (`wpa=0`), unlike the
   legitimate AP's WPA2-CCMP traffic, which appears as opaque encrypted data.
5. **802.11w (Protected Management Frames / PMF)** cryptographically signs
   management frames including deauth/disassoc, so a forged deauth frame
   from an attacker without the session key is rejected by the client
   instead of honored.
6. Because EAP-TLS validates the **RADIUS/authentication server's
   certificate** as part of association — a rogue AP without the
   corresponding private key/valid cert chain cannot complete the EAP
   exchange, so the client refuses to associate even though the SSID and
   BSSID may be spoofed.
7. A WIPS would alert on **duplicate SSID with a differing/unauthorized
   BSSID and inconsistent channel/capability fields** appearing in the RF
   environment — this is detectable purely from beacon analysis, before any
   client ever associates or submits credentials.
8. Many networks (a) don't deploy 802.11w even though it's supported,
   (b) rely on WPA2-PSK rather than certificate-based enterprise auth, where
   a shared passphrase doesn't let the client verify *which* AP it's really
   talking to, and (c) have client devices configured to auto-join known
   SSIDs without verifying the AP's identity beyond the SSID string.

> **Instructor note on Q3/Q5 (lab-fidelity caveat):** In this simulated
> environment the deauthentication frame observed in the capture is
> *station-initiated* (the victim leaving, reason code "STA is leaving"),
> produced by `simulate-deauth.sh`, rather than an *attacker-spoofed* frame
> sourced from the AP's BSSID. The frame **type/subtype students identify in
> Q3 is identical** (management / `0x0c`), so the analysis exercise is
> unaffected. The difference matters for Q5: 802.11w/PMF specifically defeats
> the *attacker-spoofed* variant, because PMF rejects deauth frames not
> protected by the session key — it would not stop a legitimate station from
> choosing to leave. Strong answers may note this distinction. If you want
> students to observe genuinely injected deauth frames, that requires a fourth
> monitor-mode radio and `wmediumd` to bridge the namespaces (out of scope for
> this build).

## 15. Submission requirements

Submit: `combined.pcap`, your written answers to §14, and your lab notebook
notes from Parts 3 and 8. See the course rubric for grading weight per
section.
