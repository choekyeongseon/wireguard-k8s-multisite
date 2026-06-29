# FortiGate DNAT — VIP + Firewall Policy

In this design the **idc-site** gateway sits behind a FortiGate. The company-site
gateway is reachable directly on its Floating IP, so it dials the idc gateway at the
FortiGate's public IP. Inbound UDP/51820 must be DNATed to the idc gateway's
OpenStack Floating IP. Two objects are required.

## 1. VIP (Virtual IP / DNAT)

```
External IP : Port  = 198.51.100.1 : 51820
Mapped  IP : Port   = 203.0.113.20 : 51820     # idc gateway (wireguard-gw-idc) Floating IP
Protocol            = UDP
```

## 2. Firewall policy

```
Incoming interface  = any        # extintf=any — see note
Outgoing interface  = <internal interface toward the idc gateway>
Source              = all
Destination         = <the VIP object above>
Service             = UDP/51820
Action              = ACCEPT
NAT                 = disabled    # the VIP already does the DNAT
```

> [!WARNING]
> **Set the incoming interface to `any` (`extintf=any`).** If the policy is scoped
> to one interface and the inbound WireGuard UDP arrives on a different one,
> FortiGate drops it and the handshake never completes.

## Verify from the peer side

```bash
# from the company-site gateway, after both configs are up:
wg show wg0          # latest handshake should appear within seconds
ping -c 3 10.0.90.2  # idc-site tunnel IP
```
