+++
author = "Ricardo Rocha"
date = "2020-11-16T16:00:00+00:00"
description = "Making work from home simpler when VPNs are not an option"
tags = ["linux", "teleworking", "kubernetes"]
title = "A VPN-less Teleworking Setup"
+++

Working from home became frequent enough since March this year to justify an improved
home work setup (both hardware and software).

Setting up a VPN would be an easy and obvious choice but this is currently not an
option at work. Alternatively and after suggestions from multiple
people a combination of ssh / proxy setup and a handy browser
extension and the result is close enough.

Ignore all the GSSAPI\* below if you don't require Kerberos or similar access.

## SSH

Like most people most of my work requires access to the internal network.
Without a VPN a configuration like the one below is a good option.
```cfg
CanonicalDomains cern.ch
CanonicalizeHostname yes

Host *.cern.ch !IGNORE*.cern.ch !IGNOREOTHER*.cern.ch
    User MYUSER
    GSSAPIDelegateCredentials yes
    ProxyJump PROXYHOST.cern.ch
```

The important bit here is
[ProxyJump](https://man.openbsd.org/ssh_config.5#ProxyJump) where we pass the host that serves as
a sort of *bastion host* - we'll need a similar machine available and open externally.

As all connections get forwarded via this host, this next bit of config is very
useful.
```cfg
Host PROXYHOST PROXYHOST.cern.ch
    HostName PROXYHOST.cern.ch
    User MYUSER
    ControlPath ~/.ssh/controlmasters/%r@%h:%p
    ControlMaster auto
    ControlPersist 10m
    GSSAPIAuthentication yes
    GSSAPIDelegateCredentials yes
    GSSAPITrustDNS yes
    DynamicForward 8011
    ExitOnForwardFailure no
```

The relevant bit is in the
[ControlMaster](http://man.openbsd.org/ssh_config.5#ControlMaster)
which will multiplex outgoing connections over a single connection - in this
case our bastion host. Start a session to this host and keep it going and new
sessions will speed up significantly, and will drop the need to do 2FA on every
session (if 2FA is enabled in your bastion host).

We can check the control session status.
```bash
ssh -O check PROXYHOST
Master running (pid=3007)
```

and manually stop it - every now and then it might hang and need to be manually
stopped, sometimes we might need to cleanup some defunct ssh processes as
well.
```bash
ssh -O stop PROXYHOST
```

## SOCKS Proxy

The second relevant directive above is
[DynamicForward](http://man.openbsd.org/ssh_config.5#DynamicForward) 
which is setting up 8011 as a local port for a SOCKS Proxy. We can rely on it
for the command line or browser - [SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif?hl=en)
is a handy chrome extension.

{{< figure src="/images/blog/switchyprofile.png"
    title="" width="100%" >}}

## Kubernetes

Last but not least a lot of my day is spent accessing internal Kubernetes
clusters. Since Kubernetes 1.19 the kubectl client supports a *proxy-url*
option taking a SOCKS proxy.
```cfg
$ cat config
- cluster:
    certificate-authority-data: ...
    server: https://IPOFCLUSTER:6443
    proxy-url: socks5://localhost:8011

$ kubectl get pod
```

Alternatively we could:
```bash
export https_proxy=socks5://localhost:8011
```

which will do the job but is not ideal if we have a mix of internal and
external (public cloud) clusters we need to deal with.

