---
layout: post
title: "Breaking prod chapter 2: disappearing friends"
date: 2025-10-18 12:25:00 +0900
categories: engineering
lang: en
---

This is another incident that is memorable because of the unwavering hate for PHP it aroused in me.

## Disappearing friend lists on Zynga Poker

Zynga Poker is one of Zynga's oldest games. When I worked on it years ago, it had a frankenstein backend consisting of sedimentary layers of PHP, strapped with duct tape to a Java-based TCP socket server. These stood atop a home-grown user storage layer, lovingly called "Sexy," that used Memcached as a front cache backed by MySQL. Custom clients for Sexy were implemented in both PHP and Java, so that each part of the stack could interact directly with it.

Among many other pieces of data, we stored user friend lists in Sexy. We served the data from our Java server, but for some reason there was an extra bit of indirection that went through PHP. So serving the friend list took a path like this:

```
Flash game client -> Java -> PHP -> Sexy -> PHP -> Java -> Flash game client
```

I don't remember exactly why, but I decided to take out the PHP part of that path so that it would look like this:

```
Flash game client -> Java -> Sexy -> Java -> Flash game client
```

It seemed like a straightforward change, I just needed to port some PHP into Java, what could possibly go wrong?

Incidentally, the friend list was stored in a data structure like this:

```
{
  // social network -> user ID list
  "1": ["user id 1", "user id 2"], // "facebook" friends
  "29": ["user id 3", "user id 4"] // "some other social network" friends
}
```

But sometimes the data looked like this:

```
{
  "0": [], // probably some pre-historic data migration bug
  "1": ["user id 1", "user id 2"], // "facebook" friends
  "29": ["user id 3", "user id 4"] // "some other social network" friends
}
```

Which is fine - it's not user-facing - but what if a user only has Facebook friends and no friends from social network 29?

PHP's `json_encode` happily gives you this:

```
[[], ["user id 1", "user id 2"]]
```

Thanks to a peculiarity of PHP arrays, our serialized data can now either be an array or an object! Which is fine and completely invisible, _if you only read and write the data in PHP_.

After porting the code to Java, everything looked perfectly fine. Until the code reached production of course, where suddenly some users suddenly lost their friends lists!

## Aftermath

It was easy enough to stop the bleeding - if memory serves me right I used some feature of the very excellent and flexible Jackson JSON API to maintain compatibility with the PHP-JSON-serialized arrays.

But I was not able to do much more! My very capable tech lead had to jump in and work with our analytics team to reconstruct friend lists from analytics.

At the time, I did not have the communication skills, nor the resourcesfulness to step outside of our day-to-day process and proactively work with the right people to manage this unique production incident to complete resolution. However, having broken prod quite a few times since then, I'd like to think that I've improved in this area.

## If it ain't broke, don't fix it?

It may be tempting to use this incident as proof of the mantra "if it ain't broke, don't fix it," but I think that this is a simplistic and dangerous statement. It instills _fear_ when instead (in my humble opinion) _skillful and calculated risk-taking_ ought to be encouraged, both for the happiness of the developer and the maintainability of the software project.
