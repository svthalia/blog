---
layout: post
title: Towards Zero-Downtime Deployments
author: jelle
---

For the thalia.nu website, we currently have quite a manual deployment method.
Although we do configuration management with Ansible, a deploy requires us to
change a variable in the config and rerun the deploy command. We have a very
simple deployment with one Docker instance running, and the Ansible
configuration change stops the container temporarily so in the meantime our
members will see a maintanance error page.

Because everything in our server config is working now, we have time to rethink our
deployment method. For the new method, I really want to move to a zero-downtime
deployment system using NixOS for configuration automization.

Because I have never done a zero-downtime deployment method before, I will explore
some thoughts in this blog post.


## Servers

We use AWS EC2 to run our application currently. The application runs in a docker
container which is part of a Docker compose file. This Docker compose file also
includes our PostgreSQL Docker, but not the NGINX server which is located on the
host for convenience mostly. For clarification, all of this is run on one EC2
instance because we do not have the traffic to justify a multi-instance setup.

For the new method I still want to use a single EC2 instance, which complicates
the zero-downtime setup. Usually these load balancer setups assume a single server
per service, so one or multiple proxy servers, one or multiple database servers and
multiple app servers. The plan is to make this easier with NixOS, because it has
a configuration system that makes a lot of things explicit which would normally be
implicit.

I want to ditch the Docker containers and replace it with systemd services,
which are restricted with the systemd security options. The containerization of
Docker is super useful to make the application more portable, and decleratively
defined. Both of these properties are retained by switching to Nix generated
systemd services. Another advantage of Docker is the security property of
containers. One part of this is the seperation of networks in Docker, but I
think this is also a downside of using Docker as iptables are automatically
added by the daemon. By switching to Nix based configuration we may loose some
seperation, but systemd also has a lot of security options so I don't think we
will be dropping a lot of security.


## Proxying

For a zero-downtime system we need a reverse proxy to balance the requests
between the current running version of the application and the new version.
Because our current system uses NGINX for TLS termination and static file
serving, we currently have the most knowledge of the NGINX reverse proxy. We
could use NGINX to balance the requests gradually to the new version, but then
we would not be able to learn anything new! So I want to try and use HAProxy as
a balancer between the two instances, and NGINX would then still run for the
static file serving and proxy to the Django app, which is done via uWSGI.

From the little research I have done into HAProxy it should be pretty easy to do
this. We could use the [unix socket] to send commands to the proxy to gradually
set the weight of the new version to 100%. For this we could create a deployment
management app in the form of a website, chat bot or maybe just command line
program.

I believe we would need to also change configuration files to accomplish this,
because the new `server` line would need to be added to the HAProxy
configuration to add the new application version. Luckily HAProxy allows a
reload of the configuration files without dropping any connections. I think the
weight changing could also be done by just writing new config and reloading it,
but the downside would be that the HAProxy master has to restart it's
subprocesses. I will have to look more into this though.

Another option for seamless weight changing would be the [HAProxy dataplane
API]. I think this requires more setup to get right though, and because we're
running on a single instance anyway, the Unix socket method would probably be
the simplest.

All in all I think this will take the most time, as with a good zero-downtime
setup we need to be careful with changing HAProxy files and not write breaking
configuration for example. We also still need to think about how our management
system is going to look like. I think a web app will be nicest, but we will see.

[unix socket]: https://cbonte.github.io/haproxy-dconv/1.6/management.html#set%20weight
[HAProxy dataplane API]: https://www.haproxy.com/blog/new-haproxy-data-plane-api/


## Migrations

When running a setup like this where two versions of the app may be running with
the same database at a time, you will have to be very careful with any
migrations. Django has a very nice migration system, but there are no builtin
checks to make sure migrations are safe to run when another version of the app
could be running. I have already made [a CI check] to notify a reviewer that a
PR makes changes in the migrations, but to make this work best we will probably
have to collect a list of migrations which are not allowed---unless the previous
version also supports this change. I think it will be very benefitial if we
could automate this, including the check for compatibility with the previous
version. But will that be possible? I'm not sure.

[a CI check]: https://github.com/svthalia/concrexit/pull/1347


## Continuous Development

If the above steps work out, I think the next step will be to make everything as
automatic as possible. When a new release is tagged on GitHub, the HAProxy
should already add the `server` line with 0 percent weight in the config so it
can be turned on at will. Afterwards we could immediately start testing it,
provided we have configured a way to override the HAProxy weight (this should be
possible, I just haven't looked up how yet). I think this is the part where
NixOS will really shine, because we will be able to build closures in GitHub
actions and push them to the server.

We have already started working on continuous deployment in the [face-detect-app
repo] it uses NixOS and automatically deploys itself on the AWS EC2 instance
when the master branch updates. We will continue to experiment with this method
for our main website.

[face-detect-app repo]: https://github.com/svthalia/face-detect-app/
