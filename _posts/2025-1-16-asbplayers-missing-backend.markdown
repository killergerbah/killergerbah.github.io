---
layout: post
title: "asbplayer's missing backend"
date: 2025-1-18 12:40:00 +0900
categories: engineering
lang: en
---

## Motivation

[asbplayer](https://github.com/killergerbah/asbplayer){:target="\_blank"} is hardly a complete application, especially when compared to similar software like [LingQ](https://www.lingq.com/){:target="\_blank"}, [Migaku](https://migaku.com/){:target="\_blank"}, [Kimchi Reader](https://kimchi-reader.app/){:target="\_blank"}, and [jpdb](https://jpdb.io/){:target="\_blank"}. There's an endless list of "missing" features, but it looks like the bare minimum feature-set that users are even willing to pay for is:

- Flashcard creation from target language content (already possible with asbplayer)
- SRS flashcard review system
- "Known word" tracking
- Built-in lemmatization and dictionary lookups

asbplayer has always been built to let users choose their own tools to serve these needs,
and indeed, many users use [Yomitan](https://github.com/yomidevs/yomitan){:target="\_blank"} for dictionary lookups and [Anki](https://apps.ankiweb.net/){:target="\_blank"} for flashcards.
A truly motivated learner will do whatever is necessary to accomplish their goals,
and asbplayer lets such learners have that responsibility by not locking them into an ecosystem of tools,
maintaining compatibility with the de-facto best tools that exist.

However, such a philosophy should not preclude making the benefits of asbplayer available to as many as people as possible - including those who might not be tools-aware enough to install two or three separate apps for a full language-learning toolset.
Ultimately, to make asbplayer truly accessible, it should include essential features out of the box.
That's where the backend comes in.
Users will always be given the choice - whether to use the asbplayer-hosted backend, to self-host the backend, or to screw all that and just keep using Anki.

## The backend

asbplayer needs a backend to permanently persist the user's language-learning journey.
Most importantly, this means flashcards and their review state.
It goes without saying, but storing this data locally would make the data susceptible to permanent loss,
whereas storing this data on the Internet not only avoids this problem,
but makes it available for further use cases - such as a flashcard review app,
similar to what Anki has done with their mobile apps and [Anki Web](https://ankiweb.net/about){:target="\_blank"}.

Hosted services cannot be provided completely for free, but users will always be encouraged to use the tools they prefer.
Anki is currently the best flashcard app now and for the foreseeable future.
Additionally, all backend code will be licensed under the [AGPL](https://www.gnu.org/licenses/agpl-3.0.html){:target="\_blank"} license,
and so will forever remain free to the public to modify and self-host if they wish - I am inspired by [Plausible](https://plausible.io/blog/open-source-licenses){:target="\_blank"}
whom I admire for having built a business while remaining open source.

## Technical thoughts

Below are the technical thoughts I have had so far given the constraints of the project: low-cost, optional, and reusable.
As with any large ambition, they are subject to change.

- SRS will be implemented via [FSRS](https://github.com/open-spaced-repetition){:target="\_blank"} which appears to be the state-of-the-art in spaced repetition algorithms.
- The backened will be implemented in Go. I have very little experience in Go, but this side projects will be an opportunity to learn. Go is not only a performant language (which will help with costs), but one that should benefit me professionally to learn well.
- A relational database will be used for persistence. This will ensure that the data access is as adaptable as possible to any future use cases that I surely have not thought of so far.
- To reduce cost, I will not consider any additional infrastructure, such as remote caches, until absolutely necessary.
- A large cloud provider like AWS will not be used for hosting.
- To preserve compatibility with existing tools, the flashcard API will implement a subset of [AnkiConnect's API](https://git.foosoft.net/alex/anki-connect){:target="\_blank"}.
- To preserve optionality, a flashcard-syncing Anki plugin _should_ be implemented that allows users to choose which flashcard app to use without repercussion.
- Some authentication is necessary to prevent abuse of the API, but it should be as painless as possible. Google provides a [way](https://developer.chrome.com/docs/extensions/how-to/integrate/oauth){:target="\_blank"} to do this for users of the extension on vanilla Chrome.
  Other OAuth providers can be used to accomodate other types of users.
- Lemmatization and dictonary lookups should remain offline.
  Wiktionary may offer a way to legally provide free dictionary lookups in multiple languages.
- The frontend should probably be a single-page-app to offload rendering compute onto the user's device, and it will probably use React. However, I would like to try a framework other than Material UI.

## The feature no one asked for

Finally, I believe the asbplayer backend will help turn language-learning through content - normally a solitary journey - into something that is more collective.
I have not seen a single language-learning application skilfully implement a social network of learners - but how cool would it be to be able to see how fellow learners are doing,
and what content they are consuming, alongside you?
Flashcards offer a natural way to know that. Nobody asked for it, but I think it would be pretty cool.
