My home internet is behind CGNAT, and it's not possible for me to
open ports to the internet. But I do want the ability to tunnel
back through my home IP address, when I'm traveling. (The clearest
example is for using Netflix, which is clever enough to reject most
VPN providers' exit IPs for purposes of geolocation.)

I had been stuck on this for a long time, until I signed up for
Mullvad VPN on a lark, and discovered that they support port
forwarding.

To set up port forwarding with Mullvad, you establish a wireguard
tunnel to a server in your configured metro. Traffic from anywhere on
the internet, to the configured port on the server's "exit" IP,
immediately starts getting delivered on the `wireguard` interface.

Forwarded port:
```
PORT=12345
```

```
MULLVAD_ENDPOINT=185.213.154.68:51820
MULLVAD_EXIT_POINT=185.213.154.68
MULLVAD_PUBKEY=BLNHNoGO88LjV/wDBa7CUUwUzPq/fO2UwcGLy56hKy4=
MULLVAD_WG=wg0
MY_MULLVAD_IP=10.68.140.184/32
MY_MULLVAD_PRIVATEKEY=/foo
ip link add dev $MULLVAD_WG type wireguard
ip addr add dev $MULLVAD_WG $MY_MULLVAD_IP scope link
wg set $MULLVAD_WG \
  private-key $MY_MULLVAD_PRIVATEKEY \
  peer $MULLVAD_PUBKEY \
  endpoint $MULLVAD_ENDPOINT \
  allowed-ips 0.0.0.0/0,::0/0 \
  persistent-keepalive 25
ip link set up dev $MULLVAD_WG
```

(We use `persistent-keepalive` in order to keep the NAT mapping in my
ISP's routers active at all times.)

Forwarded packets coming in to this interface will show up with
a destination of `$MY_MULLVAD_IP:$PORT`. Interestingly, they will be
_droped_ by the kernel by default, due to _Reverse Path Filtering_
(`rp_filter`). To get the kernel to permit a packet from X to be
delivered on `$MULLVAD_WG`, we have to set a route back to X, on the
same interface. This works:

```
ip rule add from $MY_MULLVAD_IP lookup 77
ip route add default dev $MULLVAD_WG table 77
```

All right, at this point our machine can get forwarded packets and
respond to them using the Mullvad tunnel: it essentially has a public
IP:port pair. What do we do with it?

The fowarded traffic is not authenticated or encrypted: any packets
that show up at IP:port, will be delivered to us. To make that secure,
we set up _another_ Wireguard connection. This time, our router is the
Wireguard _server_, and it listens on `$MY_MULLVAD_IP:port`.

We'll set it up such that this Wireguard server has ip
`192.168.177.1/24`, and give clients ip addresses in the same subnet.

```
MULLVAD_PUBKEY=BLNHNoGO88LjV/wDBa7CUUwUzPq/fO2UwcGLy56hKy4=
LOCAL_WG=wg1
MY_LOCAL_PRIVATE_KEY=/bar
MY_LOCAL_IP=192.168.177.1/24

ip link add dev $LOCAL_WG type wireguard
ip addr add dev $LOCAL_WG $MY_LOCAL_IP scope link
wg set $LOCAL_WG \
  private-key $MY_LOCAL_PRIVATE_KEY \
  listen-port $PORT
ip link set up dev $LOCAL_WG
```

All that remains is to set up a client, let's say my phone. On the home
router, run

```
LOCAL_WG=wg1
MY_PHONE_PUBLIC_KEY=abcdefabcdef
MY_PHONE_IP=192.168.177.100/32

wg set $LOCAL_WG \
  peer $MY_PHONE_PUBLIC_KEY \
  allowed-ips $MY_PHONE_IP
```

and on the phone, activate this Wireguard config:

```
[Interface]
PrivateKey = $MY_PHONE_PRIVATE_KEY
Address = $MY_PHONE_IP

[Peer]
PublicKey = $LOCAL_PUBLIC_KEY
AllowedIPs = 0.0.0.0/0,::0/0
Endpoint = $MULLVAD_EXIT_POINT:$PORT
```
