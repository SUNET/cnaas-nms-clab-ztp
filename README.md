# CNaaS Integration Lab

Containerlab topology for integration-testing
[CNaaS NMS](https://github.com/SUNET/cnaas-nms) with Arista switches,
including Zero Touch Provisioning (ZTP) of an access switch.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│ Host machine                                         │
│                                                      │
│ Containerlab (cnaas_mgmt network, 10.100.2.0/24)     │
│  ┌────────────────────┐        ┌────────────────────┐│
│  │ eosdist1 (cEOS)    ├──eth3──│ eosdist2 (cEOS)    ││
│  │ 10.100.2.11        │        │ 10.100.2.12        ││
│  │ 10.100.2.101 (Eth1)│        │ 10.100.2.102 (Eth1)││
│  │ 10.100.3.101 (Lo1) │        │ 10.100.3.102 (Lo1) ││
│  └────────────────────┘        └────────────────────┘│
│           │ eth2                     eth2 │          │
│           │          ┌─────────┐          │          │
│           └──────────┤eosaccess├──────────┘          │
│                      │ (vEOS)  │                     │
│                      │ ZTP, no │                     │
│                      │ config  │                     │
│                      └─────────┘                     │
│   h1 (linux host) — static routes to dist switches   │
└──────────────────────────────────────────────────────┘
```

`eosdist1` acts as a DHCP relay (Vlan1, `ip helper-address 10.100.2.2`)
so that the CNaaS DHCP server can provision `eosaccess` via ZTP.

### Nodes

| Node | Kind | Image | Management IP | Role |
|------|------|-------|---------------|------|
| `eosdist1` | `arista_ceos` | `ceos:4.33.6M` | `10.100.2.11` / `10.100.2.101` (Eth1) | Distribution switch 1 |
| `eosdist2` | `arista_ceos` | `ceos:4.33.6M` | `10.100.2.12` / `10.100.2.102` (Eth1) | Distribution switch 2 |
| `eosaccess` | `arista_veos` | `vrnetlab/arista_veos:4.33.6M` | DHCP (ZTP) | Access switch |
| `h1` | `host` | Linux | — | Static routes |

### Links

| Endpoint A | Endpoint B | Purpose |
|------------|------------|---------|
| `eosdist1:eth3` | `eosdist2:eth3` | Inter-distribution link |
| `eosdist1:eth2` | `eosaccess:eth2` | Downlink (LACP/Port-Channel3) |
| `eosdist2:eth2` | `eosaccess:eth3` | Downlink (LACP/Port-Channel3) |
| `eosdist1:eth1` | `mgmt-net` | Extra management port |
| `eosdist2:eth1` | `mgmt-net` | Extra management port |

### ZTP flow

1. `eosaccess` boots with ZTP enabled (via the patched vrnetlab image) and
   no startup config (`suppress-startup-config: true`)
2. It sends a DHCP request on Vlan1
3. `eosdist1` relays the request (`ip helper-address 10.100.2.2`) to the
   CNaaS DHCP server on the management network
4. The DHCP server provides an IP and a ZTP configuration URL
5. `eosaccess` downloads and applies the configuration, reaching DISCOVERED state

## Prerequisites

- [containerlab](https://containerlab.dev/) (runs with `sudo`)
- Docker
- Nested virtualization enabled on the host/VM (required for vEOS)

## Step 1 — Import the cEOS image

Download a cEOS64 tar archive from
[arista.com](https://www.arista.com/en/support/software-download)
(requires a free account) and import it into Docker:

```bash
docker import cEOS64-lab-4.33.6M.tar.xz ceos:4.33.6M
```

See the [containerlab cEOS guide](https://containerlab.dev/manual/kinds/ceos/)
for more details.

## Step 2 — Build the ZTP-enabled vEOS image

The access switch uses a [vrnetlab](https://github.com/srl-labs/vrnetlab)
vEOS image. Containerlab's `suppress-startup-config` flag alone is not
enough to make vEOS enter ZTP mode, so the vrnetlab build itself must be
patched to enable ZTP on the disk image and skip the default bootstrap
configuration.

### 2a — Clone vrnetlab and place the vEOS image

```bash
git clone https://github.com/srl-labs/vrnetlab.git
cd vrnetlab/arista/veos
```

Download `vEOS-lab-4.33.6M.vmdk` from
[arista.com](https://www.arista.com/en/support/software-download) and place
it in the `vrnetlab/arista/veos/` directory.

### 2b — Patch the Makefile to enable ZTP

The upstream `docker-pre-build` target in `Makefile` writes
`DISABLE=True` to the disk image, which disables ZTP. Change it to
`DISABLE=False` instead. The diff looks like this (exact lines may vary
between upstream commits):

```diff
-    if [ "$$ZTPOFF" != "DISABLE=True" ]; then \
-      echo "Disabling ZTP" && docker run --rm -it ... write /zerotouch-config "DISABLE=True"; \
-    fi
+    if [ "$$ZTPOFF" != "DISABLE=False" ]; then \
+      echo "Disabling ZTP" && docker run --rm -it ... write /zerotouch-config "DISABLE=False"; \
+    fi
```

In short: replace `DISABLE=True` with `DISABLE=False` in both the
comparison and the `write` command.

### 2c — Patch launch.py to skip bootstrap config

In `docker/launch.py`, comment out `bootstrap_config()` in the
`bootstrap_spin` method. This prevents vrnetlab from applying its own
initial configuration, allowing the switch to stay in ZTP mode:

```diff
-                self.bootstrap_config()
+                #self.bootstrap_config()
                 self.startup_config()
```

### 2d — Build the image

```bash
make
```

This produces the Docker image `vrnetlab/arista_veos:4.33.6M`.

## Step 3 — Deploy the lab

```bash
sudo containerlab deploy
```

To redeploy from scratch (wipes node state):

```bash
sudo containerlab deploy --reconfigure
```

### Host static routes

The CNaaS backend needs routes to reach the switch loopbacks and VLANs
through the distribution switches. The `h1` node sets these up
automatically inside the lab, but if you need them on the host itself:

```bash
sudo ip route add 10.0.6.0/24 via 10.100.2.101
sudo ip route add 192.168.0.0/24 via 10.100.2.101
sudo ip route add 10.100.3.101/32 via 10.100.2.101
sudo ip route add 10.100.3.102/32 via 10.100.2.102
```

> **Note:** `sudo containerlab destroy` removes these routes.
> `sudo containerlab deploy --reconfigure` preserves them.

## Usage

```bash
# Inspect the lab
containerlab inspect

# SSH into a cEOS dist switch
ssh admin@clab-cnaas-integration-eosdist1

# CLI on the vEOS access switch
docker exec -it clab-cnaas-integration-eosaccess Cli

# Bash shell on the vEOS access switch
docker exec -it clab-cnaas-integration-eosaccess bash

# Serial console (useful for watching ZTP progress)
# Find the eosaccess management IP with: containerlab inspect
telnet <eosaccess-mgmt-ip> 5000
```

### Tear down

```bash
sudo containerlab destroy
```
