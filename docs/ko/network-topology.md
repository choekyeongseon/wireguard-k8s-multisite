# 네트워크 토폴로지 & 주소 계획

> English: [`docs/en/network-topology.md`](../en/network-topology.md)

## 주소 계획

| 세그먼트 | CIDR / IP | 비고 |
|----------|-----------|------|
| company-site 내부 | `10.10.0.0/24` | K8s 워커 + 게이트웨이 `eth0` |
| idc-site 내부 | `10.20.0.0/24` | K8s 워커 + 게이트웨이 `ens3` |
| WireGuard 터널 | `10.0.90.0/30` | point-to-point, 사용 가능 주소 2개 |
| company 게이트웨이 터널 IP | `10.0.90.1/30` | `wireguard-gw`의 `wg0` |
| idc 게이트웨이 터널 IP | `10.0.90.2/30` | `wireguard-gw-idc`의 `wg0` |
| company Floating IP | `203.0.113.10` | WireGuard 직접 엔드포인트 |
| idc Floating IP | `203.0.113.20` | FortiGate 뒤 DNAT 대상 |
| FortiGate 공인 IP | `198.51.100.1` | idc 게이트웨이 앞단 VIP |

## 논리 토폴로지

![토폴로지](../../diagrams/topology.ko.png)

## 라우트 설계

### 게이트웨이

각 게이트웨이는 `AllowedIPs`를 통해 자기 사이트 서브넷과 peer 터널 주소를
광고/라우팅합니다. `wg-quick`이 설정된 `AllowedIPs`에 대한 라우트를 자동으로
설치해 줍니다.

- company peer `AllowedIPs`: `10.0.90.2/32, 10.20.0.0/24`
- idc peer `AllowedIPs`: `10.0.90.1/32, 10.10.0.0/24`

### 호스트 / 워커

호스트에는 로컬 게이트웨이를 향한 원격 서브넷 static route가 **하나**만 있으면
됩니다. 이 라우트는 원격으로 *먼저 연결을 거는* 호스트에만 필요합니다.

```bash
# company-site 호스트
ip route add 10.20.0.0/24 via 10.10.0.10

# idc-site 호스트
ip route add 10.10.0.0/24 via 10.20.0.10
```

Rocky Linux / NetworkManager에서 영구 적용하려면 다음과 같이 합니다.

```bash
nmcli con mod <connection> +ipv4.routes "10.20.0.0/24 10.10.0.10"
nmcli con up <connection>
```

> [!TIP]
> 노드를 하나씩 설정하는 대신 OpenStack으로 전 노드에 일괄 배포할 수 있습니다.
> ```bash
> openstack subnet set --host-route destination=10.20.0.0/24,gateway=10.10.0.10 <subnet>
> # 노드는 다음 DHCP 갱신/재부팅 시 적용됩니다
> ```

## 소스 NAT (리턴 경로)

각 게이트웨이는 **원격 서브넷에서 들어온** 트래픽을 로컬 LAN으로 포워딩할 때
source-NAT를 적용합니다. 그러면 로컬 목적지 호스트가 게이트웨이를 소스로 보고
응답하므로, 목적지 호스트에는 리턴 라우트가 필요 없습니다.

```
company gw (eth0):  -s 10.0.90.0/30 -j MASQUERADE ; -s 10.20.0.0/24 -j MASQUERADE
idc gw     (ens3):  -s 10.0.90.0/30 -j MASQUERADE ; -s 10.10.0.0/24 -j MASQUERADE
```

![Source-NAT 리턴 경로 — 게이트웨이가 원격 소스를 자신으로 치환하여, 로컬 호스트가 리턴 라우트 없이 on-link로 응답](../../diagrams/source-nat.ko.png)

> [!NOTE]
> masquerade 대상 소스는 게이트웨이 자신의 서브넷이 아니라 **원격** 서브넷(과 터널
> `/30`)이라는 점이 핵심입니다. 이렇게 해야 응답이 게이트웨이를 거쳐 되돌아옵니다.
> 전체 `PostUp`/`PreDown` 룰은 [`configs/wireguard/`](../../configs/wireguard)를
> 참고해 주세요.
