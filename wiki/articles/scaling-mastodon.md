# Scaling Mastodon
*Keeping up with the surge in traffic from people leaving Twitter*
## Are you a Mastodon admin? Check these out first
This article gets into my own personal fix for the huge influx in traffic that started during late November in 2022. Many other admins dealt with the same issue and I was only able to deal with mine with the information they graciously shared. Before you read my own fix, you should also check out these posts too:

* https://docs.joinmastodon.org/admin/scaling/
* https://blog.freeradical.zone/post/surviving-thriving-through-2022-11-05-meltdown/
* https://nora.codes/post/scaling-mastodon-in-the-face-of-an-exodus/

If you don't care about the intro or context, [skip down to the Solution section](#solution).

## Context
Back in November of 2022, it was announced that Elon Musk officially went through with his purchase of Twitter. The news cycle of that purchase kept getting worse and worse, with several decisions of his causing people to re-examine how they use Twitter, explore other social media, and give up on the platform entirely for some folks.

It was pretty disappointing to see. I and numerous furry friends of mine had kept up with each other and had a lot of fun with our various Twitter accounts, and it being all in one place was kinda nice. However, the new owner seems determined to run it completely into the ground even now, at time of writing.

What this meant for me and many other Mastodon admins is that we had a bunch of new people sign up all across the *fediverse*.

## About Mastodon & Merveilles.town
Mastodon is a decentralized social network based on the ActivityPub standard—to summarize, a Mastodon instance (or server) is like a smaller, roll-your-own Twitter that can to other instances like it. The collective, larger group of all instances talking to each other is called the "fediverse".

I have been the owner and admin of a Mastodon instance hosted at https://merveilles.town since 2018. Originally plain Mastodon, we're now running on a fork called Hometown, which gives us extra bonus features on top of what Mastodon offers.

For a long time I've been able to get by just on the default settings, with a few tweaks here and there, and I have the architecture and instance costs listed here if you're curious: https://github.com/merveilles/The-Town/blob/main/docs/INFRASTRUCTURE.md

Before this whole thing started, we were running on a mid-sized droplet on DigitalOcean and thanks to the donations of Merveilles.town members, we were at a pretty sustainable place—though I did have to restart Mastodon's system processes and update to a new version once in a while.

The important detail to keep in mind is that Merveilles.town is a relatively small, invite-only instance of ~400 active users (at time of writing). We only gained a handful of users during this time.

## The Problem
After waking up to the news about Elon and Twitter, I also found out that our instance was… struggling. There were constant 500 errors, people weren't able to do basic things like upload images, post, or see notifications from other users. While our memory usage seemed okay—my most common fix is to just restart processes when memory usage gets too high—we were still seeing a lot of strange bugs.

After poking around, I opened up Sidekiq, a tool that Mastodon uses to asynchronously process various jobs. For example, it pulls in posts from other instances, sends out notifications to mobile clients, populates each user's local timeline, and a whole lot more. It does this by splitting different types of jobs into different "queues", of which Mastodon uses 5.

I noticed that the number of "enqueued" (or pending) jobs in Sidekiq was the hugest I've seen for Merveilles.town: up to about ~5000 enqueued. I didn't learn until much later that larger instance owners were seeing the same thing but with way, way higher numbers. However, this explained all of the bugs that users on Merveilles.town were experiencing: if a job in one of the queues was waiting to be processed, other things couldn't happen, causing very long delays when interacting in basically every way on the instance.

The cause of this? There were so many people fleeing Twitter and using Mastodon now that their combined activity was too much for my little instance to handle. The instances that Merveilles.town is connected to were sending a lot of traffic our way, meaning that every time a new Mastodon user signed up and followed someone from Merveilles.town, that added another "job" to process. It was way too much for the default Mastodon configuration to handle.

<h2 id="solution">The Solution</h2>
The Sidekiq queues were stuck. How could I get them unstuck? The answer: parallelization. From the above articles I linked, other instance admins solved this issue by increasing the number of processes and threads that could get through those Sidekiq jobs. Spinning out services into their own separate servers to more evenly split the load also helped for them.

I'll leave out the exact details of the fixes I made since the above articles I linked at the top do a much better job of showing actual config files and such. Please use these solutions as a starting point and use the above guides for how to implement them yourself!

This was my initial attempt at fixing things:

* Sidekiq:
  * Increased threads & processes: 50 threads per process, and one process per each Sidekiq queue (`default`, `push`, `pull`, `mailers`, `scheduler`)
* Mastodon Web:
  * Increased processes and threads: 2 processes with 20 threads
* Mastodon Streaming:
  * Was already at 2 processes, increased threads to 15
* Postgres:
  * Increased `max_connections` to 200 and increased alotted memory to 4gb in the `postgresql.conf` file using `pgtune`
  * Switched to use transaction pooling and installed `pgbouncer`
* Elasticsearch:
  * JVM heap size kept at 512mb
* DigitalOcean Droplet Size:
  * Increased the droplet size from one with 8gb of RAM to 16gb of RAM

We were starting to see an improvement here, but it was still difficult to get through all the enqueued jobs when only one process could handle one queue, and we were still seeing errors. I found you could configure Sidekiq processes to handle queues in a specific order, which meant we could distribute the workload across the processes instead of confining one large queue to just one process. So, next:

* Increased Sidekiq processes from 5 to 6, and each process handled queues in a different order. Each process had 50 threads.
  * default, push, pull
  * default, pull, push
  * push, default, pull
  * pull, default, push
  * mailers
  * scheduler

This finally got us all through the queues and we were at realtime. There were no more delays, folks could upload images, and everything was great! However, those scaling changes I made were extremely memory-hungry, maxing out our doubled RAM in a matter of days. After some playing around, I settled on these changes:

* Sidekiq (weighted priority, not a specific order):
  * [15 threads] scheduler(5), default(4), push(3), pull(2), mailers
  * [15 threads] default(4), push(3), pull(2), mailers
* Mastodon Web:
  * 2 processes, 4 threads
* Mastodon Streaming:
  * 2 processes, 4 threads
* Postgres:
  * Reduced max memory to 2gb
* Elasticsearch
  * Reduced JVM heap size to 256mb

This has worked well over the past week or so since I've changed it, and our memory usage is under control.

## Where to go from here
Now, my main goal is to reduce the overall memory usage as much as possible to scale us back down to the 8gb DigitalOcean droplet. Merveilles.town is kept afloat by donations from our members, and I want to make absolutely sure we can keep the instance around without having to dip into personal funds.

This was definitely an adventure to fix—and it always feels weird getting neck-deep in server configuration when I'm a front-end web software engineer with an art degree, but I'm super glad when it works out.

A huge shout-out to the Mastodon maintaners and the folks at freeradical.zone and weirder.earth, your articles that I linked above were crucial in making sure I could get my instance fixed up.

If you have any questions or suggestions for this post, feel free to reach out! Find me on Merveilles here: https://merveilles.town/@somnius

## Changelog

* 2022-12-20
  * Initial post date
