# 3. Multiple PDU Session Setup and Validation — 2 UEs, 4 Sessions Each

## Objective
Configure and validate multiple simultaneous PDU sessions across **two UEs**, each
with **four DNNs** (`oai`, `openairinterface`, `ims`, `default`), demonstrating
independent tunnels, policy-based routing, and concurrent traffic across all 8
sessions total.

## Introduction

In a 5G SA network, a single UE can establish multiple PDU sessions concurrently,
each tied to a different DNN and potentially a different QoS profile (5QI) — useful
for separating internet traffic, IMS/voice traffic, and dedicated application
traffic over independent logical tunnels. Here, four PDU sessions per UE (DNNs
`oai`, `openairinterface`, `ims`, `default`) are configured for two UEs, each
session getting its own tunnel interface and IP address, with policy-based routing
ensuring traffic from each session routes through its correct tunnel.

## Core network configuration (OAI CN5G)

Same four checkpoints as the [single-UE baseline](01-single-ue-multi-pdu-baseline.md#core-network-configuration-oai-cn5g)
(SMF `dnnSmfInfoList`, `dnns:` block, UPF `dnnUpfInfoList`, UDM subscription DB),
extended to 4 DNNs per UE with full 5QI/QoS profiles per DNN:

```sql
-- UE 1 (IMSI 001010000000001)
INSERT INTO `SessionManagementSubscriptionData` (`ueid`, `servingPlmnid`, `singleNssai`, `dnnConfigurations`) VALUES
('001010000000001', '00101', '{"sst": 1, "sd": "FFFFFF"}',
 '{"oai":{"pduSessionTypes":{"defaultSessionType":"IPV4"},"sscModes":{"defaultSscMode":"SSC_MODE_1"},
    "5gQosProfile":{"5qi":6,"arp":{"priorityLevel":15,"preemptCap":"NOT_PREEMPT","preemptVuln":"PREEMPTABLE"},"priorityLevel":1},
    "sessionAmbr":{"uplink":"1000Mbps","downlink":"1000Mbps"},"staticIpAddress":[{"ipv4Addr":"10.0.0.2"}]},
   "openairinterface":{"pduSessionTypes":{"defaultSessionType":"IPV4V6"},"sscModes":{"defaultSscMode":"SSC_MODE_1"},
    "5gQosProfile":{"5qi":9,...},"sessionAmbr":{"uplink":"1000Mbps","downlink":"1000Mbps"}},
   "ims":{"pduSessionTypes":{"defaultSessionType":"IPV4V6"},"sscModes":{"defaultSscMode":"SSC_MODE_1"},
    "5gQosProfile":{"5qi":2,...},"sessionAmbr":{"uplink":"1000Mbps","downlink":"1000Mbps"}},
   "default":{"pduSessionTypes":{"defaultSessionType":"IPV4V6"},"sscModes":{"defaultSscMode":"SSC_MODE_1"},
    "5gQosProfile":{"5qi":9,...},"sessionAmbr":{"uplink":"1000Mbps","downlink":"1000Mbps"}}}');

-- UE 2 (IMSI 001010000000002) — identical DNN structure, static IP 10.0.0.3 for "oai"
INSERT INTO `SessionManagementSubscriptionData` (`ueid`, `servingPlmnid`, `singleNssai`, `dnnConfigurations`) VALUES
('001010000000002', '00101', '{"sst": 1, "sd": "FFFFFF"}', '{...}');
```

**Optional** — `SmfSelectionSubscriptionData` row per UE (if missing), listing all
4 DNNs under the subscribed S-NSSAI:
```sql
INSERT INTO `SmfSelectionSubscriptionData` (`ueid`, `servingPlmnid`, `subscribedSnssaiInfos`) VALUES (
  '001010000000001', '00101',
  '{"subscribedSnssaiInfos": [{"sNssai": {"sst": 1, "sd": "FFFFFF"},
     "dnnInfos": [{"dnn": "oai"}, {"dnn": "openairinterface"}, {"dnn": "ims"}, {"dnn": "default"}]}]}'
);
-- repeat for UE 2
```

### UPF entrypoint — routes for all 4 subnets
`oai-cn5g/docker-compose.yaml` 

```yaml
entrypoint: /bin/bash -c \
  " /openair-upf/bin/oai_upf -c /openair-upf/etc/config.yaml &
    UPF_PID=$$!;
    sleep 15;
    ip addr add 10.0.0.1/24 dev tun0 2>/dev/null || true;
    ip route del 10.0.0.0/24 2>/dev/null || true;
    ip route add 10.0.0.0/24 dev tun0 src 10.0.0.1 2>/dev/null || true;
    ip addr add 10.0.1.1/24 dev tun0 2>/dev/null || true;
    ip route del 10.0.1.0/24 2>/dev/null || true;
    ip route add 10.0.1.0/24 dev tun0 src 10.0.1.1 2>/dev/null || true;
    ip addr add 10.0.9.1/24 dev tun0 2>/dev/null || true;
    ip route del 10.0.9.0/24 2>/dev/null || true;
    ip route add 10.0.9.0/24 dev tun0 src 10.0.9.1 2>/dev/null || true;
    ip addr add 10.0.255.1/24 dev tun0 2>/dev/null || true;
    ip route del 10.0.255.0/24 2>/dev/null || true;
    ip route add 10.0.255.0/24 dev tun0 src 10.0.255.1 2>/dev/null || true;
    wait $$UPF_PID
  "
```
`enable_snat: yes` in `config.yaml`.

## UE configuration — 2 UEs, 4 sessions each

`ue.conf` (both UEs use the same filename; each system gets its own copy):
```
# UE 1                                    # UE 2
uicc0 = {                                 uicc0 = {
  imsi = "001010000000001";                 imsi = "001010000000002";
  key  = "fec86ba6...";                      key  = "fec86ba6...";
  opc  = "C4244936...";                      opc  = "C4244936...";
  pdu_sessions = ({                          pdu_sessions = ({
    id = 1; dnn = "oai"; nssai_sst = 1;        id = 1; dnn = "oai"; nssai_sst = 1;
    nssai_sd = 0xffffff; type = "IPV4";        nssai_sd = 0xffffff; type = "IPV4";
  }, {                                        }, {
    id = 2; dnn = "openairinterface";          id = 2; dnn = "openairinterface";
    nssai_sst = 1; nssai_sd = 0xffffff;        nssai_sst = 1; nssai_sd = 0xffffff;
    type = "IPV4V6";                            type = "IPV4V6";
  }, {                                        }, {
    id = 3; dnn = "ims"; nssai_sst = 1;         id = 3; dnn = "ims"; nssai_sst = 1;
    nssai_sd = 0xffffff; type = "IPV4V6";       nssai_sd = 0xffffff; type = "IPV4V6";
  }, {                                        }, {
    id = 4; dnn = "default"; nssai_sst = 1;     id = 4; dnn = "default"; nssai_sst = 1;
    nssai_sd = 0xffffff; type = "IPV4V6";       nssai_sd = 0xffffff; type = "IPV4V6";
  });                                         });
}                                          }
position0 = { x=0.0; y=0.0; z=6377900.0; }
@include "channelmod_rfsimu_LEO_satellite.conf"
```
UICC = Universal Integrated Circuit Card, the software representation of the
SIM/USIM in OAI.

## Launch

```bash
cd ~/oai-cn5g && docker compose up -d          # Core
docker compose down                             # to stop

cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
  --gNBs.[0].min_rxtxtime 6 -E --continuous-tx    # gNB

# On BOTH UE machines (same command, each with its own local ue.conf)
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --ue-fo-compensation \
  -E -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf
```
Each UE establishes 4 PDU sessions with distinct tunnel names, verified with
`ip a | grep oaitun`.

## Policy-based routing — per UE, per DNN

Without policy routing, all traffic defaults to Wi-Fi or the physical NIC:
```
default via 192.168.230.1 dev wlp0s20f3    (laptop, Wi-Fi)
default via 192.168.230.1 dev enp0s31f6    (PC, wired NIC)
10.0.0.0/24    oaitun_ue1
10.0.1.0/24    oaitun_ue1p2
10.0.9.0/24    oaitun_ue1p3
10.0.255.0/24  oaitun_ue1p4
```

### UE 1 — 4 routing tables
```bash
echo "200 dnn1" | sudo tee -a /etc/iproute2/rt_tables
echo "201 dnn2" | sudo tee -a /etc/iproute2/rt_tables
echo "202 dnn3" | sudo tee -a /etc/iproute2/rt_tables
echo "203 dnn4" | sudo tee -a /etc/iproute2/rt_tables

sudo ip route add default via 10.0.0.1   dev oaitun_ue1   table dnn1
sudo ip route add default via 10.0.1.1   dev oaitun_ue1p2 table dnn2
sudo ip route add default via 10.0.9.1   dev oaitun_ue1p3 table dnn3
sudo ip route add default via 10.0.255.1 dev oaitun_ue1p4 table dnn4

sudo ip rule add from 10.0.0.2   table dnn1
sudo ip rule add from 10.0.1.2   table dnn2
sudo ip rule add from 10.0.9.2   table dnn3
sudo ip rule add from 10.0.255.2 table dnn4

# Verify
ip rule
ip route show table dnn1 && ip route show table dnn2 && ip route show table dnn3 && ip route show table dnn4
```

### UE 2 — separate routing tables, same interface names, `.3` source IPs
```bash
echo "210 ue2dnn1" | sudo tee -a /etc/iproute2/rt_tables
echo "211 ue2dnn2" | sudo tee -a /etc/iproute2/rt_tables
echo "212 ue2dnn3" | sudo tee -a /etc/iproute2/rt_tables
echo "213 ue2dnn4" | sudo tee -a /etc/iproute2/rt_tables

sudo ip route add default via 10.0.0.1   dev oaitun_ue1   table ue2dnn1
sudo ip route add default via 10.0.1.1   dev oaitun_ue1p2 table ue2dnn2
sudo ip route add default via 10.0.9.1   dev oaitun_ue1p3 table ue2dnn3
sudo ip route add default via 10.0.255.1 dev oaitun_ue1p4 table ue2dnn4

sudo ip rule add from 10.0.0.3   table ue2dnn1
sudo ip rule add from 10.0.1.3   table ue2dnn2
sudo ip rule add from 10.0.9.3   table ue2dnn3
sudo ip rule add from 10.0.255.3 table ue2dnn4
```

## Traffic validation — 8 concurrent iperf3 streams

**At core+gNB host** — 8 servers (4 per UE):
```bash
sudo pkill -9 iperf3
for p in {5201..5208}; do gnome-terminal -- bash -ic "iperf3 -s -p $p"; done
ss -tlnp | grep iperf3 | sort
```

**At UE 1**:
```bash
iperf3 -c 192.168.70.129 -p 5201 -u -b 20M -t 5 -R -B 10.0.0.2    # oai
iperf3 -c 192.168.70.129 -p 5202 -u -b 20M -t 5 -R -B 10.0.1.2    # openairinterface
iperf3 -c 192.168.70.129 -p 5203 -u -b 20M -t 5 -R -B 10.0.9.2    # ims
iperf3 -c 192.168.70.129 -p 5204 -u -b 20M -t 5 -R -B 10.0.255.2  # default
```

**At UE 2** (different ports to avoid collision if run concurrently):
```bash
iperf3 -c 192.168.70.129 -p 5205 -u -b 20M -t 5 -R -B 10.0.0.3
iperf3 -c 192.168.70.129 -p 5206 -u -b 20M -t 5 -R -B 10.0.1.3
iperf3 -c 192.168.70.129 -p 5207 -u -b 20M -t 5 -R -B 10.0.9.3
iperf3 -c 192.168.70.129 -p 5208 -u -b 20M -t 5 -R -B 10.0.255.3
```

## MAC-layer confirmation

Post-traffic gNB logs confirm each PDU session mapped to a distinct LCID, active
simultaneously for both UEs:
```
[NR_MAC] Frame.Slot 640.0
UE RNTI 945d CU-UE-ID 1 in-sync PH 34 dB PCMAX 20 dBm, average RSRP -100 (16 meas)
UE 945d: LCID 4: TX 12908728 RX 8105 bytes  ------> PDU 1
UE 945d: LCID 5: TX 12888694 RX 6529 bytes  ------> PDU 2
UE 945d: LCID 6: TX 12875712 RX 6954 bytes  ------> PDU 3
UE 945d: LCID 7: TX 12936431 RX 9805 bytes  ------> PDU 4

UE RNTI ccc1 CU-UE-ID 2 in-sync PH 34 dB PCMAX 20 dBm, average RSRP -99 (16 meas)
UE ccc1: LCID 4: TX 13030885 RX 6563 bytes  ------> PDU 1
UE ccc1: LCID 5: TX 11307640 RX 5754 bytes  ------> PDU 2
UE ccc1: LCID 6: TX  1864457 RX 13023 bytes  ------> PDU 3
UE ccc1: LCID 7: TX 12929878 RX 16187 bytes ------> PDU 4
```
All four LCIDs (4–7) carrying traffic for both UEs simultaneously confirms all 8
sessions across the 2-UE setup were active and correctly isolated.

## Conclusion

This work successfully demonstrates the implementation of multiple simultaneous PDU
sessions in an OAI 5G SA network. The core and NR UE were configured to support
four DNNs per UE, resulting in independent PDU sessions with separate IP addresses
and tunnel interfaces. Policy-based routing correctly forwarded application traffic
through the intended PDU session, while parallel `iperf3` traffic validated
concurrent data transmission across all active sessions. The observed MAC-layer
statistics further confirmed that each PDU session mapped to a distinct logical
channel and was handled simultaneously by the scheduler — providing a robust
platform for evaluating QoS-aware communication, traffic isolation, network
slicing, and scheduler enhancements in future OAI-based 5G research.
