# Docker IPv6 bridge with globally routable addresses

## Goal

Provide containers on a separate Docker bridge network with globally
routable IPv6 addresses from the router's existing LAN prefix, using
IPv6 forwarding plus NDP proxying on the Docker host. No NAT for IPv6.

This setup now works for outbound, host-to-container, and LAN-to-container.
A static NDP proxy entry plus `ndppd` are in place; port publishing is
required to open Docker's `ip6tables` firewall even in routed mode.

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
  Docker IPv4 subnet                     `172.19.0.0/16`
  Docker IPv4 gateway                    `172.19.0.1`
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

Note: host `0.0.0.0:80` was already bound by another service, so the
test container publishes `8080:80` instead of `80:80`. Direct access to
the container's global IPv6 address on port 80 still works; only the
host-side mapping uses 8080.

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

## Test web container

``` bash
docker run -d --name ipv6-web --network docker-ipv6 \
  --ip6 2406:6d00:b4f8:eb00:1::100 \
  -p 8080:80 \
  -v /tmp/opencode/ipv6-web/index.html:/usr/share/nginx/html/index.html:ro \
  nginx:alpine
```

Serves a static page (`IPv6 test OK`) on `:::80`. `nginx` listens on
IPv6 by default.

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
* LAN-to-container `curl -6 http://[...:1::100]/` from Windows: ok
  (after publishing the port; before that it timed out with `curl: (28)`).
* Host default route: `default via fe80::642:1aff:fe7c:6c00 dev end1 proto ra`.
* Router-advertised LAN route: `2406:6d00:b4f8:eb00::/64 dev end1 proto ra`.

## Existing Docker networks left alone

The `opencode-sandbox_default` network was not changed. The standard
`bridge`, `host`, and `none` networks were also left alone, except that
other containers on the host publish unrelated ports (`4096`, `8000`).

## Not done yet

* No persistent sysctl configuration (`/etc/sysctl.d/`).
* Static `ip -6 neigh add proxy` entry is not persistent; confirm
  `ndppd` alone suffices after reboot, then drop the static entry if so.
* Manual `DOCKER-USER` test rule (if ever added) is not persistent;
  prefer `-p` publishing, which Docker manages.
* No automatic mechanism beyond `ndppd auto` + static IPs has been
  configured for future containers. Either assign `--ip6` explicitly or
  confirm `ndppd auto` covers dynamically assigned addresses.
* Confirm whether the ISP/router prefix is stable or can change, and
  plan how to update the Docker subnet + `ndppd.conf` if it changes.
* True internet (off-LAN) inbound with a fresh connection has been
  shown for return traffic (HTTP responses reached containers), but an
  unsolicited inbound test from a VPS / mobile data to the container
  listener has not been recorded here.

## Key lessons

* NDP proxying is not DHCPv6 or prefix delegation. The host answers
  neighbor solicitations on the LAN for container addresses and forwards
  to the Docker bridge. Each container still needs a valid address and
  route.
* In Docker's `gateway_mode_ipv6=routed`, IPv6 is not NATed, but the
  published port (`-p`) is still the firewall switch. No `-p` means
  `DOCKER ... -j DROP` for TCP even though the address is routable.
* ICMP working while TCP times out is the signature of this firewall
  drop, not an ND problem.
* Avoid the router's own low addresses when carving a `/80` out of the
  LAN `/64`. The first `/80` collides with `::1`.
