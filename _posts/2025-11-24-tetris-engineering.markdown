---
layout: post
title:  "Effective engineering on Tetris"
date:   2025-11-24 16:30:00 +0900
categories: engineering
lang: en
---

The coolest project I've had the privilege to work on is the mobile version of Tetris at N3TWORK. It was ambitious - featuring an timezone-based realtime multiplayer mode where players in a single region logged on at exactly the same time to compete with the same pieces. It also featured Royale multiplayer mode - basically the mobile version of Tetris 99. This wasn't a fast-follow. N3TWORK had already prototyped the game mode before Tetris 99 was released.

Not only was the project ambitious, we managed to pull it off with an international team across South and North America, on an aggressive schedule where we rebuilt the game for worldwide release in months, not years.

How were we able to do this? Even though it wasn't always roses and sunshine, our team had a lot going for us that made extremely effective.

# Low architectural debt

N3TWORK had already delivered, to great success, a game called Legendary: Game of Heroes. Thanks to the brilliance of the engineering leaders in our org, the game's architecture was prescriptive enough to standardize how feature were built, while being flexible enough to make almost anything possible.  Tetris inherited much of that game's architecture.

Why is prescription a good thing? Because it allows developers to not spend time on problems that the architecture already solves. Problems like:

- How are we going to configure this feature?
- Where is the server going to put new state?
- How do we guarantee that state consistency on both server and client?
- How is the client going to receive new state?
- How is the client going to receive new assets?
- How is the client going to project state onto the player's screen?

These may seem like obvious questions with obvious answers, but having been through a few companies I can say that not everyone can come up with elegant solutions to them. We were lucky to have people that did. As a result, our projects benefited immeasurably from the time developers saved by not having to solve these problems on their own, with solutions of varying quality.

# Great team

The majority of the team that developed Tetris was in Chile. I was one of a few engineers based in San Francisco. There must be something about Chilean culture, because our team just worked really well together. There was trust, respect, and dialog - all values of the company culture - but with Tetris it felt like something that was already there, and you didn't need a company to prescribe it to you.

Everyone was also very strong and accountable in their roles, relaxing the need for oversight and synchronous communication, both of which are more difficult in a remote environment.

# Almost no code review

"No code review" may seem like an engineering antipattern but I can say confidently that our velocity benefited from having almost no code review:

- Obviously, there was no time spent waiting for someone to review. On a small, capable team with low architectural debt, it's possible to do this without creating bugs that would cancel out the time saved.
- Increases accountability and ownership. _You_ are responsible for your changes, and no one else!
- No barrier to change. Created a bug? Fix it now. See something that could be improved? Just improve it.

Of course, this created other kinds of issues:

- Lower awareness of code you don't own.
- Less opportunity to learn from others.
- High bus factor.

Despite the drawbacks, it did help us ship a lot faster, which is usually what a startup needs. Different companies of different sizes may have different priorities.

# Empowered individuals

The common thread in all of the points above is that we had all of the pieces in place to empower individuals to give their very best. 

We had a bug-resistant and somewhat prescriptive architecture, allowing lower oversight. Lower oversight means lower barriers to change, allowing individuals to be more effective. When individuals are more effective, they will see that they are making an impact, and ultimately care more about what they're doing. In my view, we had all of these things going for us, and that's what made Tetris a great team to be on.

While the company no longer exists, I'm thankful to N3TWORK for making a team like this possible.
