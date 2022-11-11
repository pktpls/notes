
# Insel ans Mesh anschließen

Ubuntu 21.10 bzw. 20.04

Tunneldigger ist ein kleines Client/Server-Tool das L2TP-Tunnel aushandelt. Der Tunneldigger-Client fragt beim Server die Parameter an und konfiguriert dann mittels netlink den Tunnel im Kernel.

Es gibt im Berliner Freifunknetz zwei Tunneldigger-Endpunkte:
- bbbdigger um Inseln ans Mesh anzuschließen
- vpndigger für private Internetgateways (wegen Störerhaftung, das interessiert uns hier erstmal nicht)

### Teil eins: Tunneldigger und OLSR bauen

```sh
apt install -y mtr-tiny git build-essential bison flex cmake libnl-3-dev libnl-genl-3-dev libgps-dev
mkdir /home/user/bbb

cd /home/user/bbb
git clone https://github.com/olsr/olsrd
cd olsrd ; make all libs ; cd -
git clone https://github.com/wlanslovenija/tunneldigger
cd tunneldigger/client/ ; cmake . && make tunneldigger ; cd -

olsrd/olsrd --help
tunneldigger/client/tunneldigger --help
```

### Teil zwei: Tunnel und OLSRd starten

Nachdem der Tunnel gestartet ist, nehmen wir `nmcli` (NetworkManager) um Adresse, Gateway und .olsr-DNS zu konfigurieren. Wenn es schnell gehen muss und DNS verzichtbar ist, reicht auch `dhclient`. Und IPv6 ist hier aus purer Faulheit deaktiviert.

Bevor OLSR gestartet wird, muss in `olsrd.conf` der Nodename (nameservice-Plugin) angepasst werden.

*Hinweis: es ist ungünstig, dass Tunneldigger und OLSR unbedingt als `root` laufen müssen.*

```sh
> sudo ./tunneldigger -f -i ff0 -u $(uuidgen -r) -b a.bbb-vpn.berlin.freifunk.net:8942

> nmcli conn add type generic ifname ff0 \ # alternativ: sudo dhclient -d ff0
    ipv4.method "auto" ipv4.dns "10.36.197.5" ipv4.dns-search "~olsr" \
    ipv6.method "disabled"
> nmcli conn up generic-ff0                # alternativ: sudo ip link set ff0 up

> sudo ./olsrd/olsrd -f olsrd.conf -nofork

> nmcli conn             # interface status
> ip addr show ff0       # adressen
> resolvectl status ff0  # dns status
```

### Teil drei: was nun?

```sh
# Beispiel: Alle BBB-VPN Clients via OLSR jsoninfo
> curl -s http://a.bbb-vpn.olsr:9090/links | jq .

# Beispiel: Traceroute zur Potse
> mtr potse-core.olsr

# Beispiel: Alle bisher gefundenen Potse-Hostnamen
> cat olsr-hosts.txt | grep potse

# Beispiel: Alle bisher gefundenen announcten Services im Mesh
> cat olsr-services.txt
```

OLSR APIs
- jsoninfo: http://fesev-core.olsr:9090
- txtinfo: http://fesev-core.olsr:2006

Docs
- jsoninfo: https://github.com/OLSR/olsrd/blob/master/lib/jsoninfo/README_JSONINFO
- txtinfo: https://github.com/OLSR/olsrd/blob/master/lib/txtinfo/README_TXTINFO
- nameservice: https://github.com/OLSR/olsrd/blob/master/lib/nameservice/README_NAMESERVICE
- NetworkManager:

Adressen
- Mit `sipcalc` können easy IP-Adressblöcke und Subnetze berechnet werden
- 10.31.0.0/16, 10.36.0.0/16 und 10.230.0.0/16 werden verwaltet von https://config.berlin.freifunk.net
- Intercity-VPN: https://wiki.freifunk.net/IC-VPN
  - (die Routen anderer Communities scheinen im BBB aber nicht announct zu werden, vielleicht hilft es, die Routen selber zu setzen und den aufgelisteten Berlin-Router als Gateway zu nehmen?)

Monitoring
- https://hopglass.berlin.freifunk.net
- https://monitor.berlin.freifunk.net
- https://t-löffel.de
