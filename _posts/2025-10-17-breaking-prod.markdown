---
layout: post
title: "Breaking prod chapter 1: chat via polling"
date: 2025-10-18 12:25:00 +0900
categories: engineering
lang: en
---

I've been working on live service games for a while and have broken production plenty of times, both as an inexperienced intern and even as a slightly less-experienced middle-aged man. I thought it would be interesting to write about some of the memorable and interesting ones before they fade from my sleep-deprived brain.

## Breaking prod at gloops

This incident is memorable because it was the very first one I caused, in my very first engineering role as an intern at a Japanese games company called gloops. In fact, it happened not once, but twice! And knocked down the game during its busiest time of day when the company made the most money, resulting in a loss of tens of thousands of dollars.

The game was basically an server-side rendered web application with some of the flashier UI built in Flash. It featured a bulletin board where team members could communicate with each other to coordinate during battle events.

As a young and energetic intern, I thought it was _lame_ that the users had to refresh the entire page to see new posts on the bulletin board. Wouldn't it be _cool_ if the bulletin board updated in realtime? AJAX was all the rage in 2012, _why not_ just poll the server for new messages? Of course, I never seriously considered the question of _why not_ and sprung to action.

The result? When the battle event started, servers were almost instantly overloaded and nobody was able to play the game! And I did this, not only once, but twice, thinking for sure that Redis would save the feature. Needless to say, we did not try a third time.

## Looking back

It was a combination of a hungry but in-experienced intern (me), and a team that was okay with load-testing on prod that allowed this to happen. Was there a possible implementation that would not have broken the game? Maybe, but I did not have the skill or experience to find out.

This article is not intended to say anything about technical design choices. I have successfully used polling for receiving asynchronous, realtime notifications in other projects, and I would argue that it's much easier to maintain eventually consistent state with polling because you are forced to think about that problem, rather than throwing everything at a WebSocket and hoping for the best.
