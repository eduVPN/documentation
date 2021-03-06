## Introduction

**NOTE**: WireGuard functionality is **EXPERIMENTAL** and not available in 
any official VPN packages yet! Do NOT use in production!

## Requirements

WireGuard will only be supported on servers running Debian >= 11 and 
Fedora >= 33. Currently only Fedora 33 (x86_64, aarch64) works.

```
$ sudo dnf -y install wireguard-tools
```

Use the following repository configuration in 
`/etc/yum.repos.d/eduVPN-wg.repo`: 

```
[eduVPN-wg]
name=eduVPN with WireGuard Packages (Fedora $releasever)
baseurl=https://repo.tuxed.net/eduVPN/wg/rpm/fedora-$releasever-$basearch
gpgcheck=1
gpgkey=https://repo.tuxed.net/fkooman+repo@tuxed.net.asc
```

## WireGuard Configuration

```
$ sudo cat /etc/wireguard/wg0.conf
[Interface]
Address = 10.145.101.1/24,fd04:b3ac:674d:b32b::1/64
ListenPort = 51820
PrivateKey = (hidden)
```

To generate a private key you can use `wg genkey`. Put that in the place above
where it says `(hidden)`. In order to generate "random" IPv4 and IPv6 prefixes
to use in your configuration you can use the `vpn-server-api-suggest-ip` 
command. Make sure they match with the `rangeFour` and `rangeSix` parameters
in the portal configuration below. **NOTE**: use `.1` and `::1` in `Address` 
above and use `.0` and `::` in `rangeFour` and `rangeSix`.

To start the WireGuard interface and enable it to start on boot:

```
$ sudo systemctl enable --now wg-quick@wg0
```

Still need to figure out whether this is best way to make this persistent. 
Maybe through NetworkManager or `systemd-networkd` would be better.

## Daemon

```
$ sudo dnf -y install wg-daemon
$ sudo systemctl enable --now wg-daemon
```

You can test the interface with WireGuard out now:

```
$ curl -s http://localhost:8080/info | jq
{
  "PublicKey": "2obnZaov/Idd1zHFZqziWurRubx98ldKmDH44nB7nF0=",
  "ListenPort": 51820,
}
```

## Portal

You need to enable WireGuard in the portal configuration by modifying 
`/etc/vpn-user-portal/config.php`, you can also configure WireGuard:

```
// DEFAULT: false
'enableWg' => true,

'WgConfig' => [
    // the WireGuard interface
    // REQUIRED
    'wgDevice' => 'wg0',

    // where to reach "wg-daemon"
    // DEFAULT: http://localhost:8080
    'wgDaemonUrl' => 'http://localhost:8080',

    // the DNS to put in the configuration you send to the clients
    // DEFAULT: 9.9.9.9, 2620:fe::fe
    'dns' => ['9.9.9.9', '2620:fe::fe'],
    
    // the host name WireGuard clients will connect to
    // REQUIRED
    'hostName' => 'vpn.example.org',

    // The IPv4 range issued to WireGuard clients
    // REQUIRED
    'rangeFour' => '10.145.101.0/24',

    // The IPv6 range issued to WireGuard clients
    // REQUIRED
    'rangeSix' => 'fd04:b3ac:674d:b32b::/64',
],
```

## Firewall

We assume you are using the default firewall, some rules need to be added, 
both in the `/etc/sysconfig/iptables` and `/etc/sysconfig/ip6tables` files:

```
-A INPUT -p udp -m state --state NEW -m udp --dport 51820 -j ACCEPT
-A FORWARD -i wg+ ! -o wg+ -j ACCEPT
-A FORWARD ! -i wg+ -o wg+ -j ACCEPT
```

After modifying the firewall, restart it:

```
$ sudo systemctl restart iptables
$ sudo systemctl restart ip6tables
```

## API

The WireGuard integration also exposes an API for use by apps. It works the 
same way as the existing API, protected using OAuth. This API is in a state of 
flux and will defintely change before being rolled out in production!

The API endpoint discovery is exactly the same as documented [here](API.md). 
This documentation will eventually be merged there.

The `$(BEARER_TOKEN}` referenced below is obtained through the OAuth flow.

### Available

**NOTE**: this call will most likely be removed as there will be another 
indicator, for example in the `info.json` that indicated WireGuard support.

As WireGuard integration is currently optional, the client can determine 
whether WireGuard support is enabled on the server by requesting 
`/wg/available`, e.g.:

```
$ curl -i -H "Authorization: Bearer ${BEARER_TOKEN}" \
    https://fedora-vpn.tuxed.net/vpn-user-portal/api.php/wg/available
HTTP/1.1 200 OK
Date: Sat, 30 Jan 2021 09:43:28 GMT
Content-Type: text/plain;charset=UTF-8

y
```

### Connect

**NOTE**: this call will probably be renamed to `/wg/connect`, `/wg/add_peer`, 
`/wg/add_client` or something similar.

```
$ PUBLIC_KEY=$(wg genkey | wg pubkey)
$ curl -i -H "Authorization: Bearer ${BEARER_TOKEN}" \
    --data-urlencode "publicKey=${PUBLIC_KEY}" \
    https://fedora-vpn.tuxed.net/vpn-user-portal/api.php/wg/create_config
HTTP/1.1 200 OK
Date: Sat, 30 Jan 2021 09:45:19 GMT
Content-Type: text/plain;charset=UTF-8

[Peer]
PublicKey = 2obnZaov/Idd1zHFZqziWurRubx98ldKmDH44nB7nF0=
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = fedora-vpn.tuxed.net:51820

[Interface]
Address = 10.10.10.5/24, fd00:1234:1234:1234::5/64
DNS = 9.9.9.9, 2620:fe::fe    
```

### Disconnect

**NOTE**: this call will probably be renamed to `/wg/remove_peer`, 
`/wg/remove_client`, or something similar.

```
$ curl -i -H "Authorization: Bearer ${BEARER_TOKEN}" \
    --data-urlencode "publicKey=${PUBLIC_KEY}" \
    https://fedora-vpn.tuxed.net/vpn-user-portal/api.php/wg/disconnect
HTTP/1.1 204 No Content
Date: Sat, 30 Jan 2021 09:46:02 GMT
    
```

## App

We have an experimental Android app available for testing WireGuard. You can
download it 
[here](https://argon.tuxed.net/fkooman/eduVPN/LetsConnect-wireguard.apk).

Make sure you first connect to the server you want to use with OpenVPN and make
sure it works. Then go in the "App Settings" and enable the WireGuard" 
checkbox. Trying to reconnect at this point will use WireGuard if all goes 
well.

## TODO

- we currently have a "sync" that adds all peers to WG from DB that were 
  manually created, i.e. not through the API. This needs to be done better, 
  every 2 minutes a partial sync, only peers get added, never removed, is 
  not great... it *does* work, for now... at least we need a call for multi 
  peer add
- implement multi-profile
- implement wg-server-config.php to write `/etc/wireguard/wg*.conf` files in 
  vpn-server-node package
- add entries to `connection_log` table when peer is added/removed so we know
  who had an IP at a certain time
- prevent 1 user claiming all IPs in 2 seconds through API or web, limit to 
  maximum number of configs (also for OpenVPN perhaps...)
- clean up "dead" connections from the daemon (make the sync a *real* sync)
- show WG connections on "Connections" page
- make disable user remove/disable? WG connections
- show WG info on "Info" page
