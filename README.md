# Docker IPv6 bridge with globally routable addresses

## Goal

Provide containers on a separate Docker bridge network with globally
routable IPv6 addresses from the router's existing LAN prefix, using
IPv6 forwarding plus NDP proxying on the Docker host. No NAT for IPv6.

This setup now works for outbound, host-to-container, and LAN-to-container.
A static NDP proxy entry plus `ndppd` are in place; all IPv6 to/from
the Docker subnet is allowed via persisted `DOCKER-USER` rules, so no
port publishing is needed.

## Network details

  Item                                   Value
  -------------------------------------- -----------------------------
  Docker host                            `orange`
  Physical LAN interface                 `end1`
  Router link-local gateway              `fe80::642:1aff:fe7c:6c00`
  LAN IPv6 prefix observed from router   `2406:6d00:b4f8:eb00::/64`
  Router's LAN IPv6 address              `2406:6d00:b4f8:eb00::1`
  Docker network                         `docker-ipv6`
  Docker bridge interface                `br-d649a3a4a92b`
  Docker IPv6 subnet                     `2406:6d00:b4f8:eb00:1::/80`
  Docker bridge IPv6 gateway             `2406:6d00:b4f8:eb00:1::ffff`
  Test web container                     `ipv6-web` at `...:1::100`, `nginx:alpine`

The router's LAN prefix length was shown in its UI as `/56`, but the
host's router advertisements install `2406:6d00:b4f8:eb00::/64` as the
on-link LAN prefix. This configuration uses that observed `/64`, with
the second `/80` (`...:1::/80`) carved out for Docker. The first `/80`
(`...::/80`) was tried first and abandoned: Docker assigned containers
`...::1`, which collides with the router's own LAN address `...::1`.
Do not assume that other `/64`s from the apparent `/56` are routed to
the host unless delegation or routing is confirmed.

## Host IPv6 settings applied (runtime only)

These commands were run:

``` bash
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv6.conf.end1.proxy_ndp=1
sudo sysctl -w net.ipv6.conf.end1.accept_ra=2
```

`accept_ra=2` is important: it allows the host to continue
accepting router advertisements on `end1` while IPv6 forwarding is
enabled.

**Persistence warning:** these `sysctl -w` changes are runtime settings.
They are lost on reboot. No persistent sysctl configuration has been
created yet.

## Docker network (current, working)

The first network used `2406:6d00:b4f8:eb00::/80` with gateway `::ffff`.
It was removed for two reasons: container `::1` collided with the
router `::1`, and Docker masqueraded outbound IPv6 to the host address
(internet saw the host address, not the container address).

The replacement uses a non-conflicting `/80` and routed mode (no NAT):

``` bash
sudo docker network create \
  --driver bridge \
  --ipv6 \
  --subnet=2406:6d00:b4f8:eb00:1::/80 \
  --gateway=2406:6d00:b4f8:eb00:1::ffff \
  -o com.docker.network.bridge.gateway_mode_ipv6=routed \
  docker-ipv6
```

Network ID:

``` text
d649a3a4a92bcef0657f8d40cb856d9d36389d019ff20d7f6aa9002693560014
```

In routed mode Docker does **not** NAT IPv6, but you still need to
publish container ports. Publishing tells Docker's firewall which
inbound traffic to allow. Without it, `ip6tables` `DOCKER` chain is:

``` text
-A DOCKER -o br-d649a3a4a92b -p ipv6-icmp -j ACCEPT
-A DOCKER ! -i br-d649a3a4a92b -o br-d649a3a4a92b -j DROP
```

That is why LAN `ping6` worked while LAN `curl` to port 80 timed out.
Publishing the port adds the needed TCP ACCEPT.

The final approach instead allows all IPv6 to/from the Docker subnet via
`DOCKER-USER` (see below), so containers need no `-p` mappings at all for
IPv6. Direct access to a container's global address works on any port.

## Firewall: allow all IPv6 to the Docker network (persistent)

`DOCKER-USER` is checked before Docker's `DOCKER` chain DROP, so one pair
of subnet rules (stable across network recreates, unlike bridge names)
opens the whole subnet with zero host port bindings:

`/usr/local/sbin/docker-ipv6-firewall.sh`:

``` sh
#!/bin/sh
ip6tables -C DOCKER-USER -d 2406:6d00:b4f8:eb00:1::/80 -j ACCEPT 2>/dev/null || \
ip6tables -I DOCKER-USER -d 2406:6d00:b4f8:eb00:1::/80 -j ACCEPT

ip6tables -C DOCKER-USER -s 2406:6d00:b4f8:eb00:1::/80 -j ACCEPT 2>/dev/null || \
ip6tables -I DOCKER-USER -s 2406:6d00:b4f8:eb00:1::/80 -j ACCEPT
```

`/etc/systemd/system/docker-ipv6-firewall.service` (enabled):

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

Verified live:

``` text
Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     0    --  *      *       2406:6d00:b4f8:eb00:1::/80  ::/0
    0     0 ACCEPT     0    --  *      *       ::/0                 2406:6d00:b4f8:eb00:1::/80
```

## NDP proxying with ndppd

`/etc/ndppd.conf`:

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

Service is running (`systemctl status ndppd`, `ndppd` 0.2.4). The rule
was updated from the old `...::/80` to the new `...:1::/80` when the
network was recreated, and the service was restarted.

A static kernel proxy entry was also added during debugging and is
currently present:

``` bash
sudo ip -6 neigh add proxy 2406:6d00:b4f8:eb00:1::100 dev end1
ip -6 neigh show proxy
# 2406:6d00:b4f8:eb00:1::100 dev end1 proxy
```

`ip -6 neigh show proxy` being otherwise empty is normal with `ndppd`
in `auto` mode (it answers in userspace). The static entry is not
persistent; after a reboot, verify LAN reachability with `ndppd` alone.

## Test web container (removed after validation)

During testing this served a static page (`IPv6 test OK`) on `:::80`
(`nginx` listens on IPv6 by default). Both test containers (`ipv6-web`
at `...:1::100`, `ipv6-allports` at `...:1::101`) and the pulled images
(`nginx:alpine`, `busybox`, `curlimages/curl`) were removed afterwards;
no test containers remain. To recreate without any published ports (the
firewall rules above already allow everything):

``` bash
docker run -d --name ipv6-web --network docker-ipv6 \
  --ip6 2406:6d00:b4f8:eb00:1::100 \
  nginx:alpine
```

Omit `--ip6` for an auto-assigned address; Docker IPAM assigns the next
free `...:1::/80` address with default via `...:1::ffff`.

## Checks completed

All from the final `...:1::/80` + routed network:

* Container has `2406:6d00:b4f8:eb00:1::1/80` (ephemeral) or static
  `...:1::100`, default via `...:1::ffff dev eth0`.
* Outbound `ping6 -c3 2606:4700:4700::1111`: 3/3 ok from containers.
* Outbound HTTP `wget http://[2606:4700:4700::1111]/`: ok.
* Source IP from internet is the container address, not the host:
  `curl -6 http://ipv6.icanhazip.com` and `http://ifconfig.co` from
  `docker-ipv6` returned `2406:6d00:b4f8:eb00:1::2` (ephemeral test).
  Host returns a different `...:51bc:...` address. NAT is off.
* Host-to-container `curl -6 'http://[...:1::100]/'`: ok.
* LAN-to-container `ping -6 ...:1::100` from Windows: ok.
* LAN-to-container `curl -6 http://[...:1::100]/` from Windows: ok.
  Final validation was with zero published ports, via the `DOCKER-USER`
  subnet rules (earlier it timed out with `curl: (28)` before the firewall
  was opened).
* Host default route: `default via fe80::642:1aff:fe7c:6c00 dev end1 proto ra`.
* Router-advertised LAN route: `2406:6d00:b4f8:eb00::/64 dev end1 proto ra`.

## Key lessons

* NDP proxying is not DHCPv6 or prefix delegation. The host answers
  neighbor solicitations on the LAN for container addresses and forwards
  to the Docker bridge. Each container still needs a valid address and
  route.
* In Docker's `gateway_mode_ipv6=routed`, IPv6 is not NATed, but
  `ip6tables` still drops forwarded TCP by default. Either publish ports
  (`-p`) per container or, as done here, ACCEPT the whole subnet in
  `DOCKER-USER` (persisted via systemd) and skip `-p` entirely.
* ICMP working while TCP times out is the signature of this firewall
  drop, not an ND problem.
* Avoid the router's own low addresses when carving a `/80` out of the
  LAN `/64`. The first `/80` collides with `::1`.
