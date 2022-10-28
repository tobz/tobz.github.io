# Self-hosting the hard way: securely exposing services on the _World Wide Web_

## Prelude

Over the past few years, there's been somewhat of a rennaissance around self-hosting services, such
as media storage and e-mail. Whatever the reasons may be -- financial, technical, moral, etc --
there's an appetite to host services on ones own infrastructure, under ones control. I myself host a
number of services on my own infrastructure, complete with a cheeky location code for my basement
"datacenter". However, most people's lives don't't exist a bubble that ends exactly where their wifi
network begins. Being able to use these services on the go is just as important as using them at
home where they might be hosted.

## Why not Tailscale?

Now, I'm a tech person, and you're probably a tech person too if you're reading this blog, and
you've potentially already muttered at the screen, bewildered: "why not just use Tailscale?".
Tailscale, in a nutshell, provides point-to-point encrypted connections to allow accessing remote
devices as if you were on the same local network, but from anywhere. It has clients for all major
computing and mobile platforms and it makes this process an absolute breeze. I already use it, but
it has one particular limitation: it's not a good answer for sharing _arbitrary_ access to
self-hosted services. Sure, the client support is there, but asking friends and family to install
Tailscale on all the devices they might want to use, and then managing that access ... that's not a
task I want to take on.

## Figuring our requirements

With all of this in mind, I set out to find a solution to the problem as I saw it, and came up with
a list of requirements:

1. Services must be accessible from "anywhere"
2. New software should not be required (i.e. you shouldn't need a VPN to access the equivalent of a
  hosted GMail... just your browser)
3. Should not be directly exposed to the internet (no port forwarding)
4. All requests must be authenticated/authorized _before_ traffic ever reaches anything in our
   infrastructure
5. Should be free (no software/hosting cost, ideally)

Over the course of a few weeks, I did a bunch of research, scouring the likes of the r/selfhosted
subreddit, popular forums for home hosting like ServerTheHome, and general Google spelunking. After
a lot of tinkering and toiling, I finally came up with a solution that checks off all the boxes,
which I'll go through below.

## Charting a course

**tl;dr: Cloudflare Tunnels for avoiding directly exposing my home infrastructre, Authentik running
on Fly.io for exposing an externally-accessible Identity Provider, and Cloudflare Access using
Authentik as an IdP for authenticating/authorizing all requests before they ever hit my network**

## Wiring up our services to the internet

Cloudflare has a large catalog of services they offer, but one of the more intriguing ones for
self-hosters is their "Tunnel" service. Cloudflare Tunnel (neé _Argo Tunnel_) provides a way to run
a small daemon within your network that establishes a reverse connection _back_ to Cloudflare's
POPs, where in turn you can expose applications inside your network across that tunnel. Similar to
configuring, say, nginx or Apache, you tell `cloudflared` an upstream target, or targets, to send
traffic to (like `localhost:8080` or `svc.mylocal.network:9000`) and then configure, on the
Cloudflare side, what public hostnames to expose those services at. Traffic hits Cloudflare's edge,
gets routed internally and then eventually across the established tunnel, and ultimately lands at
your service.

This is really, really cool and helps us check off a lot of boxes, at least partially:

- Our service is _not_ exposed directly to the internet. Attackers could still exploit RCEs in our
  service or stuff like that, but we don't have to loosen our firewall rules one bit.
- Our service is, for all intents and purposes, accessible anywhere where a person could access the
  internet. We're not concerned with impediments such as country-level firewalls or DNS blackholing
  or anything like that.
- Cloudflare Tunnel is _free_. All of the other bits -- a basic Cloudflare account, hosting DNS for
  a domain, protecting subdomains with TLS, and so on -- are also free.

I created a Cloudflare account, got a domain specifically for this purpose, and configured my
Cloudflare account for that domain. After picking out the subdomain I wanted to use, I configured
the tunnel via Cloudflare's "Zero Trust" product suite (which Tunnel now lives under) to expose a
simple "hello world" service I deployed for testing, and tested it out. It took me less than 30
minutes to get a rough and tumble deployment of `cloudflared` serving traffic from my home
infrastructure _without_ having an exposed origin. This part felt a lot like magic.

Onward!

## We need a bouncer at the entrance

While having the services safely exposed to the internet was half the gattle, I still needed an
answer to the problem of authentication/authorization. As alluded to above, I tried to imagine what
the chinks in the armor might be for a set-up like this, and naturally, software vulnerabilities
came to mind. Even if a service is fronted by Cloudflare, that seems nothing of being able to stop
arbitrary requests from being sent through that ultimately exploit a vulnerability that could lead
to an RCE.

This part took the longest by far. Let's review the steps/requirements that a given solution needs
to deal with to fulfill this:

- all requests must be authenticated/authorized _before_ traffic reaches my home infrastructure, so
  _we_ can't host any part of an authn/authz solution on said home infrastructure
- we want to let users authenticate with identities they already have, otherwise we're back to the
  example given for Tailscale, where we're constantly managing access to the access mechanism
  itself, to say nothing of managing the access to the underlying services themselves

Admittedly, I'm writing this in a different order than how I approached finding a solution because
it felt a lot like having no answer at all until all of the pieces came together. With that said,
what I ultimately landed on was based on another Cloudflare product: Cloudflare Access.

As mentioned above, Cloudflare Tunnel is part of Cloudflare's overall "Zero Trust" product suite,
which is their set of offerings for small businesses and enterprises to do the whole zero trust
thing: trusted devices with device posture management, being able to access internal resources by
currying favor granted to their trusted devices, and bringing all different manner of services into
the fold in a generic way, whether they're self-hosted (Cloudflare Tunnel) or external, by providing
authorization at the Cloudflare edge (Cloudflare Access). Their product people would probably snort
here at my crude explaining of the Zero Trust product suite, but suffice to say that it provides all
of the building blocks we need to build this solution.

Anyways, Cloudflare Access provides the authorization aspect by allowing you to specify identity
providers that users can authenticate to, which then allow you to authorize them with simple
policies on the Cloudflare side, and Cloudflare handles managing the cookies/tokens/proxying of
traffic to your services. In particular, it can sit in front of a service exposed by Cloudflare
Tunnel without the two CF products having to be configured to know of one another. As long as both
the tunnel and the Access configuration are set to handle the same specific domain, it just seems to
work. This was unexpected but _very_ nice.

## Who's on the guest list?

I knew now that Cloudflare Access was the answer to how to handle authorizing all requests before
they made it to my home infrastructure, but what I didn't know yet was _how_ to authenticate users.
I also didn't know how to authenticate them off-site, to avoid the catch-22 of traffic needing to
hit my home infrastructure first.

Long story short, some more spelunking, and some options emerged from r/selfhosted: Authentik and
Authelia. These projects are essentially "build your own identity provider" solutions. They allow
you to handle all the common aspects of running an identity provider: creating/managing users and
groups, importing users from other identity providers, doing authorization, and so on. I ultimately
chose Authentik because one of the services I run is Plex, and Authentik has support for using Plex
as an identity provider itself, which meant for services where people were already accessing my Plex
content, they could authenticate using the same identity/credentials. Further, Plex provides an API
to distinguish if a user authenticated through their identity provider has access to a specific Plex
server, which meant I could essentially get Plex-based authentication and authorization in the same
step.

Configuring Authentik was fairly painful: it took me a while to pour over the documentation, figure
out how to create my own identity provider, configure all of the various flows/stages correctly,
wire it up to Cloudflare Access, and test it. Documenting all of the steps I took, and the
configuration I landed on would be too big for this post, but is something I'm looking to post about
more in the future.. ideally with an opinionated configuration to help others start from. In the
interest of brevity, here's how I configured Authentik:

- Authentik gets configured to use Plex as a federated authentication source, further constrained to
  users that have access to my Plex server
- Authentik is configured to create shadow user accounts locally to mirror the users in Plex, and
  assigns them to a specific group, used later on in the authorization flow
- Authentik exposes a dedicated OAuth2/OpenID Provider endpoint that uses this Plex-based federated
  authentication for its own authentication flow, and authorization is a no-op on the Authentik side
  since we get it implicitly based on the federated Plex authn configuration, and since we do some
  of that in Cloudflare Access
- Authentik is also configured to send the user's Plex token in the OIDC claims so that we can pass
  it to the underlying services being protected

Cloudflare Access, in addition to the cookies it uses itself for ensuring users are authenticated
before passing thr traffic through, will also send a cookie/header to the protected application
which contains not only the basic authentication data such as username/e-mail, but also additional
data like the aforementioned OIDC claims. This allows us to curry data from Authentik, such as Plex
token or user groups, all the way to Cloudflare Access for primary authorization, but also then to
protected applications in cases where we might then need to impersonate the user, such as using
their Plex token.

## Keeping the bouncer protected, too

We've managed to figure out how to protect our home infrastructure as an origin, as well as how to
expose an authn/authz mechanism and have Cloudflare protect our resources using that, but we still
had the problem that our authn/authz provider itself needed to be externally exposed in order for
Cloudflare Access to reach it, as for OIDC, there's a number of redirect steps in the browser and so
on. Even if Cloudflare was also fronting Authentik, we still had the same potential issue: what if
Authentik has a vulnerability that can be exploited prior to getting through the authn/authz flow?

I solved this by simply... running it on external infrastructure. I had been wanting to use Fly.io
for a while -- I know one of the engineers there, it's a cool product, their blog posts are great,
and they're providing a ton of value to customers -- and this struck me as the perfect chance. Since
we don't generally need any advanced/complex redundancy or resiliency -- Plex is our primary
authn/authz source, all we need to do is have a repeatable way to configure Authentik to use it --
we could afford to run Authentik is a stripped down way on lower-end hardware. Fly.io was a great
fit for this.

Right off the bat, Fly.io lets you run applications at the edge so long as you can bundle them up
into a Docker image. Authentik already had a Docker image, so that's a solved problem. We also
needed to provide Authentik a database and cache: technically, we didn't really care about long-term
storage, in-memory storage of OAuth2 tokens, etc, would have sufficed... but Authentik is more
general-purpose than that and requires Postgres and Redis. Luckily, Fly.io has been launching
managed services, either by providing the management tooling themselves (Postgres) or acting as a
marketplace for third-party providers to run their managed services on (Redis, specifically Upstash
Redis). Oh yeah, and Fly.io has a generous free tier for all three of these core components. _Nice._

I configured the requisite Postgres and Redis services on Fly.io, then crafted a Dockerfile (and
ultimately a `fly.toml` deployment configuration) for Authentik. Authentik has a feature they call
"blueprints" which allows defining its configuration (some parts of it, at least) as YAML files that
can be loaded at start... which I didn't take advantage of initially but have been working to switch
over to in order to make the "Fly.io exploded, and I need to redeploy this from scratch" scenario
more palatable. After manually configuring Authentik to do all of the OAuth2/OpenID and Plex
federated authn/authz bits, I had one last step which was to configure some vanity DNS to stick
Authentik behind, and then a small configuration on the Cloudflare Access side to point to it.

Much elided tweaking and futzing and hair pulling later, I had Cloudflare Access using my isolated
Authentik deployment to authenticate and authorize users, before sending traffic over Cloudflare
Tunnel to an application hosted on my home infrastructure... and it was all free! (Note: I ended up
purchasing some credits from Fly.io because their service is great and I wanted to spin up some
extra application deployments with resources beyond what the free tier provides)

## Let's pretend for just a moment

I've said a lot of words above but let's look at a concrete example of how this all works together.

- I have an existing Plex server, shared with friends and family, which acts as the authentication
  (and authorization) provider. Plex already has mechanisms to share itself outside of what's
  described in this blog post, but I was exposed another application that needs Plex credentials.
- I have an application inside my network, which is accessible on-network at
  `http://app.cluster.local:5000`, which I want to expose externally at
  `https://app.vanitydomain.com`
- I created a Cloudflare account and set it up to host DNS for `vanitydomain.com`
- I subscribed/opted in/whatever-you-want-to-call-it to Cloudflare Zero Trust, and set up Cloudflare
  Tunnel to proxy traffic from `https://app.vanitydomain.com` to `http://app.cluster.local:5000`,
  and deployed `cloudflared` internally to support that
- I created a Fly.io account and deployed Authentik, using Fly.io's Managed Postgres and Managed
  Redis (Upstash Redis), which I then put behind a vanity subdomain of
  `https://app.auth.vanitydomain.com`.
- I configured Authentik to use Plex as a federated authentication (and de-facto authorization)
  source by allowing users to authenticate to Plex (Plex as in `plex.tv`, not my Plex server
  specifically) which then provides de-facto authorization as it only allows authenticated Plex
  users who also have access to my Plex server
- I configured Authentik to expose an OAuth2/OpenID Connect identity provider, which ultimately uses
  the federated Plex authn/authz source for its own authn/authz, and additionally forwards data like
  their user groups, Plex token, etc, in the OIDC claims.
- I configured Cloudflare Access with a new authentication source that was pointed at our
  Authentik-based OAuth2/OpenID IdP, located at `https://app.auth.vanitydomain.com`, with a no-op
  authorization policy, as Authentik handles that for us.
- I configured Cloudflare Access to expose/protect an application located at
  `https://app.vanitydomain.com` using the authentication source we just configured.

As a user, all we have to do is navigate to `https://app.vanitydomain.com`. When we do that for the
first time, Cloudflare Access sees that we have no existing cookie marking as as authenticated, and
so we're redirected to an account-specific Cloudflare Access service provider endpoint (part of the
OAuth2/OpenID flow) which then sends us to `https://app.auth.vanitydomain.com` where we authenticate
with Plex. Once the Plex authentication happens, and things look good, we're redirected back to the
account-specific Cloudflare Access service provider endpoint, which now does the "authorization" and
if that's successful, sends us to `https://app.vanitydomain.com` but with a specific path that
Cloudflare Access intercepts, which lets Cloudflare set site-specific authentication cookies for our
domain (neat!). Finally, we're redirected to the original resource, now with our authentication
cookies being sent with the request, which lets Cloudflare Access know we're authenticated and
allowed to access to the given resource. Cloudflare Tunnel takes over from here, routing the request
over the tunnel, with the cookie/header from the Cloudflare Access side so our application can
access whatever authn/authz data it needs to, and finally we can talk to the application.

## What we've learned/accomplished

We set out to expose an application running on our home infrastructure without having to necessarily
expose it directly to the internet, involving messing with routers and firewalls. We also set out to
allow access to that application only to authenticated/authorized users based on an identity
provider we already implicitly used and trusted.

We did all of this without _any_ traffic ever hitting our home infrastructure before the
user/request was authenticated and authorized, and without needing to host the
authentication/authorization endpoints on our home infrastructure so that Cloudflare and Fly.io bear
the full weight of any potential DDoS/intrustion attempts. If a bug in Authentik was found and
exploited, we still have potential for the authentication/authorization flow to be compromised,
leading to getting access to the protected application, but we can focus more of our time and energy
on securing the Authentik deployment rather than also having to harden every single application we
want to expose.

Finally, we did this all for free, with primarily open-source software that can be examined and
replaced if need be, saved for the Tunnel magic provided by Cloudflare.
