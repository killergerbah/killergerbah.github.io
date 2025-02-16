---
layout: post
title: "Building a largish browser extension project using Vite"
date: 2025-2-16 11:43:00 +0900
categories: engineering
lang: en
---

# Why?

I wouldn't have thought to migrate asbplayer to Vite were it not for a [bug](https://github.com/killergerbah/asbplayer/issues/644){:target="\_blank"}. asbplayer managed to trip a rare edge-case bug in MUI4, and [with great effort](https://github.com/killergerbah/asbplayer/commit/1714bb4336abc6ff10529fea150ab37b216715de){:target="\_blank"} I managed to work around it, but it occurred to me that rather than depending MUI 4 which has been unsupported for years, I should consider upgrading to MUI 6 where the bug is likely to have been already fixed. Looking into this more, I found that Pigment CSS is to be the default styling solution going forward, and [Vite is the only (frontend) build system that currently supports it](https://mui.com/material-ui/migration/migrating-to-pigment-css/#supported-frameworks){:target="\_blank"}. Whether to use Pigment CSS is another issue, but in any case I got the sense that Vite is the current hotness, and it might make sense to move away from Webpack, which currently takes 30 grueling seconds to produce a development build of asbplayer. What I didn't realize was that that building a browser extension with Vite takes a little bit more work than your standard webapp. There doesn't seem to be a lot of information on this, so I thought it might be useful to talk about my own experience.

# Method

It goes without saying, but client-side-rendered webapps tend to be bundles of JavaScript referenced by an entrypoint `index.html`. Tools like Vite convert source code into such distributable, entrypoint files. However, whereas such a webapp is likely to have only one entrypoint, a browser extension is likely to have many. In most cases, at least a content script and background script are generated, with separate constraints for each type of artifact.

Vite is great for generating ES Modules, and indeed, it can be used to simultaneously build several ES Modules very quickly. The problem is that content scripts cannot be used as ES Modules - you are forced to build them as an "immediately invoked function expression," which Vite is capable of only building one-at-a-time. I did naiively try to build _all_ entrypoints as IIFEs, but this method took a whopping 60 seconds to build asbplayer.

Therefore, to have any hopes of speeding up build time, I had to build non-module entrypoints and module entrypoints separately. To do this, you could execute `vite build` with separate config files, but I chose to use to directly call `build` on an inline config instead, as I could carefully adjust the script behavior as necessary. Below is what asbplayer's build script now looks like.

```typescript
import { build } from "vite";

// ...

const moduleEntryPoints = {
  background: "./src/background.ts",
  "side-panel": "./src/side-panel.ts",
  "settings-ui": "./src/settings-ui.ts",
  "popup-ui": "./src/popup-ui.ts",
  "anki-ui": "./src/anki-ui.ts",
  "video-data-sync-ui": "./src/video-data-sync-ui.ts",
  "video-select-ui": "./src/video-select-ui.ts",
  "ftue-ui": "./src/ftue-ui.ts",
  "mobile-video-overlay-ui": "./src/mobile-video-overlay-ui.ts",
  "notification-ui": "./src/notification-ui.ts",
  "offscreen-audio-recorder": "./src/offscreen-audio-recorder.ts",
};

const nonModuleEntryPoints = {
  ...Object.fromEntries(
    glob
      .sync("./src/pages/*.ts")
      .filter((p) => p !== "./src/pages/util.ts")
      .map((filePath) => [
        filePath.substring(
          filePath.lastIndexOf("/pages") + 1,
          filePath.length - 3
        ),
        filePath,
      ])
  ),
  "mp3-encoder-worker": "../common/audio-clip/mp3-encoder-worker.ts",
  "pgs-parser-worker": "../common/subtitle-reader/pgs-parser-worker.ts",
  video: "./src/video.ts",
  page: "./src/page.ts",
  asbplayer: "./src/asbplayer.ts",
};

const entries = Object.entries(nonModuleEntryPoints);
const firstEntry = entries[0];

// Build the first entry point synchronously
// Its config is setup to clean the build folder
await build(configForNonModuleEntryPoint(firstEntry[0], firstEntry[1], 0));

// Build the rest asynchronously, in parallel
const promises: Promise<any>[] = [];

for (let index = 1; index < entries.length; ++index) {
  const [name, path] = entries[index];
  promises.push(build(configForNonModuleEntryPoint(name, path, index)));
}

// ES Module entrypoints can be built with a single invocation of `build`
promises.push(build(configForModuleEntryPoints(moduleEntryPoints)));
await Promise.all(promises);
```

This script achieved a build time of 13 seconds a 2x speedup over Webpack! This is a pretty big win for productivity.

## Conclusion

Do I recommend Vite to other browser extension developers? It's honestly hard to say. I've read that Webpack can be used with `esbuild` to go really fast, so it's not clear that Vite is always the faster option. Additionally, as you can see from the build script, `build` invocations increase with your content script count, and could make the build much slower, without bound. Developers will need to make the decision that works for their project.
