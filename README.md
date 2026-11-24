# clab-orchestrator-demo

EVPN VXLAN configs for the Arista devices of the Containerlab Orchestrator demo data.

The configs use the physical topology of the demo data: the device names, the physical port names, the cables, and
the IP addresses that `invoke generate-test-data` makes in Nautobot. The configs add only the values that the demo
data does not have: loopbacks, ASNs, and the tenant VRF.

## Files

| File | Device | Role |
| --- | --- | --- |
| `configs/dc1-spine1.cfg` | `dc1-spine1` | Spine, DCS-7280CR3-32P4 |
| `configs/dc1-spine2.cfg` | `dc1-spine2` | Spine, DCS-7280CR3-32P4 |
| `configs/dc1-leaf1.cfg` to `configs/dc1-leaf4.cfg` | `dc1-leaf1` to `dc1-leaf4` | Leaf and VTEP, DCS-7050SX3-48YC8 |

## Design

- The underlay is eBGP over the routed /31 links between the spines and the leafs.
- The overlay is eBGP EVPN between the Loopback0 addresses of the leafs and the spines.
- The spines are route servers. They keep the next hop of each EVPN route.
- Each leaf is a VTEP. Loopback1 is the VXLAN source.
- The link from each leaf to its Linux host is a routed /31 in the VRF `TENANT_A`.
- The leafs send the `TENANT_A` routes as EVPN type 5 routes with the L3 VNI `50001`.

## Addresses and ASNs

These values come from the demo data:

| Item | Value |
| --- | --- |
| Spine to leaf links | `10.1.0.0/24`, one /31 for each link. The spine gets the even address. |
| Leaf to host links | `10.1.1.0/24`, one /31 for each link. The leaf gets the even address. |
| Management | `10.20.20.0/24` on `Management1`, in the VRF `MGMT`. The gateway is `10.20.20.1`. |

These values are new. Nautobot does not have them:

| Device | ASN | Loopback0 | Loopback1 (VTEP) |
| --- | --- | --- | --- |
| `dc1-spine1` | 65100 | `10.255.0.1/32` | None |
| `dc1-spine2` | 65100 | `10.255.0.2/32` | None |
| `dc1-leaf1` | 65101 | `10.255.0.11/32` | `10.255.1.11/32` |
| `dc1-leaf2` | 65102 | `10.255.0.12/32` | `10.255.1.12/32` |
| `dc1-leaf3` | 65103 | `10.255.0.13/32` | `10.255.1.13/32` |
| `dc1-leaf4` | 65104 | `10.255.0.14/32` | `10.255.1.14/32` |

The login is `admin` with the password `admin`, the cEOS default. The configs keep only the SHA-512 hash.

## Linux hosts

Each host must send the tenant traffic to its leaf. Run these commands on the containerlab host after the deploy:

```bash
docker exec clab-demo-spine-leaf-dc1-host1 sh -c "ip addr add 10.1.1.1/31 dev eth1; ip link set eth1 up; ip route add 10.1.1.0/24 via 10.1.1.0"
docker exec clab-demo-spine-leaf-dc1-host2 sh -c "ip addr add 10.1.1.3/31 dev eth1; ip link set eth1 up; ip route add 10.1.1.0/24 via 10.1.1.2"
docker exec clab-demo-spine-leaf-dc1-host3 sh -c "ip addr add 10.1.1.5/31 dev eth1; ip link set eth1 up; ip route add 10.1.1.0/24 via 10.1.1.4"
docker exec clab-demo-spine-leaf-dc1-host4 sh -c "ip addr add 10.1.1.7/31 dev eth1; ip link set eth1 up; ip route add 10.1.1.0/24 via 10.1.1.6"
```

## Checks

Run these commands on a leaf:

```text
show bgp summary
show bgp evpn summary
show bgp evpn route-type ip-prefix ipv4
show ip route vrf TENANT_A
show vxlan vtep
```

Then do a ping from one host to another host:

```bash
docker exec clab-demo-spine-leaf-dc1-host1 ping -c 3 10.1.1.7
```

## Limits in the lab

The configs use the physical names. In a cEOS node of the lab, two names are different:

- cEOS calls the management interface `Management0`, not `Management1`. containerlab sets the management address of
  `Management0`.
- The Golden Config sources of the app change each physical name to the container name of the interface table. The
  demo table maps `Ethernet49` to `eth49`, and cEOS does not accept `eth49` in a config.
