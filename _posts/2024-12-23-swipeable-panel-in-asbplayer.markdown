---
layout: post
title: "A different approach to implementing swipeable panels in the browser"
date: 2024-12-23 11:11:00 +0900
categories: engineering
lang: en
---

I did not originally build [asbplayer](https://github.com/killergerbah/asbplayer){:target="\_blank"} to be used on smartphones, but it has come a long way.
To make this possible, I implemented an overlay UI on top of streaming videos that allow users to interact with asbplayer through
buttons, rather than keyboard shortcuts. There are obvious UI/UX challenges with such a feature. On small screens, the overlay can take up a lot of space, and may get in the way of existing video player controls. So, it makes sense to provide a way to dismiss the overlay, but I did not try to think of an elegant solution to this problem until now.

My initial solution was to simply give the overlay an X button that closes the overlay when tapped:

<video controls width="100%" src="/assets/videos/swipeable-panel-in-asbplayer-0.mp4"></video>

This was fine as a temporary solution, but recently I had to pack more buttons into the overlay, and I needed something else to make the overlay less crowded. It didn't take long to realize that a swipeable overlay would not only solve this problem, but would also be more intuitive to the average smartphone user.

Coincidentally, I was also experimenting with [scroll snap](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_scroll_snap){:target="\_blank"} CSS properties for a different part of the overlay. While probably not an intended use-case, this seemed like something I could use for overlay dismissal. After all, "swipe-to-dismiss" is really the same thing as "scroll-the-thing-off-screen." And the snappiness of the scrolling will make it clear that swiping is only for dismissing, with no in-between states. Without doing much further research, I allowed myself to get sucked into this idea.

The insight needed for this to work is that swiping up to dismiss requires the overlay to be scrolled upward, which in turn requires that there is something below the overlay that can be scrolled into view. I'll refer to this as the "scroll buffer." The overlay is already inside an `iframe`, so the problem can be broken down:

- Add the scroll buffer DOM element inside the `iframe`, below the overlay DOM element.
- Give both of these elements a `scroll-snap-align` property to make them snap-scrollable children.
- Give the `iframe` body a `snap-scroll-type` to make it a snap-scrollable parent.
- The scroll bar can be hidden using [scrollbar-width](https://developer.mozilla.org/en-US/docs/Web/CSS/scrollbar-width).
- Once the `iframe` has reached its final position, it can be completely removed so that it no longer absorbs touch inputs using the [scrollend](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollend_event) event.

After a few hours of fiddling and [91 lines later](https://github.com/killergerbah/asbplayer/commit/0f0b496c3015b19217081596f0bfb79f9df8d5dd){:target="\_blank"}, I got something that works:

<video controls width="100%" src="/assets/videos/swipeable-panel-in-asbplayer-2.mp4"></video>

And with the scroll buffer highlighted in red:

<video controls width="100%" src="/assets/videos/swipeable-panel-in-asbplayer-1.mp4"></video>

The top solutions to this problem on Google are more JS-heavy, using touch event hooks to detect swipes and then finally animate the DOM element off-screen. Even MUI, which is heavily used within asbplayer, has its own `Swipeable Drawer` which at first glance seems like an exact use-case match for my problem. However, this is [2KB of code](https://v4.mui.com/components/drawers/#swipeable){:target="\_blank"} that I have apparently avoided by using the more CSS-heavy approach instead. Additionally, the CSS approach feels and performs better, since the snappy physics are presumably implemented directly within the browser's rendering engine, with no JS between user interactions and the actual scrolling.
