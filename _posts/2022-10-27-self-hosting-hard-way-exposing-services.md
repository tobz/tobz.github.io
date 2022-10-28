# Self-hosting the hard way: securely exposing services on the _World Wide Web_

## Prelude

Over the past few years, there's been somewhat of a rennaissance around self-hosting services, such
as media storage and e-mail. Whatever the reasons may be -- financial, technical, moral, etc --
there's an appetite to host services on ones own infrastructure, under ones control. I myself host a
number of services on my own infrastructure, complete with a cheeky location code for my basement
"datacenter". However, most people's lives don't exist a bubble that ends exactly where their wifi
network begins. Being able to use these services on the go is just as important as using them at
home where they might be hosted.

**tl;dr: Cloudflare Tunnel for avoiding directly exposing my home infrastructre, Authentik running
on Fly.io for exposing an externally-accessible Identity Provider, and Cloudflare Access using
Authentik as an IdP for authenticating/authorizing all requests before they ever hit my network**

## Why not Tailscale?

Now, I'm a tech person, and you're probably a tech person too if you're reading this blog, and
you've potentially already muttered at the screen, bewildered: "why not just use Tailscale?".
[Tailscale](https://tailscale.com/), in a nutshell, provides point-to-point encrypted connections to
allow accessing remote devices as if you were on the same local network, but from anywhere. It has
clients for all major desktop and mobile platforms and using it is an absolute breeze. Tailscale is
already part of my homelab ecosystem, but it has one particular limitation: it's not good for
sharing _arbitrary_ access to self-hosted services. Sure, the client support is there, but asking
friends and family to install Tailscale on all the devices they might want to use, and then managing
that access ... that's not a task I want to take on.

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

Over the course of a few weeks, I did a bunch of research, scouring the likes of the
[r/selfhosted](https://www.reddit.com/r/selfhosted/) subreddit, popular forums for home hosting like
[ServeTheHome](https://forums.servethehome.com/index.php), and general Google spelunking. After a
lot of tinkering and toiling, I finally came up with a solution that checks off all the boxes, which
I'll go through below.

## Wiring up our services to the internet

Cloudflare has a large catalog of services they offer, but one of the more intriguing ones for
self-hosters is their "Tunnel" service.
[Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/) (neé _Argo Tunnel_) provides a way
to run a small daemon within your network that establishes a reverse connection _back_ to
Cloudflare's POPs, where in turn you can expose applications inside your network across that tunnel.
Similar to configuring, say, nginx or Apache, you point `cloudflared` at an upstream target (or
targets) to send traffic to (like `localhost:8080` or `svc.mylocal.network:9000`) and then
configure, on the Cloudflare side, what public hostnames to expose those services at. When traffic
hits Cloudflare's edge, it gets sent across the established tunnel, and ultimately lands at your
service.

This is really, really cool and helps us check off a lot of boxes, at least partially:

- Our service is _not_ exposed directly to the internet. Attackers could still exploit RCEs in our
  service or stuff like that, but we don't have to loosen our firewall rules one bit.
- Our service is, for all intents and purposes, accessible to anyone with an internet connection.
  We're not concerned with impediments such as country-level firewalls, DNS blackholing, corporate
  proxies, etc.
- Cloudflare Tunnel is _free_. All of the other bits -- a basic Cloudflare account, hosting DNS for
  a domain, protecting subdomains with TLS, and so on -- are also free.

> Admittedly, I started looking at Cloudflare Tunnel before Cloudflare's absolute fumbling with the
> whole Kiwi Farms situation. In no uncertain terms: fuck Kiwi Farms and all of the dipshits who
> gave it life.
>
> If there was another company providing a service equivalent to **Cloudflare Tunnel**, I'd use it.
> Until then, I'm not actually giving Cloudflare money, and I'm mostly trying to provide a service
> to friends and family in a secure way.
>
> If you know of such a service, I'm all ears!

I repurposed my existing Cloudflare account to host the DNS for my quirky self-hosting domain (this
one!). The setup instructions for Cloudflare Tunnel and `cloudflared`, the daemon that runs in your
infrastructure, are short but straightforward. I spun up a simple "Hello world!" app on one of my
servers, and ran `cloudflared` via Docker. After a small bit of configuration in the Zero Trust
dashboard to create a new subdomain and associate it with a target on my side of the tunnel, we were
serving traffic... except I had misspelled "world" as "wordl", so now my low-grade dyslexia was
publically -- albeit _securely_ -- on display to the whole... "wordl".

All told, this proof of concept took less than 30 minutes. Honestly, it felt a lot like magic. I was
on a roll at this point.

Onward!

## We need a bouncer at the entrance

While having the services safely exposed to the internet was half the battle, I still needed an
answer to the problem of authentication/authorization. As I alluded to above, I was trying to
imagine what the chinks in the armor might be for a set-up like this, and naturally, software
vulnerabilities came to mind. Even if a service is fronted by Cloudflare, an attacker can still get
requests to the service. I write software for a living, and I know just how much code is out there,
waiting for someone to walk by and notice how trivially it can be eviserated.

I needed a way to actually protect the service before traffic was allowed through, by authenticating
and authorizing users. I considered the security of the applications themselves as out of scope
here:

- generally, I trust the people to whom I've given/will give access to
- some of these applications have a `node_modules` folder that would make a security researcher
  either salivate or run away from screaming... so I chose to value my sanity and ignore delving too
  deep on the application side

With that said, again, I tried to be regimented and came up with a list of requirements/invariants
for doing authentication/authorization:

- all requests must be authenticated/authorized _before_ traffic reaches my home infrastructure, so
  _we_ can't host any part of an authn/authz solution on said home infrastructure
- we want to let users authenticate with identities they already have, otherwise we're back to the
  Tailscale problem, where we're constantly managing not only access to be on the same tailnet, but
  to the self-hosted applications themselves (I don't want a 'local network has admin' authorization
  scheme)

Admittedly, I'm writing this in a different order than how I approached finding a solution because
it felt a lot like having no answer at all until all of the pieces came together. With that said,
what I ultimately landed on was based on another Cloudflare product: 
[Cloudflare Access](https://www.cloudflare.com/products/zero-trust/access/).

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
Tunnel without the two CF products having to be configured to know about each other. As long as both
the tunnel and the Access configuration are set to handle the same specific domain, it just seems to
work.... the authentication/authorization happens first, and then it continues on with tunneling the
traffic.

Honestly, a lot like magic.

## Who's on the guest list?

I knew now that Cloudflare Access was the answer to how to handle authorizing all requests before
they made it to my home infrastructure, but what I didn't know yet was _how_ to authenticate users.
I also didn't know how to authenticate them off-site, to avoid the catch-22 of traffic needing to
hit my home infrastructure first.

Some more spelunking later, I uncovered a name that kept showing up while sifting **r/selfhosted**:
[Authentik](https://goauthentik.io/) and [Authelia](https://www.authelia.com/). These projects, and
others like them, are essentially "build your own identity provider" solutions. They allow you to
handle all the common aspects of running an identity provider: creating/managing users and groups,
importing users from other identity providers, doing authorization, and so on. I ultimately chose
Authentik because one of the services I run is Plex, and Authentik has support for using Plex as an
identity provider itself, which meant for services where people were already accessing my Plex
content, they could authenticate using the same identity/credentials. Further, Plex provides an API
to distinguish if a user authenticated through their identity provider has access to a specific Plex
server, which meant I could essentially get authorization for free in some cases. If a user should
only get access to an application because it's related to their access to my Plex server, then
authentication and authorization through Plex becomes one and the same.

Configuring Authentik was fairly painful: it took me a while to pour over the documentation, figure
out how to create my own identity provider, configure all of the various flows/stages correctly,
wire it up to Cloudflare Access, and test it. Documenting all of the steps I took, and the
configuration I landed on would be too big for this post, but is something I'm looking to post about
more in the future.. ideally with an opinionated configuration to help others start from. In the
interest of brevity, here's how I configured Authentik:

- Authentik gets configured to use Plex as a federated authentication source, further constrained to
  users that have access to my Plex server
- Authentik is configured to create shadow user accounts locally to mirror the users in Plex, and
  assigns them to a specific group (not required, but for my own sanity)
- Authentik exposes a dedicated OAuth2/OpenID Connect endpoint that uses this Plex-based federated
  authentication for its own authentication flow, and authorization is a no-op on the Authentik side
  since we get it implicitly based on how the Plex authentiation works
- Authentik is configured to send the user's Plex token in the OIDC claims so that we can pass it to
  the underlying services being protected

Cloudflare Access, in addition to the cookies it uses itself for ensuring users are authenticated
before passing thr traffic through, will also send a cookie/header with a
[JSON Web Token](https://jwt.io/) of the user's information. You get the common stuff from the
common OpenID scopes -- `username`, `email`, yadda yadda -- but you can also shove in custom fields
from the OIDC claims -- which you can customize on the Authentik side -- into the JWT. This means
not only can we validate the JWT on our side to make sure Cloudflare was really involved, but that
we can shuttle along custom data -- like the authenticated uer's Plex token, which we need for
forward authentication -- in the JWT as well.

## Keeping the bouncer protected, too

At this point, I managed to figure out how to protect my home infrastructure when used as an origin
server, as well as how to expose an authentication/authorization mechanism and have Cloudflare
protect our resources with it, but we still had the problem that our identity provider itself needed
to be pubically accessible in order for Cloudflare Access to reach it. Even if Cloudflare was also
fronting Authentik, we still had the same potential issue: what if Authentik has a vulnerability
that can be exploited prior to getting through the authentication/authorization flow?

I solved this by simply... running it on external infrastructure. I had been wanting to use
[Fly.io](https://fly.io) for a while -- I know one of the engineers there, it's a cool product,
their blog posts are great, and they're providing a ton of value to customers -- and this struck me
as the perfect opportunity. Since we don't generally need any advanced/complex redundancy or
resiliency -- Plex is our primary authentication/authorization source, so all we need to do is have
a repeatable way to configure Authentik to use it -- we could afford to run Authentik is a stripped
down way on lower-end hardware. Fly.io was a great fit for this.

Right off the bat, Fly.io lets you run applications at the edge so long as you can bundle them up
into a Docker image. Authentik already had a Docker image, so that's a solved problem. We also
needed to provide Authentik a database and cache: technically, I didn't really care about long-term
storage of anything since we could suffice with in-memory storage of OAuth2 tokens, etc... but
Authentik is more general-purpose than that and _requires_ Postgres and Redis.

Luckily, Fly.io has been launching managed services, either by providing the management tooling
themselves (Postgres) or acting as a marketplace for third-party providers to run their managed
services on (Redis, specifically Upstash Redis). Oh yeah, and Fly.io has a generous free tier for
not only deployed applications, but also for these managed services. _Nice._

I configured the requisite Postgres and Redis services on Fly.io, then crafted a Dockerfile (and
ultimately a `fly.toml` deployment configuration) for Authentik. Authentik has a feature they call
"blueprints" which allows defining its configuration (some parts of it, at least) as YAML files that
can be loaded at start... which I didn't take advantage of initially but have been working to switch
over to. Blueprints are the starting point of "how do I reconfigure this if my Fly.io account
explodes somehow?", or if I want to migrate all of this to another cloud provider.

After manually configuring Authentik to do all of the OAuth2/OpenID and Plex federated
authentication bits, I had one last step which was to configure some vanity DNS to stick Authentik
behind, and then a small configuration on the Cloudflare Access side to point to it. Much elided
tweaking and futzing and hair pulling later, I had Cloudflare Access using my isolated Authentik
deployment to authenticate and authorize users, before sending traffic over Cloudflare Tunnel to an
application hosted on my home infrastructure... and it was all free!

> I ended up purchasing some credits from Fly.io because their service is great and I wanted to spin
> up some ex tra application deployments with resources beyond what the free tier provides.
> Authentik also ended up using a lot of memory when running its Django-based migrations, which
> would cause OOMs.
>
> Ultimately, it should cost me like $5-7/month for my upsized VMs, but I'm still passively looking
> into possible performance optimizations that could be contributed upstream to Authentik that would
> allow dropping back down to the free tier VM sizing.

## Let's pretend for just a moment

I've said a lot of words above but let's look at a concrete example of how this all works together.

- I have an existing Plex server, shared with friends and family, which acts as the authentication
  (and authorization) provider. Plex already has mechanisms to share itself (network-wise) outside
  of what's described in this blog post, but I was exposing another application that needs Plex
  credentials.
- I have an application inside my network, which is accessible on-network at
  `http://app.cluster.local:5000`, which I want to expose externally at
  `https://app.vanitydomain.com`.
- I created a Cloudflare account and set it up to host DNS for `vanitydomain.com`.
- I set up Cloudflare Tunnel to proxy traffic from `https://app.vanitydomain.com` to
  `http://app.cluster.local:5000`, and deployed `cloudflared` internally to support that.
- I created a Fly.io account and deployed Authentik, using Fly.io's Managed Postgres and Managed
  Redis (Upstash Redis) services, which I then put behind a vanity subdomain of
  `https://app.auth.vanitydomain.com`.
- I configured Authentik to use Plex as a federated authentication (and de-facto authorization)
  source by allowing users to authenticate to Plex (Plex as in `plex.tv`, not my Plex server
  specifically) which then provides de-facto authorization as it only allows authenticated Plex
  users who also have access to my Plex server.
- I configured Authentik to expose an OAuth2/OpenID Connect endpoint, which ultimately uses the
  federated Plex authentication source as a passthrough, and additionally forwards data like their
  user groups, Plex token, etc, in the OIDC claims.
- I configured Cloudflare Access with a new authentication source that was pointed at our
  Authentik-based OAuth2/OpenID IdP, located at `https://app.auth.vanitydomain.com`, with a no-op
  authorization policy, as Authentik handles that for us.
- I configured Cloudflare Access to expose/protect an application located at
  `https://app.vanitydomain.com` using the authentication source we just configured.

As a user, all they have to do is navigate to `https://app.vanitydomain.com`. When they do that for
the first time, Cloudflare Access sees that they have no existing cookie marking them as
authenticated, and so they enter the authentication and authorization flow. Cloudflare Access
redirects to an account-specific Cloudflare Access service provider endpoint (part of the
OAuth2/OpenID flow) which then sends them to `https://app.auth.vanitydomain.com` where they
authenticate with Plex. Once the Plex authentication happens, and things look good, they're
redirected back to the account-specific Cloudflare Access service provider endpoint, which now does
the "authorization" and if that's successful, sends them to `https://app.vanitydomain.com`, but with
a Cloudflare Access-specific URI path. This specific path is handled by Cloudflare, but crucially,
provides a secure mechanism for cookies to be set on our application domain, by Cloudflare, on our
behalf. (neat!) Finally, the user is redirected to the original resource, now with their
authentication cookies being sent with the request, which lets Cloudflare Access know they're
authenticated and authorized to access to the given resource. Cloudflare Tunnel takes over from
here, routing the request over the tunnel, with the cookie/header from the Cloudflare Access side so
our application can access any of the authentication/authorization data that's relevant, and finally
the user is interacting with the application.

## What we've learned/accomplished

I set out to expose an application running on my home infrastructure without having to necessarily
expose it directly to the internet, messing with routers and firewalls. I also set out to only allow
authorized users to access that application, based on federated authentication/authorization that
these users had already onboarded with and that I ultimately controlled.

I was able to do all of this without _any_ traffic ever hitting my home infrastructure before the
user/request was authenticated and authorized, and without needing to host the
authentication/authorization endpoints on the very same home infrastructure, such that Cloudflare
and Fly.io bore the full weight of any potential DDoS/intrustion attempts. If a bug in Authentik was
found and exploited, there's the potential for the authentication/authorization flow to be
compromised, leading to getting access to the protected application... but I can focus more of our
time and energy on securing the Authentik deployment rather than also having to harden every single
application I want to expose.

Finally, but just as importantly: I was able to do this all for free*, with primarily open-source
software that can be examined and replaced if need be, save for the Tunnel/Access magic provided by
Cloudflare.

All in all, not the most convoluted infrastructure I've ever spun up, but sure as heck one of the
more useful bits of infrastructure I've ever set up.

More to come.
