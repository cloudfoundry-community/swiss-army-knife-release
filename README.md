## Swiss Army Knife BOSH Release

The Swiss Army Knife [†](#dagger) BOSH release enables VM customization beyond
what many single-purpose BOSH releases allow; it enables the customization of
the five [BOSH jobs lifecycle](https://bosh.io/docs/job-lifecycle/) stages.  Its
single job, `knife`, has five properties (`ctl`, `drain`, `pre_start`,
`post_start`, and `post_deploy`) which correspond to the lifecycle stages. Each
property can be set to an arbitrary shell script.

## Quick Start

In the releases section of your BOSH manifest:

```yaml
- name: swiss-army-knife
  sha1: e0ac6bdd4ae1a0674e4e0d3b5c9c3b7f8921ff62
  url: https://github.com/cunnie/swiss-army-knife-release/releases/download/1.0.0/swiss-army-knife-release-1.0.0.tgz
  version: 1.0.0
```

In the following manifest snippet, we colocate the `knife` job on the
`haproxy` instance_groups and use the `pre_start` property to set the IPv6
default route:

```yaml
instance_groups:
  name: haproxy
  jobs:
  - name: knife
    release: swiss-army-knife
    properties:
      pre_start: |
        #!/bin/bash
        ip -6 route add default via 2601:646:100:69f5:20d:b9ff:fe48:9249
```

## Notes

If you're using the Swiss Army Knife release to set an ephemeral setting (a
setting that's cleared on reboot, e.g. setting a secondary route), then use the
`ctl` property, for that's the only stage that's run on reboot.

The `ctl` script's requirements are more complex than the four other scripts':
it must accept both `stop` and `start` options, it must create the PID file
(`/var/vcap/sys/run/knife/pid`) when started, and remove the PID file when
stopped. If unfamiliar with writing `ctl` scripts, you may find it helpful to
refer to the
[`nginx-release`](https://github.com/cloudfoundry-community/nginx-release/blob/b00e3c8e9f000f7bf190a35ea930ad866bcfed2f/jobs/nginx/templates/ctl.sh)
or the
[`pdns-release`](https://github.com/cloudfoundry-community/pdns-release/blob/b3401506005398ed516e5bccfb5e6da84cc91df0/jobs/pdns/templates/ctl.erb).

A useful debugging technique is to `ssh` onto the VM and view the log files:

```
sudo tail /var/vcap/sys/log/knife/*.log
```

Running the `ctl` script (or any of the scripts) by hand may also help debug:

```
sudo bash -x /var/vcap/jobs/knife/bin/ctl start
```

The first line of the five templates should always be the
[shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)): `#!/bin/bash`.

This release does not take advantage of [bpm](https://bosh.io/docs/bpm/bpm/), a
mechanism to isolate collocated BOSH jobs. `bpm` ["will always execute your
process with de-escalated
privileges"](https://bosh.io/docs/bpm/transitioning/#executable), which is not
the use case for the Swiss Army Knife release—it's used most often to set a
system-wide setting requiring unfettered privileges (e.g. setting a `sysctl`,
defining an IP route).

If there's a BOSH release that's specifically crafted to accomplish what you
need, use that instead of the Swiss Army Knife release. For example, if you want
to set a `sysctl`, try the
[`os-conf-release`](https://github.com/cloudfoundry/os-conf-release) first. If
you want set a route, try the
[`networking-release`](https://github.com/cloudfoundry/networking-release).

You don't need to set the all the `knife` job's properties; they have
reasonable defaults. Set the ones you want and leave the rest.

### Inspiration

Our initial inspiration for the release was simple: we wanted to deploy
a HAProxy VM with both IPv4 and IPv6 default routes. You can read more about it
[here](docs/inspiration.md).

---

<a id="dagger">†</a> The Swiss Army Knife BOSH release has nothing to do
with Switzerland, its army, or its army's knives.
