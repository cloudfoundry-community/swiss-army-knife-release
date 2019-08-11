## Inspiration

We wanted to deploy an HAProxy VM with both IPv4 and IPv6 default routes.

BOSH doesn't natively understand dual-stack (a single interface with both IPv4
and IPv6 addresses). Instead, BOSH implements IPv6 as a distinct network
interface. In other words, our VM has two ethernet interfaces, one with an IPv4
address, and one with an IPv6.  Since it has two interfaces, BOSH
[requires](https://bosh.io/docs/networks/#multi-homed) one of the interfaces to
be the default gateway. Simple choice: since our BOSH Director is IPv4-only and
since it resides on a different subnet (10.2.0.0/24) than our deployed VM
(10.0.250.0/24), the default gateway must be applied to the IPv4 interface so
that it can reach the BOSH Director when it boots.  This begs the question,
"How do we apply a default route for the IPv6 interface?".

Our first attempt was to use the [BOSH Networking
Release](https://github.com/cloudfoundry/networking-release).  In our first
pass we attempted to use the _gateway_ job, but had a shortcoming: [it
attempted to resolve every route as an IPv4
address](https://github.com/cloudfoundry/networking-release/blob/ad3930498cd4818fffa225eb13d497c879c222a7/jobs/gateway/templates/bin/gateway_ctl.erb#L9-L16).
It did not work for our IPv6 address.

We briefly considered using the _routes_ job instead, but we were unhappy with
its plethora of properties (does anyone really need to set the `mss`, `window`,
and `irtt` options?). Also, it uses the old `route` command, which as been
[obsoleted](https://www.linux.org/docs/man8/route.html) ("This program is
obsolete. For replacement check ip route.").

We shifted gears: instead of struggling to set routes, why not take advantage
of IPv6's [Neighbor Discovery
Protocol](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol) (NDP)?
The IPv6 router advertisement has been turned off in the BOSH stemcells, but we
hoped we could enable it using the _sysctl_ job in the [os-conf BOSH
release](https://github.com/cloudfoundry/os-conf-release), by setting the
`net.ipv6.conf.*.accept_ra=1` 

The success was mixed: on hand, the default IPv6 route was set. In fact, the VM
was able to pick up ephemeral IPv6 addresses. On the downside, we lost our
statically-assigned IPv6 address (a side effect of one of the
[stemcell-installed
sysctl's](https://github.com/cloudfoundry/bosh-linux-stemcell-builder/blob/741f485675f13ec3cda19d375b8f30b1aa1c584c/stemcell_builder/stages/bosh_sysctl/assets/60-bosh-sysctl.conf#L21-L22)).

We were frustrated—we needed one command entered into our HAProxy VM (`ip -6
route add default via 2601:646:100:69f5:20d:b9ff:fe48:9249`), but we couldn't
do it.

So we wrote a BOSH release which allowed us to inject arbitrary commands.
