# Self-hosting chronicles: Authentik configuration

In my [inaugural
post](https://notes.catdad.science/2022/10/27/self-hosting-hard-way-exposing-services.html), I
briefly covered the shape of my new approach to exposing self-hosted applications to the public
internet in a reasonably secure way. Most of this set up depends on a piece of software called
[Authentik](https://goauthentik.io/), an open-source identity provider which acts as the glue
between Cloudflare Access and the actual authentication mechanisms I want to depend on.

Getting Authentik set up and configured the way I wanted it to be configured was by far the hardest
part of the entire process, so it felt worth writing up on the off-chance that someone running into
the same problems as I did can potentially find this post, presented as an encapsulated list of
problem/answer tuples, and get themselves sorted that much faster.

## Deployment

Authentik, as an identity provider, has to run somewhere. When you think of Google or Github or Okta
as identity providers, what's the one thing they all share in common? Their IdP endpoints are
exposed to the public internet, because you need to be able to communicate with them from your
browser or mobile device. Authentik is no different in this regard, and so I needed to run it
somewhere where I could expose it to the public internet. As laid out in the aforementioned post, I
chose [Fly.io](https://fly.io/) for this purpose.

Fly.io is a ... something between an infrastructure-as-a-service provider and platform-as-a-service
provider, depending on how hard you squint. They have the trappings of a PaaS provider, where their
`flyctl` CLI tool, and a simple `fly.toml` configuration file, is essentially all you need to start
creating and deploying applications.

Despite this, I still hit some roadblocks that were a culmination of how Authentik expects things
to be done, and how Fly.io works.

### Writing to standard error, or not

The first problem I encountered was that `/dev/stderr` does not seem to be available on Fly.io.

The ["lifecycle" script](https://github.com/goauthentik/authentik/blob/main/lifecycle/ak) used by
Authentik has a simple logging function which pipes its output to `/dev/stderr`. I forget the exact
error message that I got... maybe "file doesn't exist" or maybe "permission denied", but I got an
error, consistently.

My original searching lead me to
[this forum post on community.fly.io](https://community.fly.io/t/redirect-logs-to-stdout/514),
which made me originally think that this was potentially a bug with both stdout and stderr, and
perhaps it hadn't yet been fixed for stderr? I had also stumbled onto this
[Unix StackExchange question](https://unix.stackexchange.com/questions/38538/bash-dev-stderr-permission-denied)
where one of the answers suggests just using the `1>&2` style of redirection instead of the stderr
`/dev` entry itself.

Admittedly, I never investigated this more holistically and instead took the simpler approach of
just not having output redirected to `/dev/stderr`:

```dockerfile
FROM ghcr.io/goauthentik/server:2022.9.0
USER root
RUN sed -i -e 's# > /dev/stderr##g' /lifecycle/ak # <-- This lil' guy right here.
USER authentik
ENTRYPOINT ["/lifecycle/ak"]
CMD ["server"]
```

### Limitations with Upstash Redis

The next issue I encountered, once Authentik could actually start, was that the managed Redis
service ([Upstash Redis](https://docs.upstash.com/redis)) doesn't support multiple Redis databases.

Authentik uses Redis for two purposes: caching within Django and Celery. Celery is a Python library
for distributed task processing, where you can enqueue tasks to be run by a pool of Python worker
processes, and those workers pick up the tasks and run them and report the status and so on. Celery
has a concept of "backends" which are the systems that actually get used for the transport of task
messages, of which Redis is a supported backend.

Authentik takes advantage of already wanting Redis for Django caching and configures Celery to use
it, too. The problem is that Authentik points itself, and Celery, at different Redis "databases" in
an attempt to isolate the data between the two use cases. Redis databases are a logical construct
for isolating the keyspace, so that keys don't overlap and clobber each other, and so on.

Upstash Redis, which exists by itself but is offered as a managed service on Fly.io, doesn't support
multiple Redis databases, instead only supporting one database, the default database, at index 0.
Luckily, Authentik already exposed the ability to change the database selection as part of its
configuration, which was simply achieved by setting the follow environment variables:

```
AUTHENTIK_REDIS__CACHE_DB = "0"
AUTHENTIK_REDIS__MESSAGE_QUEUE_DB = "0"
AUTHENTIK_REDIS__WS_DB = "0"
```

This shoves all usages of Redis -- both Authentik itself and Celery -- onto the same Redis database.
So far, I've yet to experience any issues with doing so, and even in a brief conversation with the
creator of Authentik, they weren't necessarily sure if Authentik needed to actually isolate the data
like this, or at least isolate it any longer.

Admittedly, I ended up spinning up my own Redis container/application on the side because using
Upstash Redis kept leading to weird Redis issues from Celery's perspective. There was originally a
bug with how Upstash Redis implemented `MULTI`/`EXEC` for certain commands that was definitively
wrong (compared to the official Redis behavior) which Upstash fixed a few weeks after I reproduced
and reported the behavior... but even after it was fixed, somehow, their service still acted weird
when used by Authentik/Celery.

I didn't have the time or energy to keep debugging it, so I spun up my own Redis
container/application. Maybe there were more bugs with Upstash Redis that are now fixed, or maybe I
was doing something wrong with my particular configuration... who knows.

## Configuration

At this point, I at least had Authentik deployed and running. I won't go into all aspects of
configuring it prior to the first initial successful deployment, because the rest of it is fairly
mundane and covered by their documentation.

The real bugaboo, however, was actually configuring Authentik in terms of its behavior.

### How Authentik works

Authentik has a very particular viewpoint/approach in terms of how it operates. This is not to say
that the approach Authentik takes is bad, just that deviating from it can often leave you
frustrated.

To start, Authentik has an out-of-the-box experience that configures a number of stages, policies,
and flows to give you a setup that is close to how most people might be expected to use it. You can
and _should_ read the documentation, but Authentik uses the concept of _flows_ and _stages_ as a way
to describe the authentication/authorization journey a user takes. Configuring these flows/stages is
how you configure how Authentik works: how users authenticate (username/password? federated login?),
whether or not they're authorized to access a particular applicatiom, enforcement of password
strength or two-factor measures, and so on.

To achieve this, Authentik supports a feature called "blueprints." These are YAML files that can be
processed to programmatically create flows, stages, policies, bindings, and most all of the various
model objects that are used to configure Authentik's behavior. These YAML files are essentially data
mocks on steroids: you define the model objects themselves, and are given access to helper logic in
the form of YAML "tags"... such as setting the value of a primary key field to `!KeyOf ...` to have
the blueprint importer search for another blueprint-managed object by a logical identifier and
insert the actual primary key for you.

### Using blueprints

As Authentik is an identity provider, many users do in fact use it as the source of
authentication/authorization itself... as in, users are registered in Authentik as the source of
truth, and everything flows outwards from Authentik. The out-of-the-box experience works well for
this, and in many instances, I've seen users be told to simply tweak the out-of-the-box flows/stages
to achieve their desired outcome.

As part of my deployment, though, Authentik was simply glue logic between an existing authn/authz
mechanism -- Plex's own identity provider -- and Cloudflare Access. I didn't want local users
created, and I didn't care about specific applications, and I certainly didn't want Authentik to
proxy any access or anything like that. I just wanted an identity provider.

Most importantly, I wanted to use blueprints, because they seemed to be the only way (a good way, to
be fair!) to idempotently configure Authentik.

### Blueprint pitfalls

As I started to look into configuring Authentik entirely via blueprints, I hit numerous pitfalls and
struggled often with finding a consistent source of documentation and answers to mystifying
behavior. I'll list these out in no particular order.

#### Blueprint file structure is documented with varying levels of specificity

Looking at the existing blueprint files can be a useful hands-on example of how to write your own,
but this leaves a lot to be desired. As blueprints are essentially model object mappings, you're sort
of writing your very own `INSERT INTO table (field, field, ...) VALUES (...)` but in YAML. This
means you actually need to go and look at the model definition for objects you want to add, or look
at the source code.

There's no documentation for the models, either by themselves or in the context of blueprint
development. This means you have to become intimately familiar with the models if you're trying to
create something via blueprints that doesn't already have representation in the out-of-the-box
blueprints.

Beyond that, there's a lot of uncertainty around parts of the blueprint definition such, as, for
example, specifiying identifiers. Identifiers are used to provide a unique identifier (duh) for a
given model object.. which is fine, and makes sense. No issue there. Again, however, there's little
to no documentation on the models, so if you don't specify the right identifier fields, the
blueprint importer just yells at you.

The experience is poor, to say the least.

#### Blueprints have no ordering constraints

Authentik supports what they call a "meta model": since blueprints are tied almost exclusively to
actual models, they have a concept of "meta models" which allow operations other than creating a
model object. The only meta model currently is "apply blueprint."

If you're like me, you might read that and think "oh nice, dependency management!". Or maybe
something along those lines. It's certainly what I thought, given that meta models are described as:

> This meta model can be used to apply another blueprint instance within a blueprint instance. This
> allows for dependency management and ensuring related objects are created.

Alas: _nope_.

The meta blueprint apply operation certainly can import and apply other blueprints, but it _cannot_
use them in any sort of way that would actually qualify as dependency management. There's no
ordering (apply blueprint first, and then create the actual model objects described further down in
the blueprint, etc) and there's no logical inclusion, which means you can't even reference model
objects created in meta-applied blueprints (i.e. using the special YAML "tags") because it doesn't
actually load the blueprint in an inclusive way, or dependably apply it first.

That's not to say any of this is easy to program up, but, I mean... the documentation _says_ the
words "dependency management", and there is no hint whatsoever of anything that could reasonably be
considered dependency management. All it lets you do is prevent one blueprint from running in the
meta-applied blueprints have been run successfully... so you get to wait multiple hours for
blueprint runs to finally make enough progress to cover all of the blueprint dependencies. Not great
at all.

I currently use a single blueprint file to reliably construct all of the necessary model objects and
be able to correctly reference them in the creation of subsequent model objects.

### Actual flow behavior and figuring out what matters

As I was trying to write blueprints for repeatable, idempotent configuration, I also still had to
figure out what I even needed to configure to achieve my desired setup.

Remember that Authentik uses a concept of flows and stages, where flows are tied to specific phases
of the journey of a user through an identity provider, such as identification, authorization,
enrollment, and so on. Stages are merely the next step down, and allow configuring the actual steps
taken within each of those flows.

This is where things mostly got/felt wonky because, again, the documentation has oscillating
specificity. If you go to the documentation section on "Flows", the landing page has some decent
high-level information on flows, and the various available flows, but it has more content by volume
on the various ways you can configure a flow's web UI (flows are also weirdly intertwined with the
context they're used in i.e. web via "headless" (LDAP)) so you really have to spelunk here.

#### A lot of boilerplate to satisfy various flow constraints

As an example of where this all gets wonky and cargo cult-y, let's consider the authentication flow.

Authentik itself is always the main entrypoint, even if like me, you end up totally depending on a
federated social login setup a la Google/Plex/etc. This means we need an authentication flow, which
is all fine and good. We can configure an identification stage in that flow that specifies how a
user should identify themselves. There's already a fairly intuitive bit of verbiage on the
configuration modal for an identification stage around showing buttons for specific identification
sources, such as these federated social login options. Think of it like the typical "Sign In with
Google" buttons you may have seen before.

"Great", you think. You dutifully select the federated social login source you want to use, and
unselect the username/email options because we don't want to login as local users. You try to use it
and... weird, doesn't work. As it turns out, you actually need to specify an authentication flow for
the federal social login source itself. Additionally, you also need an enrollment flow for your
federated source login source.

This is ultimately because you need to log in to Authentik itself (remember the part about Authentik
being the "entrypoint"), which means need to creating a local user in Authentik even if you're
farming out that aspect to something like Google... which necessitates the separate authentication
and enrollment flows.

This isn't called out anywhere in the documentation that I have been able to find. Pure trial and
error here.

#### Making everything a flow leads to a suboptimal user experience

Despite the above, I eventually managed to get my intended flow to working. Handing it over to a
user to test out though yielded the following: "my browser refreshed a lot of times and seemed like
it didn't actually work, and then I eventually landed on the application".

What the user was describing was an idiosyncracy of everything -- including sources -- needing a
flow, and how flows are executed.

Since our federated social login source needed source-specific flows -- two of them -- this actually
meant that the web-based flow redirects them three to four times in total:

- they land at the Authentik "identify yourself" page where they have to manually click on the
  button for authenticating via Plex
- a modal pops up for Plex login, etc, and closes once they do so
- the page then redirects to source authentication page which logs them in
- if they're brand new, they're also redirected to the source enrollment flow first
- then they're redirected to the source authorization flow (which is a no-op, in my case) which
  sits around for 2-3 seconds inefficiently calculating JWTs or something
- finally, they're redirected back to the SP initiator (the entity that triggered the IdP flow in the
  first place) and can actually use the thing

Some of this could be solved by allowing for short circuiting behavior around default options, such
as not requiring the user to have pick an identification source if there's only one available... or
not redirecting to a specific flow if it's literally a no-op. There's an
[open PR](https://github.com/goauthentik/authentik/pull/3696) for the former, albeit stalled on
getting pushed through.

These are paper cuts that make using Authentik clunky from the user perspective. I'm not happy
having to spend so much time actually configuring flows to do what I want, but at least if I can
turn that into a consistent and intuitive experience for users, then the effort was worth it. When a
user is getting redirected multiple times, or hitting pages that are so slow to execute that they
wonder if they're stuck... then the experience is not consistent or intuitive.

## Conclusion

I intended to write this post in a very problem/answer-oriented way, and I managed that in some
regard, but clearly I diverged a bit because recounting the sheer effort I expended to get things
configured made it hard to not vent a bit.

Hopefully for anyone reading this, it gives you some additional insight to get Authentik configured
with less effort than I personally spent.

As for me, I do intend to eventually contribute back, or try to contribute back, improvements to the
sharp edges that chewed up so much of my time. I do think there's something to be said, though,
about focusing on the user experience aspect where possible, especially when it comes to security.
My users can't get frustrated and actually, like... do anything less secure as a workaround to avoid
redirect hell: they use this thing I've configured and provided, or they don't get access at all.
For other people deploying Authentik, though? Who knows what their use case is and if these
shortcomings might undermine their ability to provide an alternative authn/authz solution to their
users that doesn't feel more clunky to use than whatever they came from.
