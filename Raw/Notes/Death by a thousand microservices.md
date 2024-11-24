---
title: Death by a thousand microservices
source: https://renegadeotter.com/2023/09/10/death-by-a-thousand-microservices.html
clipped: 2023-09-13
published: 
category: development
tags:
  - development
read: true
---

### The Church of Complexity

There is a pretty well-known sketch in which an engineer is explaining to the project manager how an overly complicated maze of microservices works in order to get a user’s birthday - and fails to do so anyway. The scene accurately describes the absurdity of the state of the current tech culture. We laugh, and yet bringing this up in a serious conversation is tantamount to professional heresy, rendering you borderline un-hirable.

How did we get here? How did our aim become not addressing the task at hand but instead setting a pile of cash on fire by **solving problems we don’t have**?

### The perfect storm

There are a few things in recent history that may have contributed to the current state of things. First, a whole army of developers writing JavaScript for the browser started self-identifying as “full-stack”, diving into server development and asynchronous code. JavaScript is JavaScript, right? What difference does it make what you create using it - user interfaces, servers, games, or embedded systems. *Right*? Node was still kind of a [learning project of one person](https://www.youtube.com/watch?v=M3BM9TB-8yA), and the early JavaScript was a deeply problematic choice for server development. [Pointing this out](https://notes.ericjiang.com/posts/751) to still green server-side developers usually resulted in a lot of huffing and puffing. This is all they knew, after all. The world outside of Node effectively did not exist, the Node way was the only way, and so this was the genesis of the stubborn, dogmatic thinking that we are dealing with to this day.

![[Raw/Media/Resources/3d5b90770703924856a2ee2914b5c527_MD5.webp]]

**And then**, a steady stream of [FAANG](https://en.wikipedia.org/wiki/Big_Tech) veterans started merging into the river of startups, mentoring the newly-minted and highly impressionable young JavaScript server-side engineers. The apostles of the Church of Complexity would assertively claim that “how they did things over at Google” was unquestionable and correct - even if it made no sense with the given context and size. What do you *mean* you don’t have a separate *User Preferences Service*? That just *will not scale, bro!*

But, it’s easy to blame the veterans and the newcomers for all of this. What else was happening? Oh yeah - easy money.

What do you do when you are flush with venture capital? You don’t [go for revenue](https://www.youtube.com/embed/BzAdXyPYKQo?si=3lOVi1rhtaPC-nv4), surely! On more than one occasion I received an email from management, asking everyone to be in the office, tidy up their desks and look busy, as a clouder of Patagonia vests was about to be paraded through the office. Investors needed to see explosive growth, but not in profitability, no. They just needed to see how quickly the company could hire ultra-expensive software engineers to do … *something*.

And now that you have these developers, what do you do with them? Well, they could build a simpler system that is easier to grow and maintain, or they could conjure up a monstrous constellation of “microservices” that no one really understands. Microservices - the new way of writing scalable software! Are we just going to pretend that the concept of “distributed systems” never existed? (Let’s skip the whole parsing of nuances about microservices not being real distributed systems).

Back in the days when the tech industry was not such a bloated farce, distributed systems were respected, feared, and generally avoided - reserved only as the weapon of last resort for particularly gnarly problems. Everything with a distributed system becomes more challenging and time-consuming - development, debugging, deployment, testing, resilience. But I don’t know - maybe it’s all super easy now because *toooollling*.

There is no standard tooling for microservices-based development - there is no common framework. Working on distributed systems has gotten only marginally easier in 2020s. The Dockers and the Kuberneteses of the world did not magically take away the inherent complexity of a distributed setup.

I love referring to this [summary of 5 years of startup audits](https://kenkantzer.com/learnings-from-5-years-of-tech-startup-code-audits/), as it is packed with common-sense conclusions based on hard evidence (and paid insights):

> … the startups we audited that are now doing the best usually had an almost brazenly ‘Keep It Simple’ approach to engineering. Cleverness for cleverness sake was abhorred. On the flip side, the companies where we were like ”woah, these folks are smart as hell” for the most part kind of faded.

Literally - “complexity kills”.

The [audit](https://podcasts.apple.com/mt/podcast/lessons-from-5-years-of-startup-code-audits/id341623264?i=1000567623452) revealed an interesting pattern, where many startups experienced a sort of collective imposter syndrome while building straight-forward, simple, performant systems. There is a dogma attached to not starting out with microservices on day one - no matter the problem. “Everyone is doing microservices, yet we have a single Django monolith maintained by just a few engineers, and a MySQL instance - what are we doing wrong?”. The answer is almost always “nothing”.

Likewise, it’s common for seasoned engineers to experience hesitation and inadequacy in today’s tech world, and the good news is that, no - it’s probably not you. It’s common for teams to pretend like they are doing “webs cale”, hiding behind libraries, ORMs, and cache - confident in their expertise (they crushed that Leetcode!), yet they may not even be [aware of database indexing basics](https://www.reddit.com/r/programming/comments/f46f5a/comment/fhp26k8/?context=3). You are operating in a sea of unjustified overconfidence, waste, and Dunning-Kruger, so who is really the imposter here?

### There is nothing wrong with a monolith

The idea that you cannot grow without a system that looks like the infamous diagram of Afghanistan war strategy is largely a myth.

![[Raw/Media/Resources/426a1e0347046474de2d2db8a875f00e_MD5.png]]

Dropbox, Twitter, Facebook, Instagram, Shopify, Stack Overflow - these companies and others started out as monolithic code bases. Many have a monolith at their core to this day. Stack Overflow makes it a [point of pride](https://stackexchange.com/performance) how little hardware they need to run the massive site. Shopify is still a [Rails monolith](https://blog.quastor.org/p/shopify-ensures-consistent-reads), leveraging the tried and true [Resque](https://twitter.com/ShopifyEng/status/1597983928018948096) to proces billions of tasks.

WhatsApp went supernova with their [Erlang monolith and 50 engineers](https://blog.quastor.org/p/whatsapp-scaled-1-billion-users-50-engineers). How?

> WhatsApp consciously keeps the engineering staff small to only about 50 engineers.
> 
> Individual engineering teams are also small, consisting of 1 - 3 engineers and teams are each given a great deal of autonomy.
> 
> In terms of servers, WhatsApp prefers to use a smaller number of servers and vertically scale each server to the highest extent possible.

Instagram was acquired for billions - with a crew of 12.

And do you imagine Threads as an effort involving a whole Meta campus? Nope. They followed the [Instagram model](https://instagram-engineering.com/static-analysis-at-scale-an-instagram-story-8f498ab71a0c), and this is the entire Threads team:

![[Raw/Media/Resources/89b35467ab2901d1d2e6f3be50fedd1c_MD5.webp]]

Credit: Substack - The Pragmatic Engineer

Perhaps claiming that *your* particular problem domain requires a massively complicated distributed system and an open office stuffed to the gills with turbo-geniuses is just crossing over into arrogance rather than brilliance?

Try [Friendly Fire](https://friendlyfire.tech/) - our system for GitHub and Slack to streamline the code review process.

### Don’t solve problems you don’t have

It’s a simple question - what problem are you solving? Is it scale? How do you know how to break it all up for scale and performance? Do you have enough data to show what needs to be a separate service and why? Distributed systems are built for size and resilience. Can your system scale and be resilient at the same time? What happens if one of the services goes down or comes to a crawl? Just scale it up? What about the *other* services that are going to get hit with traffic? Did you war-game the endless permutations of things that can and will go wrong? Is there back pressure? Circuit breakers? Queues? Jitter? Sensible timeouts on every endpoint? Are there fool-proof guards to make sure a simple change does not bring everything down? The knobs you need to be aware of and tune are endless, and they are all specific to your system’s particular signature of usage and load.

The truth is that most companies will never reach the massive size that will actually require building a true distribute system. Your cos playing Amazon and Google - without their scale, expertise, and endless resources - is very likely just an egregious waste of money and time. Religiously following all the steps from an article called “Ten morning habits of very successful people” is not going to make you a billionaire.

*The only thing harder than a distributed system is a BAD distributed system*.

![[Raw/Media/Resources/872d47f98a10c0a11829da2839642062_MD5.png]]

### “But each team…but separate… but API”

Trying to shove a distributed topology into your company’s structure is a noble effort, but it almost always backfires. It’s a common approach to break up a problem into smaller pieces and then solve those one by one. So, the thinking goes, if you break up one service into multiple ones, everything becomes easier.

The theory is sweet and elegant - each microservice is being maintained rigorously by a dedicated team, walled off behind a beautiful, backward-compatible, versioned API. In fact, this is all so steely that you rarely even have to communicate with that team - as if the microservice was maintained by a 3rd party vendor. It’s *simple*!

If that doesn’t sound familiar, that’s because this rarely happens. In reality, our Slack channels are *flooded* with messages from teams communicating about releases, bugs, configuration updates, breaking changes, and PSAs. Everyone needs to be on top of everything, all the time. And if that wasn’t great, it’s normal for one already-slammed team to half-ass multiple microservices instead of doing a great job on a single one, often changing ownership as people come and go.

In order to win the race, we don’t build *one* good race car - we build a fleet of shitty golf carts.

![[Raw/Media/Resources/3553b1a14fb01be66a07cf6c7542834a_MD5.png]]

### What you lose

There are multiple pitfalls to building with microservices, and often that minefield is either not fully appreciated or simply ignored. Teams spend months writing highly customized tooling and learning lessons not related at all to the core product. Here are just some often overlooked aspects…

#### Say goodbye to DRY

After decades of teaching developers to write Don’t Repeat Yourself code, it seems we just stopped talking about it altogether. Microservices by default are not DRY, with every service stuffed with redundant boilerplate. Very often the overhead of such “plumbing” is so heavy, and the size of the microservices is so small, that the average instance of a service has more “service” than “product”. So what about the common code that *can* be factored out?

-   Have a common library?
-   How does the common library get updated? Keep different versions everywhere?
-   Force updates regularly, creating dozens of pull requests across all repositories?
-   Keep it all in a monorepo? That comes with its *own* set of problems.
-   Allow for some code duplication?
-   Forget it, each team gets to reinvent the wheel every time.

Each company going this route faces these choices, and there are no good “ergonomic” options - you *have* to choose your version of the pain.

#### Developer ergonomics will crater

“Developer ergonomics” is the friction, the amount of effort a developer must go through in order to get something done, be it working on a new feature or resolving a bug.

With microservices, an engineer has to have a mental map of the entire system in order to know what services to bring up for any particular task, what teams to talk to, whom to talk to, and what about. The “you have to know everything before doing anything” principle. How do you keep on top of it? Spotify, a multi-billion dollar company, spent probably not negligible internal resources to build [Backstage](https://backstage.spotify.com/), software for cataloging its endless systems and services.

This should at least give you a clue that this game is not for everyone, and the price of the ride is *high*. So what about the *tooooling*? The Not Spotifies of the world are left with MacGyvering their own solutions, robustness and portability of which you can probably guess.

And how many teams actually streamline the process of starting a *YASS* - “yet another stupid service”? This includes:

-   Developer privileges in GitHub/GitLab
-   Default environment variables and configuration
-   CI/CD
-   Code quality checkers
-   Code review settings
-   Branch rules and protections
-   Monitoring and observability
-   Test harness
-   Infrastructure-as-code

And of course, multiply this list by the number of programming languages used throughout the company. Maybe you have a usable template or a runbook? Maybe a frictionless, one-click system to launch a new service from scratch? It takes months to iron out all the kinks with this kind of automation. So, you can either work on your product, or you can be working on *toooooling*.

#### Integration tests - LOL

As if the everyday microservices grind was not enough, you also forfeit the peace of mind offered by solid integration tests. Your single-service and unit tests are passing, but are your critical paths still intact after each commit? Who is in charge of the overall integration test suite, in Postman or wherever else? Is there one?

![[Raw/Media/Resources/75bf7513293ea2e7b1c000bf6d016968_MD5.gif]]

Integration testing a distributed setup is a nearly-impossible problem, so we pretty much gave up on that and replaced it with another one - Observability. Just like “microservices” are the new “distributed systems”, “observability” is the new “debugging in production”. Surely, you are not writing real software if you are not doing…. observability!

Observability has become its own sector, and you will pay in both pretty penny and in developer time for it. It doesn’t come as plug-and-pay either - you need to understand and implement canary releases, feature flags, etc. Who is doing that? One already [overwhelmed](https://renegadeotter.com/2023/07/26/i-am-not-your-cloud-person.html) engineer?

As you can see, breaking up your problem does not make solving it easier - all you get is another set of *even harder problems*.

### What about just “services”?

Why do your services need to be “micro”? What’s wrong with [just services](https://leeatchison.com/app-architectures/moving-beyond-microservices-hype/)? Some startups have gone as far as create *a service for each function*, and yes, “isn’t that just like Lambda” is a valid question. This gives you an idea of how far gone this unchecked cargo cult is.

So what do we do? [Starting with a monolith](https://www.fearofoblivion.com/build-a-modular-monolith-first) is one obvious choice. A pattern that could also work in many instances is “trunk & branches”, where the main “meat and potatoes” monolith is helped by “branch” services. A branch service can be one that takes care of a clearly-identifiable and separately-scalable load. A CPU-hungry *Image-Resizing Service* makes way more sense than a *User Registration Service*. Or do you get so many registrations per second that it requires independent horizontal scaling?

![[Raw/Media/Resources/207d883c700a7e185c1b3752f2a203c4_MD5.gif]]

![[Raw/Media/Resources/727a36f9e5be47a8e6a39182aaa3b552_MD5.png]]

### The pendulum is swinging back

The hype, however, seems to be dying down. The VC cash faucet is tightening, and so the businesses have been market-corrected into exercising common-sense decisions, recognizing that perhaps splurging on web-scale architectures when they don’t have web-scale problems is not sustainable.

![[Raw/Media/Resources/a346708babcd61ace02057cdbc13ce25_MD5.png]]

![[Raw/Media/Resources/64156c16a8ed9b6af6973b217840e36e_MD5.png]]

![[Raw/Media/Resources/368ea71535cd4f668cef777dc174868f_MD5.png]]

Ultimately, when faced with the need to travel from New York to Philadelphia, you have two options. You can either attempt to construct a highly intricate spaceship for an orbital descent to your destination, or you can simply purchase an Amtrak train ticket for a 90-minute ride. *That* is the problem at hand.

### Additional reading

[How to recover from microservices](https://world.hey.com/dhh/how-to-recover-from-microservices-ce3803cc)

[You want modules, not microservices](https://blogs.newardassociates.com/blog/2023/you-want-modules-not-microservices.html)

[Build a modular monolith first](https://www.fearofoblivion.com/build-a-modular-monolith-first)