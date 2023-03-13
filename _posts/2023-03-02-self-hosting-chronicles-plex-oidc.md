# Self-hosting chronicles: plex-oidc, or just writing everything yourself

When I originally designed my home network, in conjunction with allowing outside access to some
internal apps, [Authentik][authentik] was the lynchpin of providing the authentication/authorization
part of the equation. It's configurable, supports forwarding to Plex as an authentication and
pseudo-authorization provider, and it's open source. It was a solid choice to build things the way I
wanted, knowing that I could at least fork it if I needed to tweak its behavior if it went closed
source, and so on.

While I got Authentik configured to perform the task at hand, it didn't come without some
[hardships][hardships] along the way. However, I figured once I had gotten it installed, that would
be that: no more hardships, nothing else to tweak. Over the course of a few months of usage, though,
the rough edges had built up and it got me thinking about how to shore up that corner of the
infrastructure.

## Issues with Authentik

### Hard to customize styling

This one was superficial, but given the amount of time I spent configuring Authentik just to do the
necessary authentication/authorization, it was frustrating not being able easily change the styling
of various flows.

Authentik normally allows you to configure the "flow background" which is just the background image
displayed on a given flow. It also has a pre-existing stylesheet import in its HTML templates for
custom CSS, so long as you can put a CSS file at the right path on disk. Unfortunately, due to
Authentik's UI being heavily based on custom components using the shadow DOM, they cannot be styled
simply by injecting a CSS stylesheet at the head of the HTML document.

This limitation made it effectively impossible to style the user-facing flows without manually
overriding the various HTML templates _and_ various bits of bundled JavaScript and CSS. Doing so
would not be conducive to updating to new versions of Authentik without having to make sure our
changes still applied cleanly and hadn't broken UI functionality.

> Author's note: There was an [issue][authentik_2693] filed for this shortcoming which now
> theoretically has a fix, which is great!

### High memory overhead

I ran Authentik on [fly.io][fly_io], which notably has a free tier: up to 3 virtual machines with 1
shared vCPU and 256MB of memory. This should be more than enough for a simple web application that
serves maybe 10-15 requests a day at most, and yet, I had to go above and beyond the free tier to
run Authentik.

Fundamentally, Authentik and its requirements meant we needed to deploy three applications, and thus
use up all of the allocatable free tier VMs: one for Authentik itself, one for Redis, and one for
Postgres. This would be fine if the free timer VM size but adequately, but you can probbably guess
where this is going...

Generally speaking, Redis was able to fit within the free tier constraints because it's already
designed to be configured with memory limits. As far as Postgres, it was also usually able to fit
within limits but during its own startup, as well as Authentik's startup, it would often hit the
memory limits and OOM itself. The amount of data stored was less than 5MB total. Still, between
pre-allocated buffers and various queries made by Authentik, it often seemed to manage to just crest
the memory limits long enough to be hit by the OOM killer. Authentik itself was the significant
outlier, frequently ballooning up to 350-400MB idle RSS over time.

Due to this, I had to increase the size of the two VMs used for Postgres and Authentik to have 512MB
of memory. Increasing the memory allocation knocked me out of the free tier. As I've said in
previous posts, fly.io is awesome and I'm okay with paying them, but it certainly rubbed me the
wrong way: the application is idle nearly 100% of the time, and yet I had to bump up the VM
size, and incur a monthly bill, just because it was a hungry hungry hippo for memory those few times
it happened to serve a request every week.
### Fundamental flow-based design led to a janky browser experience

While Authentik's flow-based design is excellent from a composability standpoint, being able to
build complex flows with branching logic and so on, my needs were far simpler: always show users the
button to login via Plex on the landing page, and when the login pop-up modal closes because they
successfully authenticated to Plex, the landing page should be able to send all the data back to
Authentik, verify that the user was authenticated and authorized and so on, and then immediately
respond with the data required to trigger a browser redirect back to the OIDC SP (service provider,
Cloudflare Access in this case) right then and then.

In practice, Authentik added many redirect steps that were also just needlessly slow.

The authorization flow always starts with a landing page that provides a button to trigger a login
modal pop-up. This is necessary because without the login modal being triggered from a direct user
action, modern browsers will stop the pop-up, as it was performing a cross-domain request. Alright,
so we're always stuck having to load a page where the user needs to at least click the login button.

Once a user successfully authenticated with Plex, the modal closes, and Authentik does its first
redirect to its authentication page, where a little flash banner quickly pops up to say "you're
authenticated!". In reality, the page is saying the user is authenticated to Authentik itself, and
not exactly Plex: while we authenticate the user via Plex, we have to create a user in Authentik to
map the Plex user to, and then force the user to be authenticated with Authentik. Understandable,
but slightly clunky for our usecase.

After that, the user is redirected _again_ because now we have to authorize them, and again, they
see a flash banner pop-up that they were successfully authorized. As a user, you may now be confused
on what the hell the login pop-up did if you had to be redirected to all of these other pages. Most
users don't understand (nor should they need to) the difference between their successful
"authentication" and their successful "authorization."

Technically speaking, though, this was suboptimal because authenticating to Plex was also how we
authorized the user: do they have access to a specific Plex server? Authentik's design, though,
didn't allow this to be collapsed: we had to authenticate the user with Plex first and then
essentially start back at the very top of a traditional authentication/authorization flow.

### Slow OIDC performance led to confusion on if things were "stuck"

On the final redirect, where the user was told they were now authenticated, a redirect would
eventually be generated to send the user back the OIDC SP. This involved generating signed tokens
and other OIDC-related bits, which is fine and dandy. This part was frustratingly slow, though. due
to some inefficiencies in Authentik around loading and handling signing keys.

From a user perspective, the main problem was that this delay in processing created confusion around
whether or not the user was successfully authenticated/authorized, and if they were supposed to be
seeing the actual application. Of course, they only needed to wait a few seconds and then the page
would finally load, and they would be redirected to the protected resource. On a page that appears
to be fully loaded, though, with no progress indicator, having to wait three or four seconds can
certainly cause the user to think something has gone wrong.

Admittedly, I traced this down to signing keys being reloaded multiple times in a single request,
making a response that should have taken one second to return sometimes take nearly three to four
seconds instead. I tweaked a local checkout of Authentik to find and fix this inefficiency, but
admittedly, I never contributed it back upstream. I do still plan to.

## Crafting the minimum viable solution: `plex-oidc`

All of this led me to think: could I just write my own solution?

In theory, all we need to do is make the same API calls to Plex to handle the Plex authentication
flow, and then query for the Plex user's server associations to handle the authorization. Sprinkle
some OIDC on top of that to use it from Cloudflare Access, and voilà!

My day job involves writing Rust, and I had already written a [ForwardAuth service for working with
Cloudflare Access tokens][cloudflare-access-forwardauth] based on Rust, which has to deal with OIDC
tokens.. so I had a good idea of how I'd approach this and decided to just sketch something out and
see how far I could get in a sitting.

### Handling the basics

Most of this is somewhat rote, but I started out with an equivalent skeleton based on
`cloudflare-access-forwardauth`, which uses [`tracing`][tracing] for logging/tracing, [`axum`][axum]
for the web "framework" -- routing, request handlers, parsing headers/query parameters, etc. -- and
a cast of other helper crates: [`axum_sessions`][axum_sessions] for handling sessions,
[`tower_http`][tower_http] for adding per-request logging and CORS,
[`tower_governor`][tower_governor] for rate limiting, [`include_dir`][include_dir] for handling
static content, and a few others. We're also using [`tokio`][tokio] as the executor, since `axum` is
based on [`hyper`][hyper], which is built on `tokio`.

This let us rig up the basics of our service: async runtime, logging, exposing an HTTP API with
sensible observability and hardened defaults. I additionally built out some of the configuration
types which I used with [`serde`][serde] to deserialize our service configuration from a
configuration file on disk.

Again, more or less what you'd consider the "basics."

### Handling OIDC

Next, I needed to handle the OIDC flow that this service would be a part of. I'm using Cloudflare
Access to front all of my exposed self-hosted services, which meant that I only needed to handle the
standard "authorization code" flow.

Cloudflare Access hits the "authorize" endpoint of my service, at which point we trigger the Plex
login flow. If the user successfully authenticates with Plex, and they're authorized to access our
Plex server, we redirect back the user back to the redirect URI provided in the authorization flow
request. Finally, Cloudflare Access calls the "token" endpoint of the service behind-the-scenes to
retrieve a token for the now-authorized user, and if that goes well, the user is redirected to the
intended resource, and we're done.

I used the [`openidconnect`][openidconnect] crate to build the OIDC routes and handle all of the
relevant bits:
- a configuration endpoint which details which OAuth 2.0 flows are available, what scopes can be
  requested, what signing algorithms we can use, and so on
- a JWKS (JSON Web Key Set) endpoint, which describes which signing keys are valid in terms of
  verifying a signed JWT (JSON Web Token) returned by the service
- the authorize and token endpoints, which start the authorization flow and allow retrieving the
  token of an authorized user, respectively

`openidconnect` is more geared towards building OIDC clients, rather than servers, but all of the
necessary and relevant primitives exist in the crate. This did mean, however, that we had to lean a
lot of the Authentik source itself, and [random blog posts][random_oidc_blog] and [application
docs][auth0_oidc_docs] from Google on OIDC flows -- including a surprisingly [straightforward set of
OIDC docs][ibm_oidc_docs] from IBM (go figure) -- to understand what each step needed to do,
functionally. RFCs are utterly dense, and I find them impenetrable for understanding things
holistically like "ok, how does OIDC work?".

Thanks, random bloggers, for all of your OIDC explanations.

I also used an in-application, in-memory cache to hold all of our pending authorization flow data.
Most full solutions (Authentik, Keycloak, etc.) will actually put this stuff in a database for you,
since they rightfully expect that things such as refresh tokens will be, you know, refreshed over
time... and that you want to keep authorized users authorized even if the service restarts. My
stakes are lower -- authorizing friends and family to access media apps -- so I went with the
simplest possible approach.

I used [`moka`][moka] for this, which is an easy-to-use and capable caching crate. `moka` also uses
[`quanta`][quanta], one of my crates for providing cross-platform access to fast, high-precision
timing sources like TSC. Something something subscribe to my Soundcloud... but really, I wanted to
support people who support me. Thanks `moka`! 👋

### Authenticating and authorizing users via Plex

The final piece of the puzzle was getting the users authenticated with Plex, which we then piggyback
on to do authorization. Plex's API lets you query if a user has access to a particular server, so
the flow ends up looking something like:

- user authenticates to Plex
- once authenticated, we can query their friend list to see if a specified user ID (our user, the
  admin user/owner of the given Plex server) is contained
- if contained, they are authorized, otherwise, they're not authorized

This is pretty straightforward, so I whipped up some helper types to use for (de)serializing request
and response payloads and wrote a simple client wrapper based on [`reqwest`][reqwest], which was a
bit simpler to deal with than using `hyper` directly. If I was on a budget, maybe in terms of
reducing transitive dependencies or maximizing performance or memory usage or something, I might use
`hyper` directly to only do _exactly_ what was needed, but that wasn't a concern here.

### Flexing those frontend muscles and writing some HTML/CSS/JS

Ultimately, the service needs to show at least one page to the user where the Plex login flow itself
is triggered, which means we need to render some HTML and provide some JS. As mentioned early on, I
wasn't particularly happy with the lack of styling capabilities exposed by Authentik so I took my
time here to tweak things exactly how I wanted them.

We don't use any sort of CSS framework, or any JS libraries, or build tooling. All of the HTML, CSS,
and JS was hand-rolled. Yes, it's possible to write HTML, CSS, and JS that return a decent-looking
page on modern browsers, across multiple device types and screen sizes, without having to possess
otherworldly knowledge. Could I have whipped up the UI much faster if I used something like
[Tailwind CSS][tailwind_css]? Yes. Would my static assets be much smaller and faster to load if I
used a build pipeline like [Webpack][webpack] or [Vite][vitejs] to bundle and minify and tree shake
and all of that? Yes.

Is what I have more than acceptable for my use case? Unquestionably "yes."

Using `include_dir`, we bundle these static assets directly into the binary. They get served
normally via a dedicated path for static assets, which means they can then be trivially CDN-ified,
but ultimately, they're just part of the binary, which is one less thing to need to faff about with
during deployment.

### Putting it all together

Now with the service written, and the various phases complete, the lifecycle of authorizing a user
looks like this:

- a user navigates to a resource protected by Cloudflare Access
- the user is not authorized, so they're redirected to the OIDC authorization endpoint
- we initialize the authorization flow for the user, and return the landing page with sparkly
  HTML/CSS
- the user clicks a button to pop-up a login modal for Plex
- the landing page polls the Plex API on an interval to figure out when the user has finally
  authenticated with Plex
- once they've been authenticated, the landing page sends the user's Plex token back to the service
  to continue the authorization flow
- we actually authorize the user by checking if the configured Plex server is shared with them
- if they're authorized, we redirect back to Cloudflare Access with the relevant OIDC authorization
  code
- Cloudflare Access calls the OIDC token endpoint to exchange the authorization code for their
  access/ID token and verifies that the tokens came from us
- if all of that succeeded, they're eventually redirect to the underlying resource, with their
  Cloudflare Access JWT token (which is where `cloudflare-access-forwardauth` would take over)
- fin!

## Mission accomplished?

Simply put: yes.

### Improved user experience

The authentication/authorization flow now feels like you're really only hitting a single page -- the
landing page -- and then you're being whisked off to the protected resource.

In contrast, a user going through the Authentik flow would see the landing page, and then the
authentication flow page, and then the authorization flow page. Add to that fact that the
authorization flow page sat there for a while due to the aforementioned slowness with generating
OIDC-related responses, and now things feel vastly snappier.

In practice, Cloudflare Access itself is always going to do some redirects -- two before the user
gets to the landing page, and two after we send the user to the OIDC redirect URI -- but those pages
load quickly, which is no surprise given that they live on the Cloudflare side. Perhaps most
importantly, they load/display no user-visible content, so again, the user only ever feels like they
load a single page -- the landing page -- and then they're redirected and it spins ever so briefly
before the protected resource is loaded for them.

This is a testament to going with a hand-rolled solution to optimize for the requirements at hand.
This isn't so much a knock against Authentik itself, but for this use case, it was entirely overkill
and proved to provide a suboptimal UX.

### Improved styling

Not a whole lot to say here. We control all of the HTML/CSS styling now, so we can (and did) do
whatever we want with it.

### Improved memory overhead

We were able to go from three deployed Fly apps (Authentik, Postgres, Redis) down to one app
(`plex-oidc`). We were also able to switch back to the smallest VM size -- 1 shared vCPU, 256MB
memory -- and `plex-oidc` idles at around 30MB RSS.

Our bill is now comfortably within the free tier for Fly.io, so we pay nothing to host it. Woot!

## Was it worth it?

While I definitely achieved the things I wanted to achieve, it's still worth considering: *was it
worth it?*

I spent close to 2-3 weeks of part-time tinkering to build `plex-oidc` and get it functioning how I
wanted. Realistically, my users log in once a week at most. When you do the math -- 5-10 logins a
week, and 6-7 seconds "wasted" by using Authentik -- the amount of user time saved is very small.
That fact is inescapable.

Having operated this service for a few weeks now, the one thing I didn't originally consider, or at
least didn't have top of mind, was that Authentik burned some of _my_ time, in an operations sense.
Sometimes people would hit bugs with the authentication/authorization flow, or Authentik would crash
due to OOM and get into a weird state with Postgres locks or stalled Celery tasks or whatever.

In almost all of those cases, I just restarted the relevant Fly app, but it still required me to
disengage from whatever I was doing, mentally and physically. This was a system I pointed friends
and family at, that I just wanted to work. Even knowing that it's not the end of the world if I
didn't check on a problem for a few hours, it felt like a splinter in my mind.

All of this said, `plex-oidc` has not had a single hiccup since I deployed it. It just works. No OOM
issues, no weird issues seemingly related to shoehorning my intended authentication/authorization
flow into Authentik's model of flows, nothing. It just works, and keeps on working. That part made
it worth it, and continues to make it worth it every day.

## But where's the source code, bub?

Admittedly, I started working on this as a public repo, because _why not?_, but then I made it
private.  In fact, I bundled it into my infra monorepo of sorts, since this allowed me to iterate
faster by inlining secrets and tokens directly without having to nerdsnipe myself: oh, where should
I store these secrets? how will I deploy these secrets? There's also the aspect of the static
content being hand-rolled for my particular set-up, which means colors and background images and
hastily-designed logos specific to my set-up.

I'll likely open source this at some point in the near future once I have some time to clean up the
aforementioned things, and figure out how to generify the HTML/CSS and all of that.

[authentik]: https://goauthentik.io
[hardships]: /2022/11/25/self-hosting-chronicles-authentik-configuration.html
[authentik_2693]: https://github.com/goauthentik/authentik/issues/2693
[fly_io]: https://fly.io
[cloudflare-access-forwardauth]: https://github.com/tobz/cloudflare-access-forwardauth
[tracing]: https://docs.rs/tracing
[axum]: https://docs.rs/axum
[axum_sessions]: https://docs.rs/axum-sessions
[tower_http]: https://docs.rs/tower-http
[tower_governor]: https://docs.rs/tower_governor
[include_dir]: https://docs.rs/include_dir
[tokio]: https://docs.rs/tokio
[hyper]: https://docs.rs/hyper
[serde]: https://docs.rs/serde
[openidconnect]: https://docs.rs/openidconnect
[random_oidc_blog]: https://darutk.medium.com/diagrams-of-all-the-openid-connect-flows-6968e3990660
[auth0_oidc_docs]: https://auth0.com/docs/authenticate/login/oidc-conformant-authentication/oidc-adoption-auth-code-flow
[ibm_oidc_docs]: https://www.ibm.com/docs/en/was-liberty/base?topic=connect-configuring-openid-client-in-liberty
[moka]: https://docs.rs/moka
[quanta]: https://docs.rs/quanta
[reqwest]: https://docs.rs/reqwest
[tailwind_css]: https://tailwindcss.com/
[webpack]: https://webpack.js.org/
[vitejs]: https://vitejs.dev/