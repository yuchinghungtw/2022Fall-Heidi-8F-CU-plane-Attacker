# C/U-Plane Attacker User Guide

:::info
**Outline:**
[TOC]

**Others:**
All E2E system information you can find in this book link!
:book:  [BMW Lab E2E Integration Book](https://hackmd.io/@Summer063/ryDBN4Qpn/https%3A%2F%2Fhackmd.io%2FCb9sqY3pRYup-FFadV4GqQ%3Fview)


:::


## Topology
![image](https://hackmd.io/_uploads/SJr14bpDA.png)

## Fronthaul Gateway Setting
- **Do remember use cross-over or attacker will not work properly!**

![image](https://hackmd.io/_uploads/rJfDQbawC.png)


## Attack Guide
In order to facilitate quick testing, I put all malformed packets into a folder.

> [Implementation Note for attacker](https://hackmd.io/DfXUD2-TQfaJrkILY-Oc6w?view), this note is write the code how to manipulate the packet .

### Step1. Go to the malformed packets folder

- OAI gNB and LiteON
```bash=
cd /home/oai/heidi/heidi_ufi/PacketField/PacketField_U_ulink_OAI_LO
```

- TY gNB and NKG

```bash=
cd /home/oai/heidi/heidi_ufi/PacketField/PacketField_U_ulink_TY_NKG
```

### Step2. Use Tcpreplay to send the packet

```bash=
timeout 30 tcpreplay -i enp3s0f1 -K --loop=0 -M 1 u_plane_frameId.pcap
```

:::warning
This command uses tcpreplay to replay a packet capture file. Here's a breakdown:

-   `timeout 30`: Limits the execution time to 30 seconds
-   `tcpreplay`: The tool used to replay network traffic
-   `-i enp3s0f1`: Specifies the network interface to send packets on
-   `-K`: Keeps the source MAC addresses from the capture file
-   `--loop=0`: Continuously loops the replay (0 means infinite)
-   `-M 1`: Sets the multiplier for packet timing to 1 (original speed)
-   `u_plane_frameId.pcap`: The packet capture file to replay

:::
