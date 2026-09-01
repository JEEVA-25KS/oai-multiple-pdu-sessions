# Multiple PDU Session Establishment and Validation in OAI 5G SA

Configuring and validating multiple simultaneous PDU sessions per UE in an OAI 5G
Standalone network — from a single-UE, 2-session baseline through a full 2-UE,
4-session-each deployment with policy-based routing and concurrent traffic
validation, plus a deep dive into how the core (UPF) actually distinguishes
sessions at the GTP-U level.

## Contents

| Doc | Covers |
|-----|--------|
| [01-single-ue-multi-pdu-baseline.md](docs/01-single-ue-multi-pdu-baseline.md) | Core (SMF/UPF/UDM) and UE config for 2 simultaneous PDU sessions on one UE; policy routing basics |
| [02-teid-and-gtp-tunneling.md](docs/02-teid-and-gtp-tunneling.md) | Deep dive: how the UPF differentiates sessions via GTP-U TEID, not Linux interfaces; Wireshark capture walkthrough |
| [03-two-ue-four-pdu-sessions.md](docs/03-two-ue-four-pdu-sessions.md) | Full 2-UE, 4-DNN-each deployment; per-UE routing tables; concurrent 8-stream iperf3 validation; MAC-layer confirmation |

## Objective

Configure and validate multiple simultaneous PDU sessions in an OAI 5G SA network.
Each session is associated with a different Data Network Name (DNN) — and
potentially a different QoS profile (5QI) — enabling scenarios like separating
internet traffic, IMS/voice traffic, and dedicated application traffic over
independent logical tunnels.

## Key facts

- Multiple PDU sessions are currently only supported via **OAI nrUE** (not COTS UEs)
- Each PDU session gets its own **TUN interface** at the UE (`oaitun_ue1`,
  `oaitun_ue1p2`, `oaitun_ue1p3`, `oaitun_ue1p4`) and its own IP subnet
- At the core, the UPF's single `tun0` interface handles **all** sessions — the real
  separation happens via **GTP-U TEID** (Tunnel Endpoint ID), not the Linux
  interface
- Each PDU session maps to a distinct **LCID/DRB** at the radio layer, confirmed
  directly in gNB MAC logs

## Result summary

Extended to a full 2-UE deployment, each UE established **4 PDU sessions** (DNNs:
`oai`, `openairinterface`, `ims`, `default`), each with its own tunnel interface and
IP. Policy-based routing correctly forwarded traffic through the intended session's
tunnel. Concurrent `iperf3` traffic (8 simultaneous streams — 4 per UE) validated
that all sessions carried traffic independently, and gNB MAC-layer statistics
confirmed each session mapped to a distinct logical channel (LCID 4–7) handled
simultaneously by the scheduler.
