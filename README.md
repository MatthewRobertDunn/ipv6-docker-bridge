# Docker IPv6 bridge with globally routable addresses

Give containers on a separate Docker bridge network globally routable
IPv6 addresses from the router's existing LAN prefix: no NAT, no DHCPv6,
no prefix delegation. The host answers neighbor solicitations for
container addresses on the LAN (`ndppd`) and forwards between the LAN
interface and the Docker bridge. A pair of persisted `ip6tables` rules
allows all IPv6 to/from the Docker subnet, so containers need no
published ports.

## Prerequisites

* Docker host on the LAN with Docker Engine (needs
  `com.docker.network.bridge.gateway_mode_ipv6=routed` support).
* Router advertising a `/64` on-link LAN prefix (check with
  `ip -6 route | grep 'proto ra'`).
* `ndppd` (`sudo apt install ndppd` on Debian/Ubuntu).
* A second host on the LAN (or phone/VPS) to verify inbound access.

## Values used in this guide

Adapt the addresses to your LAN. The example values below are from a
working setup:

  Item                     Example value
  ------------------------ --------------------------------
  LAN interface            `end1`
  Observed LAN prefix      `2406:6d00:b4f8:eb00::/64`
  Router LAN address       `2406:6d00:b4f8:eb00::1`
  Docker network name      `docker-ipv6`
  Docker IPv6 subnet       `2406:6d00:b4f8:eb00:1::/80`
  Docker IPv6 gateway      `2406:6d00:b4f8:eb00:1::ffff`

How to pick the `/80`: carve it out of the observed on-link `/64`, but
skip the range holding the router's own addresses. The first `/80`
(`...::/80`) hands out `...::1`, which collides with a router at
`...::1` — the second `/80` (`...:1::/80`) avoids that. Do not assume
other `/64`s are routed to the host unless delegation is confirmed.

## 1. Kernel settings (persisted)

``` bash
sudo tee /etc/sysctl.d/99-docker-ipv6.conf <<'EOF'
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.end1.proxy_ndp = 1
net.ipv6.conf.end1.accept_ra = 2
EOF
sudo sysctl --system
```

Replace `end1` with your LAN interface. `accept_ra=2` keeps accepting
router advertisements while forwarding is enabled. Verify:

``` bash
sysctl net.ipv6.conf.all.forwarding net.ipv6.conf.end1.proxy_ndp net.ipv6.conf.end1.accept_ra
```

## 2. Docker network

Routed mode disables IPv6 NAT for the network:

``` bash
sudo docker network create \
  --driver bridge \
  --ipv6 \
  --subnet=2406:6d00:b4f8:eb00:1::/80 \
  --gateway=2406:6d00:b4f8:eb00:1::ffff \
  -o com.docker.network.bridge.gateway_mode_ipv6=routed \
  docker-ipv6
```

Verify with `docker network inspect docker-ipv6`. An empty bridge shows
`NO-CARRIER` / `linkdown` — normal until a container attaches.

## 3. NDP proxy (`ndppd`)

`/etc/ndppd.conf` — the rule must cover the Docker subnet:

``` text
route-ttl 30000

proxy end1 {
    router yes
    timeout 500

    rule 2406:6d00:b4f8:eb00:1::/80 {
        auto
    }
}
```

Then:

``` bash
sudo systemctl enable --now ndppd
# after any config change:
sudo systemctl restart ndppd
```

`auto` proxies the whole subnet from the routing table, so dynamically
assigned container addresses need no extra entries.

## 4. Firewall: allow all IPv6 to the Docker subnet (persisted)

Routed mode still drops forwarded TCP by default (`DOCKER ... -j DROP`,
ICMP only). These subnet-based rules (stable across network recreates,
unlike bridge names) open everything with no port mappings:

`/usr/local/sbin/docker-ipv6-firewall.sh`:

``` sh
#!/bin/sh
ip6tables -C DOCKER-USER -d 2406:6d00:b4f8:eb00:1::/80 -j ACCEPT 2>/dev/null || \
ip6tables -I DOCKER-USER -d 2406:6d00:b4f8:eb00:1::/80 -j ACCEPT

ip6tables -C DOCKER-USER -s 2406:6d00:b4f8:eb00:1::/80 -j ACCEPT 2>/dev/null || \
ip6tables -I DOCKER-USER -s 2406:6d00:b4f8:eb00:1::/80 -j ACCEPT
```

`/etc/systemd/system/docker-ipv6-firewall.service`:

``` text
[Unit]
Description=Restore custom Docker IPv6 firewall rules
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/docker-ipv6-firewall.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

``` bash
sudo chmod 755 /usr/local/sbin/docker-ipv6-firewall.sh
sudo systemctl daemon-reload
sudo systemctl enable --now docker-ipv6-firewall.service
sudo ip6tables -L DOCKER-USER -n -v
```

Alternative per-port approach: publish ports (`-p 80:80`) instead, and
Docker adds the ACCEPTs itself. The trade-off is one firewall entry per
port plus occupied host ports.

## 5. Run a container

No `-p` needed with the rules from step 4. Omit `--ip6` for an
auto-assigned address, or set one for a stable address:

``` bash
docker run -d --name ipv6-web --network docker-ipv6 \
  --ip6 2406:6d00:b4f8:eb00:1::100 \
  nginx:alpine
```

## 6. Verify

From a container (ephemeral test):

``` bash
docker run --rm --network docker-ipv6 busybox ping6 -c3 2606:4700:4700::1111
docker run --rm --network docker-ipv6 curlimages/curl -6 -s http://ipv6.icanhazip.com
```

The second command must return a container address in the Docker subnet
(e.g. `...:1::2`), not the host's address — that proves NAT is off.

From the host:

``` bash
ping6 -c3 2406:6d00:b4f8:eb00:1::100
curl -6 -s 'http://[2406:6d00:b4f8:eb00:1::100]/'
```

From another LAN host (proves ND proxy + forwarding + firewall):

``` shell
ping -6 2406:6d00:b4f8:eb00:1::100
curl -6 http://[2406:6d00:b4f8:eb00:1::100]/
```

## Podman (rootful, same LAN prefix)

Use **rootful** Podman (`sudo podman`). Rootless Podman funnels traffic
through the userspace NAT (`slirp4netns`/`pasta`), which defeats the
routed goal; rootful gives a real host bridge like Docker's. Carve a
different `/80` (`...:2::/80`) so it can't clash with Docker's
`...:1::/80`.

`podman network create` refuses subnets overlapping the host's on-link
`/64`, so write the CNI config by hand. `ipMasq: false` is the analogue
of Docker's `gateway_mode_ipv6=routed` (no NAT); the CNI `firewall`
plugin then allows the traffic with no manual rules.

`/etc/cni/net.d/99-podman-ipv6.conflist`:

``` json
{
  "cniVersion": "0.4.0",
  "name": "podman-ipv6",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni-podman1",
      "isGateway": true,
      "ipMasq": false,
      "hairpinMode": true,
      "ipam": {
        "type": "host-local",
        "routes": [{ "dst": "::/0" }],
        "ranges": [
          [
            {
              "subnet": "2406:6d00:b4f8:eb00:2::/80",
              "gateway": "2406:6d00:b4f8:eb00:2::ffff"
            }
          ]
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    },
    {
      "type": "firewall"
    },
    {
      "type": "tuning"
    }
  ]
}
```

Add the second subnet to `/etc/ndppd.conf` (same `proxy end1` block as
step 3) and restart `ndppd`:

``` text
    rule 2406:6d00:b4f8:eb00:2::/80 {
        auto
    }
```

Run and verify (busybox `ping6` may report `permission denied` from
missing capabilities; the HTTP checks below are the stronger proof):

``` bash
sudo podman run -d --name pv6-test --network podman-ipv6 docker.io/library/busybox:latest sleep 600
sudo podman exec pv6-test ip -6 addr show eth0
sudo podman exec pv6-test wget -qO- http://ipv6.icanhazip.com
```

The last command must print the container's own address
(`2406:6d00:b4f8:eb00:2::1`) — routed, NAT off. Serve something and
check from host and LAN:

``` bash
sudo podman exec pv6-test sh -c 'echo ok > /tmp/index.html; httpd -p 8000 -h /tmp'
curl -6 -s 'http://[2406:6d00:b4f8:eb00:2::1]:8000/index.html'
```

``` shell
ping -6 2406:6d00:b4f8:eb00:2::1
curl -6 http://[2406:6d00:b4f8:eb00:2::1]:8000/
```

Off-LAN (phone on mobile data) reaching the same URL confirms global
routability. No published ports and no extra firewall steps were needed:
the CNI `firewall` plugin inserts per-container `/128` ACCEPTs
automatically (visible in `sudo ip6tables -S | grep CNI-FORWARD` while a
container runs). Those rules are generated from
`/etc/cni/net.d/99-podman-ipv6.conflist` on every container start, so
there is nothing else to persist — that file is the persistent firewall
configuration for Podman.

## Troubleshooting

* LAN `ping6` works but TCP times out: firewall. Check
  `sudo ip6tables -L DOCKER -n -v` for the DROP, and confirm the
  `DOCKER-USER` ACCEPTs from step 4 are present.
* Container gets the router's own address (e.g. `...::1`): the chosen
  `/80` overlaps router addresses. Recreate the network with a different
  `/80` and update `ndppd.conf` + the firewall script.
* `-p 80:80` fails with `address already in use`: a host service owns
  port 80. Direct access to the container's own IPv6 on port 80 is
  unaffected; use another host port or skip `-p` (step 4 covers it).
* Bridge shows `NO-CARRIER` / route `linkdown` with no containers
  attached: expected, not an error.

## If the ISP prefix changes

The Docker subnet, gateway, `ndppd.conf` rules, and firewall script all
embed the prefix, as do the Podman CNI file and its `ndppd` rule. If the
ISP delegates a new prefix: recreate the Docker network, update
`ndppd.conf` (both rules) and restart `ndppd`, update the firewall
script and restart its service, rewrite the CNI file with the new
Podman `/80`, then re-verify from the LAN.
