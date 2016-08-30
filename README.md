# The Project

This is a research project (a.k.a. a pile of hacks) with the goal of studying the Tumblr API to see how it can be bent to the will of alternative client interfaces, namely the [Interleaf](https://github.com/tsutsu/interleaf-client) dashboard-reader client. Tumblr's API is unusual compared to that of other feed services, and I got stuck when designing the architecture for Interleaf's Tumblr sync-backend, so I figured I would write an isolated engine that only talks to, and models, Tumblr.

The main feature of the project is a two-way syncing engine, that understands both the internal model Tumblr has of a user's account and posts, and also the internal database schema that the [Reeder](http://reederapp.com) RSS app uses for tracking its posts and feeds. The syncing engine can translate freely between these formats, along with its own internal representation of content items.

For anyone other than me, the only value you'll probably get from this code is in using it to understand Tumblr's resource schema (and maybe some Reeder internals), not so much in running it.

# The Binary

That said, provided in this project is a binary, **tumblrctl**.

## Setup

`tumblrctl` is a Ruby executable, which must be set up like so:

1. `bundle install` in the project directory

2. Copy the `config.yaml.example` to `config.yaml` and fill it out. Get OAuth credentials by [registering a Tumblr application](https://www.tumblr.com/oauth/apps).

3. If you are running tumblrctl from a VM or container, use [SSHFS](https://github.com/libfuse/sshfs) (or whatever host-guest mount you wish) to mount your host's (or any remote's) Reeder database directory to tumblrctl's `rkit-sshfs-mountpoint` directory. For example:

```bash
sshfs myhost:"/Users/you/Library/Containers/com.reederapp.rkit2.mac/Data/Library/Application Support/Reeder/rkit" /home/you/tumblrctl/rkit-sshfs-mountpoint -o 'sshfs_sync,cache=no'
```

## Low-level functionality

* `tumblrctl sync-owned-blog-pages` will download all the posts you've made in each of your Tumblr accounts, which haven't yet been downloaded, into tumblrctl's internal database. Posts are discovered by iterating through each of your blogs pagewise, as if scraping their webpages.
* `tumblrctl sync-owned-blog-reblog-sources` will, for each downloaded post in your accounts, download (if it hasn't already) the "source" post of that post—which is to say, the "root" post in the reblog-tree that your post is a member of. These posts are used by other operations for reblog-tree membership testing.
* `tumblrctl sync-unread-rss-items` will, for each *currently-unread* item in a given Reeder account (specified in the config file), attempt to map the RSS feed-item to a Tumblr post, and then add that Tumblr post *and its source* to tumblrctl's database.
* `tumblrctl sync-starred-rss-items` will do the same as the above, but for *currently-starred* items in the specified Reeder account.
* `tumblrctl clear-unread-rss-items` and `tumblrctl clear-starred-rss-items` will forcefully remove all records of {unread, starred} items from tumblrctl's internal database. tumblrctl currently does no garbage-collection of items, so you can use these to keep the database size from growing. (It stays surprisingly small, though.)
* `tumblrctl push-starred-items-to-drafts` takes each currently-starred item loaded into tumblrctl's internal database, posts it to a specified Tumblr account (the `:reblog_blog` setting in the config) *as a draft*, and then *unstars* the starred item both in its internal database and in Reeder. (Do not run this command with Reeder open! tumblrctl relies on adding items to Reeder's "offline edit history to be synced when next online" queue table; inserting into this table has undefined behavior when Reeder considers itself to be online.)
* `tumblrctl sync-drafts` downloads all the current draft posts from the config-specified Tumblr blog into the internal database.
* `tumblrctl clear-drafts` removes all draft posts from the internal database.
* `tumblrctl publish-random-draft` will take a random item from the drafts loaded into tumblrctl's internal database (i.e. the combination of drafts downloaded by `sync-drafts`, and drafts created by `push-starred-items-to-drafts`), and publish that item as a public post immediately to that same Tumblr blog. The draft item will be removed from the internal database and the published copy of the post will be downloaded in its stead, as if `sync-owned-blog-pages` was run.
* `tumblrctl purge-trash` will scan the Reeder database's "cached item" data for each currently-unread item in the config-specified Reeder account (you must have this feature enabled in Reeder to use this.) Each of these feed-items will be mapped to a Tumblr post, and then each {Reeder feed-item, Tumblr post} pair will be run through a set of user-definable filters. Any post that returns positive to any filter (is "trash") will be marked as read. (Don't use this command with the Reeder app running, either.)

## High-level functionality

### Syncing

`tumblrctl sync` is a macro-command that does the following in order: syncs account posts and reblog-sources, clears starred items, syncs unread and starred items, syncs drafts, pushes starred items to drafts, and then purges trash.

How I use this: I have an account in Reeder where every blog I follow on Tumblr has its RSS feed added as a subscription. This is the Reeder account I specify in tumblrctl's configuration. I read "my Tumblr dashboard" by reading the equivalent RSS items in Reeder, and by starring the ones that I want to be automatically reblogged. After I'm done using Reeder, I quit the Reeder app and run `tumblrctl sync`. tumblrctl's internal model now represents the up-to-date contents of my Tumblr dashboard, with all the Reeder {unread, starred} state attached; and Reeder's database now has its starred items unstarred, and its "trash" items marked as read. I reopen the Reeder app, temporarily, to commit these changes, then quit it again.

### Queuing Reblogs

When I run `tumblrctl sync` above, items I have starred in Reeder are automatically converted into draft posts to the Tumblr blog specified in tumblrctl's configuration.

I can then take a random one of these draft posts and post it, using `tumblrctl publish-random-draft`. This emulates Tumblr's own functionality of having a queue.

`publish_loop` is a script provided to just sit idle and call `tumblrctl publish-random-draft` every once in a while (with jitter). Using this script emulates the Tumblr queue's automatic posting behavior, but, in my opinion, in a better fashion: posts are mixed up rather than posted in queuing order, more than 250 posts can be posted from this queue per day, and posts are posted at "natural" intervals (a normal distribution of times between 5 and 15 minutes) rather than robotically on-the-dot every N minutes.

The `publish_loop` script won't use any memory while it's sitting idle, since the Ruby interpreter is unloaded between calls. You can replace this behavior with a cronjob or systemd unit or whatever else if you prefer.

*N.B.*: These items are created as draft posts because this is the only choice without negative consequences:

* using Tumblr's native queue would limit the number of items that could be queued.

* using an internal queue in tumblrctl would allow queued items to "rot": when a post is deleted, it can no longer be reblogged, unless you've already made your own queue-entry or draft-post out of it on the Tumblr server-side.

As such, I recommend not using the draft-posts of this blog for anything else. (The usefulness of drafts could be restored by giving all the "queue-alike drafts" a particular tag; but even with this, both Tumblr's website and API give you no way to browse drafts, only retrieve them in a linked-list-like fashion, so having thousands of automated queue-drafts makes it impossible to *find* your own manual drafts anyway.)

## Maintenance functions

* `tumblrctl purge-invalidated` repairs a Reeder database that has been corrupted by running `tumblrctl` with Reeder open. You can recognize a corrupted Reeder database by seeing posts in Reeder that have lost their permalinks, and thus don't "open" to anything other than a blank page. This function simply deletes these posts from Reeder's database; Reeder will download them again (correctly) the next time it opens.

## REPL

`tumblrctl` on its own starts a REPL (specifically [Pry](http://pryrepl.org)) with access to all of the above commands (each command is a function, where the name is the command name with the dashes replaced by underscores.)

You can use this REPL to fiddle around with individual feed-items (`Reeder::Item`) or Tumblr posts in the internal database (`Tumblr2::Post`). Look at the code to see what's available.

Good starting places for getting a handle on something in the object-graph to play around with:

* `Reeder::User.by_username("foo")`
* `Tumblr2::Blog.owned`
* `Tumblr2::Blog.by_alias("blogname")`

# 
