# Evil Twin Attack — Easy Lab Manual

A beginner-friendly walkthrough using the **`eviltwin`** helper commands.
Everything runs safely inside three isolated Linux network namespaces on one
Kali VM — no real Wi‑Fi, no real network, nothing leaves the machine.

> **You do not need `sudo`.** Every `eviltwin` command elevates itself when it
> needs to. Each command also prints the *real* command it runs (the `+` line),
> so you still learn the underlying tools.

---

## 1. The attack in five lines

1. The victim trusts **`CorpNet-Secure`** — a normal WPA2 Wi‑Fi network.
2. The attacker clones that **name** on an **open** access point (the *evil twin*).
3. A (simulated) **deauth** knocks the victim off the real AP.
4. Seeing the same name, the victim **auto‑joins the imposter**.
5. A fake login page (**captive portal**) captures the password **in cleartext**.

**Key idea:** both APs shout the same **SSID** (`CorpNet-Secure`). The only thing
that tells them apart is the **BSSID** (the radio's MAC address). The whole lab
is about watching the victim's BSSID flip from the *real* AP to the *twin* while
the name never changes.

| Term | Plain meaning |
|------|---------------|
| **SSID** | The Wi‑Fi **name** you see (`CorpNet-Secure`). |
| **BSSID** | The AP's **hardware address** — unique per AP, even with the same name. |
| **Deauth** | A frame that kicks a device off its Wi‑Fi. |
| **Captive portal** | The "sign in to use Wi‑Fi" page — here it's a trap. |
| **WPA2 vs Open** | Real AP is encrypted (needs a password); the twin is open (no password). |

---

## 2. The whole lab on one page

Run these in order. Steps 5 and the capture happen in a **second terminal**.

| # | Command | What it does | Manual part |
|---|---------|--------------|-------------|
| 1 | `eviltwin setup` | Create the 3 namespaces (once per boot) | — |
| 2 | `eviltwin corp` | Start the **real** corporate AP (WPA2) | Part 2 |
| 3 | `eviltwin twin` | Start the **evil twin** (open) + captive portal | Part 3 |
| — | `eviltwin compare` | See how the two AP configs differ | Part 3 |
| 4 | `eviltwin victim` | Put the victim on the **real** AP (baseline) | Part 4 |
| — | `eviltwin show` | **Dashboard: which AP is the victim on?** | any time |
| 5 | `eviltwin capture` | *(2nd terminal)* record all traffic — start **before** the deauth | Part 5 |
| 6 | `eviltwin deauth` | Kick the victim off the real AP | Part 6  |
| 7 | `eviltwin roam` | Move the victim onto the **evil twin** | Part 5 |
| 8 | `eviltwin portal` | Victim gets an IP + submits its password | Part 8 |
| — | `eviltwin creds` | See the harvested passwords | Part 8 |
| 9 | *(2nd terminal)* `Ctrl-C`, then `eviltwin merge` | Combine the captures | Part 6 |
| 10 | `eviltwin analyze` | Run the packet‑analysis filters | Part 7 |
| 11 | `eviltwin teardown` | Stop everything, delete namespaces | Cleanup |

> **Stuck?** Run `eviltwin show` to see the current state, or `eviltwin --help`
> for the full command list.

---

## 3. Step by step

Each `eviltwin` command can be run in one shot, **or** broken into smaller steps
so you can watch each action. The smaller steps are shown in *(italics)*.

### Part 1 — Check your environment
```bash
eviltwin status
```
Confirms the three namespaces (`ns-corp`, `ns-eviltwin`, `ns-victim`) exist. If
any are missing, run `eviltwin setup` (needed once after every reboot), such as:

<img width="850" height="259" alt="image" src="https://github.com/user-attachments/assets/51033a9c-3b32-413e-990c-269e417305a3" />

In this case, you got Not all namesspaces exist yet. Run: eviltwin setup

<img width="1152" height="415" alt="image" src="https://github.com/user-attachments/assets/960536e9-6b89-4b1c-bc27-41f33b20ceb5" />



### Part 2 — Start the real Corporate AP
```bash
eviltwin corp
```
*(smaller steps: `eviltwin corp up` = start it, `eviltwin corp info` = show radio)*

Starts the legitimate WPA2 access point named `CorpNet-Secure`. **Look for**
`type AP` and `ssid CorpNet-Secure` in the output.

<img width="1535" height="553" alt="image" src="https://github.com/user-attachments/assets/314b2fb2-ed97-49c1-9403-3892d04f0078" />

### Part 3 — Start the Evil Twin
```bash
eviltwin twin
```
This starts a **second** AP with the **same name** but **open** (no password),
plus DHCP and the fake login page.

<img width="1457" height="553" alt="image" src="https://github.com/user-attachments/assets/13ae65c2-33c6-45fd-a262-5d81f409ae01" />

Now compare the two APs' configs:
```bash
eviltwin compare
```
**You'll see** what is identical (the SSID/name) and what differs:

| Setting | Real AP | Evil Twin | Why it matters |
|---------|---------|-----------|----------------|
| `ssid` | `CorpNet-Secure` | `CorpNet-Secure` | Same name → victim reconnects automatically |
| `wpa` | `2` (WPA2) | `0` (open) | Twin needs no password → anyone joins, nothing to crack |
| `channel` | `6` | `11` | Different channel so the two APs don't interfere |

<img width="1415" height="1291" alt="image" src="https://github.com/user-attachments/assets/fa256bc2-997d-465b-abde-9744a4a7adee" />

### Part 4 — Baseline: put the victim on the REAL AP
```bash
eviltwin victim
```
*(smaller steps, run in order:)*

| Step | Command | What you observe |
|------|---------|------------------|
| 1 | `eviltwin victim up` | Radio is **UP** but **Not connected** yet |
| 2 | `eviltwin victim conf` | The config file **pins** the real AP's BSSID; WPA2 needs a password |
| 3 | `eviltwin victim join` | Uses `wpa_supplicant` (WPA2 needs it) to associate |
| 4 | `eviltwin victim verify` | `Connected to <real BSSID>` — this is your **BEFORE** state |

<img width="1474" height="1039" alt="image" src="https://github.com/user-attachments/assets/429c69b3-2bbd-4f8f-aed3-efbddd3c3939" />

Check the state at any time:
```bash
eviltwin victim state    # shows radio UP/DOWN and connected/not connected
eviltwin show            # dashboard: which AP the victim is on
```
**Write down** the BSSID the victim is connected to — you'll compare it later.

<img width="1657" height="704" alt="image" src="https://github.com/user-attachments/assets/1993bb34-ba60-406e-b954-5ae995197739" />

You may find the Channell information from the FREQ: 2462.0 MHz

<img width="816" height="624" alt="image" src="https://github.com/user-attachments/assets/5478b658-1e1f-4828-be3d-191504780ad0" />

```bash
eviltwin show
```
<img width="1258" height="253" alt="image" src="https://github.com/user-attachments/assets/6885a6a1-40fd-4fcf-8483-b2af7f106870" />

### Part 5 — Capture the traffic
In a **second terminal**, *before* the deauth:
```bash
eviltwin capture      # press Ctrl-C to stop
```
<img width="1506" height="393" alt="image" src="https://github.com/user-attachments/assets/30e847af-bd17-4e1d-964c-1e204e827107" />


After the attack, stop it (`Ctrl-C`) and combine the files:
```bash
eviltwin merge
```

### Part 6 — Force the roam (the attack)

**Start your capture first**  in a second terminal if you want the
attack in your `.pcap`.

**Kick the victim off the real AP:**  Run  the following command from the first terminal window:
```bash
eviltwin deauth
```
It shows the victim's state before and after, and confirms it's now
disconnected and won't reconnect on its own.

<img width="1679" height="1135" alt="image" src="https://github.com/user-attachments/assets/476ba620-0230-457a-9638-6eeede6ec731" />

**Move the victim onto the evil twin:**
```bash
eviltwin roam
```
*(smaller steps:)*

| Step | Command | What you observe |
|------|---------|------------------|
| 1 | `eviltwin roam up` | Bring the radio back up (`state UP`) — it may be down after the deauth |
| 2 | `eviltwin roam connect` | Uses `iw connect` (the twin is **open**, so no password) |
| 3 | `eviltwin roam verify` | `Connected to <evil BSSID>` — different from your baseline! |

<img width="1653" height="1215" alt="image" src="https://github.com/user-attachments/assets/9d76dfca-c4b3-4778-bac3-2eb47f32f549" />

Now run the dashboard again:
```bash
eviltwin show
```
The **Victim** line now says **ON THE EVIL TWIN** — even though the name is still
`CorpNet-Secure`. That's the whole attack in one screen.

<img width="1540" height="258" alt="image" src="https://github.com/user-attachments/assets/3095a681-4559-43a8-9924-535647b51f3e" />


### Part 7 — Analyze the packets
```bash
eviltwin analyze
```
Runs `tshark` filters for beacons, the deauth frame, both APs by SSID, the
credential **POST**, and the victim's DHCP lease.
```bash
tshark -r /opt/eviltwin-lab/captures/combined.pcap -Y "wlan.fc.type_subtype == 0x08"
```
<img width="1919" height="355" alt="image" src="https://github.com/user-attachments/assets/4e78d8c7-8e7f-44e6-99f2-712c2b706978" />


```bash
tshark -r /opt/eviltwin-lab/captures/combined.pcap -Y "wlan.fc.type_subtype == 0x0c"
```
<img width="1624" height="124" alt="image" src="https://github.com/user-attachments/assets/9dea4081-27e6-48ce-b9f6-aa9577eb7e83" />

### Part 8 — Harvest credentials through the portal
```bash
eviltwin portal
```
*(smaller steps:)*

| Step | Command | What you observe |
|------|---------|------------------|
| 1 | `eviltwin portal dhcp` | Victim gets a **10.10.11.x** address — from the **attacker's** DHCP |
| — | `eviltwin portal form` | *(optional)* read the login page's real field names |
| 2 | `eviltwin portal submit` | Sends a username + password to the fake page |

Use your own values if you like:
```bash
eviltwin portal submit --user alice --pass Hunter2!
```
Then see what the attacker captured **in cleartext**:
```bash
eviltwin creds
```
> **Think about this:** the same password over the real AP's **WPA2** link would
> be **encrypted** on the air. On the open twin, it's readable. That's the point.

### Cleanup
```bash
eviltwin teardown
```
Stops the APs, DHCP/DNS, portal, and the victim's client, and deletes the
namespaces. (After a reboot, start again from `eviltwin setup`.)

---

## 4. Quick troubleshooting

| Problem | Try this |
|---------|----------|
| "Namespace ... MISSING" | `eviltwin setup` (needed once per boot) |
| Victim won't connect after deauth | `eviltwin victim state` — radio must show **state UP**; `eviltwin roam up` fixes it |
| Not sure where the victim is | `eviltwin show` |
| BSSID looks empty | Make sure the AP is up: `eviltwin corp` / `eviltwin twin`, then `eviltwin bssids` |
| Want the full command list | `eviltwin --help` |

---

## 5. Check your understanding

1. Both APs use the name `CorpNet-Secure`. What single value lets the victim (and
   you) tell them apart?
2. Why does the **real** AP need `wpa_supplicant` while the **evil twin** only
   needs `iw connect`?
3. After the deauth, why might `iw connect` silently fail — and which command
   reveals the cause?
4. The captured password is readable in cleartext here. Why would that **not**
   work the same way against the real WPA2 AP?
5. Name one defense that would stop this attack (hint: think about how the client
   *verifies* the network before trusting it).

---

*Lab use only. Everything here runs in isolated namespaces on a training VM.
Never run these techniques against networks or devices you don't own or aren't
authorized to test.*
