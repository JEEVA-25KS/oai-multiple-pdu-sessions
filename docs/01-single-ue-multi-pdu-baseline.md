# 1. Single-UE, Multi-PDU-Session Baseline

## Core network configuration (OAI CN5G)

Four things need to be checked/configured for multiple DNNs to work:

**1. SMF config** (`oai-cn5g/conf/config.yaml`):
- `smf_info` → `dnnSmfInfoList` must include every DNN in use
- `local_subscription_infos`: one block per `(slice, dnn)` pair

**2. `dnns:` block** — each DNN needs its own `ipv4_subnet` / `pdu_session_type`.

**3. UPF config** — `dnnUpfInfoList` must list the same DNNs so the UPF accepts
sessions for them.

**4. UDM subscription (MySQL)** — if using the DB (not only SMF locals),
`SessionManagementSubscriptionData.dnnConfigurations` must include every DNN the UE
may request. Multiple DNN keys in one JSON object for the same `ueid` is the pattern
for "this subscriber may open more than one type of session":

```sql
-- oai-cn5g/database/oai_db.sql
INSERT INTO `SessionManagementSubscriptionData` (`ueid`, `servingPlmnid`, `singleNssai`, `dnnConfigurations`) VALUES
('001010000000001', '00101', '{"sst": 1, "sd": "FFFFFF"}','{"oai":{...},"ims":{...}}');
```
Align the IMSI in the UE's UICC block with the row used in the database.

### UPF Docker entrypoint — routing for each DNN's subnet

`oai-cn5g/docker-compose.yaml` — the `oai-upf` service needs a custom entrypoint
adding a route for each DNN's subnet on `tun0`:

```yaml
oai-upf:
  container_name: "oai-upf"
  image: oaisoftwarealliance/oai-upf:develop
  expose:
    - 2152/udp
    - 8805/udp
  volumes:
    - ./conf/config.yaml:/openair-upf/etc/config.yaml
  environment:
    - TZ=Europe/Paris
  entrypoint: /bin/bash -c \
    " /openair-upf/bin/oai_upf -c /openair-upf/etc/config.yaml &
      UPF_PID=$$!;
      sleep 15;
      ip addr add 10.0.1.1/24 dev tun0 2>/dev/null || true;
      ip route del 10.0.1.0/24 2>/dev/null || true;
      ip route add 10.0.1.0/24 dev tun0 src 10.0.1.1 2>/dev/null || true;
      wait $$UPF_PID
    "
  depends_on:
    - oai-nrf
    - oai-smf
  cap_add:
    - NET_ADMIN
    - SYS_ADMIN
  cap_drop:
    - ALL
  privileged: true
  networks:
    public_net:
      ipv4_address: 192.168.70.134
```
Also enable SNAT in `config.yaml`: `enable_snat: yes`.

## UE configuration (OAI nrUE)

**Note**: multiple PDU sessions currently only work with OAI's own nrUE, not COTS
phones.

`targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf`:
```
uicc0 = {
  imsi = "001010000000001";
  key  = "fec86ba6eb707ed08905757b1bb44b8f";
  opc  = "C42449363BBAD02B66D16BC975D77CC1";
  pdu_sessions = ({
    id = 1;
    dnn = "oai";
    nssai_sst = 1;
    nssai_sd = 0xffffff;
    type = "IPV4";
  }, {
    id = 2;
    dnn = "openairinterface";
    nssai_sst = 1;
    nssai_sd = 0xffffff;
    type = "IPV4V6";
  });
}
position0 = { x = 0.0; y = 0.0; z = 6377900.0; }
@include "channelmod_rfsimu_LEO_satellite.conf"
```

## Launch sequence

```bash
# Core
cd ~/oai-cn5g && docker compose up -d
docker compose down   # to stop

# gNB
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
  --gNBs.[0].min_rxtxtime 6 -E --continuous-tx

# UE (with USRP B210)
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --ue-fo-compensation \
  -E -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf
```
The UE establishes 2 PDU sessions with different tunnel names. Check with:
```bash
ip a | grep oaitun
```

## Tunnel interfaces — UE side vs core side

**UE side** — `oaitun_ue1`, `oaitun_ue1p2` (etc.):
- created by `nr-uesoftmodem` when a PDU session is established
- Linux TUN interfaces; the UE application reads/writes IP packets here
- `oaitun_ue1` → PDU session 1 (DNN `oai`); `oaitun_ue1p2` → PDU session 2 (DNN `openairinterface`)
- traffic on these gets GTP-U encapsulated by the UE software and sent over the air to the gNB

**Core side** — `tun0` (inside the UPF container):
- created by the OAI UPF process at startup
- the UPF process reads decapsulated UE packets from here and writes DL packets to it
- in this setup, `tun0` handles **both** PDU sessions (assigned two IPs: `10.0.0.1` and `10.0.1.1`)
- ideally there'd be a separate `tun0`/`tun1` per session, but this OAI UPF version only creates one — see [doc 2](02-teid-and-gtp-tunneling.md) for how sessions still stay separated

**Between UE and gNB**: radio bearers (one DRB per PDU session).
**Between gNB and UPF**: GTP-U tunnels over UDP port 2152 on the N3 interface — each
PDU session has its own GTP TEID.

## Basic policy routing (1 UE, 2 sessions)

Without policy routing, all traffic defaults to Wi-Fi/the physical interface:
```
default via 192.168.230.1 dev wlp0s20f3   <- Wi-Fi, not a PDU session
10.0.0.0/24  oaitun_ue1
10.0.1.0/24  oaitun_ue1p2
```
To route traffic through the PDU sessions instead:
```bash
echo "200 dnn1" | sudo tee -a /etc/iproute2/rt_tables
echo "201 dnn2" | sudo tee -a /etc/iproute2/rt_tables
sudo ip route add default via 10.0.0.1 dev oaitun_ue1   table dnn1
sudo ip route add default via 10.0.1.1 dev oaitun_ue1p2 table dnn2
sudo ip rule add from 10.0.0.2 table dnn1
sudo ip rule add from 10.0.1.2 table dnn2
ip rule
ip route show table dnn2
```

## Validation
```bash
iperf3 -c 192.168.70.129 -p 5201 -u -b 20M -t 5 -R -B 10.0.0.2
iperf3 -c 192.168.70.129 -p 5201 -u -b 20M -t 5 -R -B 10.0.1.2
```
Both sessions carried traffic independently, confirming the 2-session baseline
works end-to-end.
