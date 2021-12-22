---
title: "I declare! Personal infrastructure!"
layout: "post"
---

The time had come to rebuild my (circa 2014) VPS,
which was running Ubuntu 18.04 and couldn't be upgraded further.
Inspired by glyph's [A Tired Raccoon's Containerization Manifesto][raccoon],
I decided to try to push as much as I can into containers.
Here are a few notes about things I learned on the way.

![](/img/bankruptcy.webp)

## Declarative configuration management

Part of the ordeal of a migration is re-discovering all of the things
I had to learn in order to configure the system the first time.
This time, I'm capturing everything I'm changing
by using a configuration management tool
to set up the services on the host.
_Vim ist verboten!_

I wanted a solution where I'd be able to edit the configuration on my laptop
and just mash a button locally to synchronize the state of my VPS with the configuration.

I liked the idea of writing declarative configurations
and I don't like YAML,
so between Ansible, Chef, Puppet, and Salt,
I chose Puppet.

The onboarding documentation for these tools
tends to focus on setting up the infrastructure you need
to manage and monitor a whole fleet of servers.

Happily, Puppet's Bolt tool lets you
[apply a Puppet manifest](https://puppet.com/docs/bolt/latest/applying_manifest_blocks.html)
to any node you can ssh to,
without any persistent management or agent services,
which was exactly what I wanted.

Getting Docker services set up and working with Google Artifact Registry
required installing the Google Cloud SDK
and provisioning it with credentials from a service account.
Once that's in place, running containerized services works out of the box
with the docker module.

This is what the manifest ends up looking like: [Gist](https://gist.github.com/tdsmith/478f6346ed227e19132c1cb413821cb4)

Most of the arguments to the Docker resources end up being passed through
to the `docker` command line one way or another.

Applying the manifest is as simple as `bolt apply manifest.pp --targets myhosts`
after writing a Bolt `inventory.yaml` file.

## Certificate management

Among the services I run on my server are static Web hosting for my partner and me,
and a znc bouncer for libera.chat.
I'd like to secure connections to these services with TLS.

The old server served HTTP with Apache
and I managed TLS certificates using the Let's Encrypt client,
running a short shell script periodically to sync the refreshed certificates to znc.
We can do better!

[Caddy](https://caddyserver.com/) is an open-source,
memory-safe web server with robust reverse proxy support
and simple automatic TLS configuration
that knows how to obtain certificates with ACME out of the box.

Using the [level4 extension](https://github.com/mholt/caddy-l4),
it can happily provision certificates,
terminate TLS,
and reverse-proxy for arbitrary streams,
including IRC.

Caddy ends up being the only exposed service,
forwarding cleartext traffic to znc over the Docker bridge network.

A downside of using level4 is that you can't use it with
the friendly Caddyfile configuration DSL,
so I had to figure out how
to write configuration in Caddy's JSON configuration language.
This is partly mitigated by the `caddy adapt` utility,
which emits the equivalent JSON for a Caddyfile,
which is straightforward to graft into the proper JSON config,
so I can continue to describe my web hosting in Caddyfile.

A couple of tricks are:

* For the ZNC web interface to work,
  you have to keep Caddy from advertising http/2 availability
  in ALPN during the TLS handshake.
* Not all IRC clients send SNI, so you need to define a default SNI
  so that caddy can choose a certificate to send.

The configuration for layer4 ends up looking like:

```jsonc
{
  "apps": {
    "layer4": {
      "servers": {
        "znc": {
          "listen": [
            ":1337"
          ],
          "routes": [
            {
              "handle": [
                {
                  // Terminate TLS
                  "handler": "tls",
                  "connection_policies": [
                    {
                      "alpn": [
                        "http/1.1",
                        "http/1.0"
                      ],
                      "default_sni": "kvm.tds.xyz"
                    }
                  ]
                },
                {
                  // Forward cleartext to ZNC
                  "handler": "proxy",
                  "upstreams": [
                    {
                      "dial": [
                        "znc:2337" // Network address to proxy to
                      ]
                    }
                  ]
                }
              ]
            }
          ]
        }
      }
    }
  }
}
```

## What have we learned?

It's possible to rub some declarative configuration onto a VPS
to make administering personal infrastructure a little more predictable.

Throwing some services into a container and putting Caddy in front
is a convenient way to get automatic certificate renewal
without too many hijinks.

Death still comes for us all,
but at least migrating to a new VPS provider won't suck as much.

[raccoon]: https://glyph.twistedmatrix.com/2021/06/trash-panda-devops.html