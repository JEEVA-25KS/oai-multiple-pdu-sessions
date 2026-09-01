# 2. How the UPF Differentiates PDU Sessions — TEID and GTP-U Tunneling

Since the UPF only creates a single `tun0` interface even with multiple active PDU
sessions, a natural question is: how does it keep session traffic separate?

## The core answer

At the core (UPF), session separation does **not** rely on Linux interfaces like
`tun0` — that's just an I/O pipe. The real identification happens using **GTP-U
tunnel metadata**, specifically the **TEID (Tunnel Endpoint ID)** inside the GTP-U
header.

When a packet arrives at the UPF:
```
[Outer IP/UDP] + [GTP-U Header: TEID] + [Inner UE IP packet]
```
The UPF:
1. Reads the TEID
2. Looks up its internal session table
3. Maps it to the corresponding PDU session context
4. Processes the packet accordingly

For each session, the UPF maintains: TEID (UL & DL), UE IP address, DNN, and
QoS/FAR/PDR rules. Example mapping:
```
TEID 0xABC123 → PDU Session 1 → UE IP 10.0.0.2 → DNN "oai"
TEID 0xDEF456 → PDU Session 2 → UE IP 10.0.1.2 → DNN "openairinterface"
```

## Uplink flow (UE → UPF)
1. UE sends a packet via `oaitun_ue1` or `oaitun_ue1p2`
2. UE stack encapsulates into GTP-U, adding the TEID for that PDU session
3. gNB forwards to the UPF
4. UPF reads the TEID and identifies the correct session

Even though both sessions share the same `tun0` at the UPF, they don't mix — the
TEID keeps them separated before/after the shared interface.

## Downlink flow (UPF → UE)
1. Packet arrives at the UPF from the internet
2. UPF checks the destination UE IP (e.g. `10.0.0.2`)
3. Finds the corresponding session
4. Encapsulates with the correct TEID
5. Sends to gNB → UE

## Where TEID actually lives

TEID exists **only on the N3 interface (gNB ↔ UPF)** — not between the UE and gNB:

```
UE ─── DRB (radio bearer, PDCP/RLC/MAC — no GTP, no TEID) ─── gNB
gNB ═══ GTP-U + TEID (N3 interface, UDP 2152) ═══ UPF
```

**Step-by-step (uplink)**:
1. UE sends a packet (e.g. `10.0.0.2 → Internet`) over its DRB — no TEID at this stage
2. gNB already knows which DRB maps to which PDU session, so it encapsulates into
   GTP-U and attaches the TEID for that session
3. Packet sent to UPF: outer `gNB → UPF` (UDP 2152), GTP header `TEID = 0xAAA`,
   inner `10.0.0.2 → Internet`
4. UPF reads the TEID, looks it up, and identifies "this is PDU Session 1" — it
   **does not** see the DRB and does not rely on the UE IP first; it goes purely by
   TEID → session context

| Direction | TEID name | Created by | Used by |
|-----------|-----------|-----------|---------|
| Uplink | UL TEID | UPF (assigned per session, upon SMF request) | gNB inserts → UPF reads |
| Downlink | DL TEID | gNB (created as a distinct random number) | UPF inserts → gNB reads |

## "There's only one tunnel, right?"

**Correct framing**: there's **one transport tunnel** (the UDP/IP path between gNB
and UPF) but **multiple logical GTP tunnels**, distinguished by TEID:
```
Session 1: UL TEID = A, DL TEID = B
Session 2: UL TEID = C, DL TEID = D
Total = 4 TEIDs across 2 sessions
```
Because everything shares the same interface (`tun0`), same UDP port (2152), and
same IPs (`gNB ↔ UPF`), it visually looks like one tunnel — but the TEIDs are what
actually separate the sessions internally.

**On UL vs DL TEIDs looking different for the same IP**: this is expected and
correct — the UL and DL TEIDs are intentionally different values. TEID identifies
the **receiver's** tunnel endpoint context, not the sender: "this packet is intended
for THIS tunnel endpoint." So a packet with source `10.0.0.2` (uplink, going to the
UPF) and a packet with destination `10.0.0.2` (downlink, coming from the UPF) will
show different TEIDs — one belongs to the UPF's receiving context, the other to the
gNB's.

| For session | Take |
|-------------|------|
| Session 1 | UL TEID from packets where `ip.src = 10.0.0.2` |
| Session 2 | UL TEID from packets where `ip.src = 10.0.1.2` |

(similarly for DL TEID, filtering on destination instead)

## Capturing TEIDs with Wireshark

```bash
sudo apt update
sudo apt install -y wireshark
```
During install, answer **Yes** to "Should non-superusers be able to capture
packets?" If missed:
```bash
sudo dpkg-reconfigure wireshark-common
sudo usermod -aG wireshark $USER
sudo reboot
```

**Capture steps**:
1. Open Wireshark, double-click the `any` interface
2. Apply filter: `udp.port == 2152`
3. Start `iperf3` traffic from gNB/UE side
4. GTP-U packets will appear
5. Click any GTP packet → expand **User Datagram Protocol → GPRS Tunneling
   Protocol → TEID: 0xXXXXXXXX**

Example flows observed:
```
Source: 10.0.0.2         → Destination: 192.168.70.129   (UL, session 1)
Source: 10.0.1.2         → Destination: 192.168.70.129   (UL, session 2)
Source: 192.168.70.129   → Destination: 10.0.0.2          (DL, session 1)
Source: 192.168.70.129   → Destination: 10.0.1.2          (DL, session 2)
```
