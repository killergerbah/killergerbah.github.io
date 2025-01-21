---
layout: post
title: "Extracting subtitle data from Yle Areena"
date: 2025-01-02 02:23:00 +0900
categories: engineering
lang: en
---

## Subtitle extraction from streaming video services

I am frequently asked by [asbplayer](https://github.com/killergerbah/asbplayer){:target="\_blank"} users to "please implement subtitle detection from streaming video service XYZ." Fulfilling these requests does not usually take a huge amount of effort and in many cases can be a fun reverse-engineering exercise, so I am usually happy to do this. If anything, asbplayer's advantage over similar software is its very light integration with any given streaming service, where usually the _only_ problem to solve is how to extract subtitles from the site.

Extracting subtitles usually works out to the following steps:

- Use the browser's network monitoring tool, and inspect the `.har` file, to determine subtitle-related traffic while streaming a video. Usually this is is a `GET` request for a subtitle file that ends in `.vtt`.
- Try to trace where the URL might have come from. Usually this is some manifest file with links related to the streaming video. A lot of websites use `.mpd` or `.m3u8` files, and yet others will use something custom.
- Continue going backwards - try to trace where this manifest file came from. If you're lucky it's a specific network request that just returns the manifest.
- Determine how to extract the subtitle URLs from those set of network requests. The most convenient way to do this is usually by hijacking web APIs used in the HTTP request/response loop - `fetch`, `JSON.stringify`, `JSON.parse`, `XHRWebRequest`, etc.
- Write the page script to extract the data and deliver it to asbplayer's content script via a `document.dispatchEvent`.

## Yle Areena

I was most recently asked to add subtitle detection for [Yle Areena](https://areena.yle.fi/tv){:target="\_blank"}, which is the apparently one of the biggest Finnish streaming platforms. This exercise was not so different from other websites, but it's fresh in my memory, so it makes sense to write about this particular experience.

### Obtaining the [multivariant playlist](https://developers.broadpeak.io/docs/foundations-hls#multivariant-playlist){:target="\_blank"} URL

Yle Areena uses an [m3u8 manifest](https://developers.broadpeak.io/docs/foundations-hls#manifest) for video streaming. asbplayer already has an `m3u8` integration for Disney Plus so this was not completely unfamiliar. Similar to Disney Plus, Yle Areena segments subtitles into multiple `vtt` files. Unlike Disney Plus, the multivariant playlist URL is apparently not supplied by the backend to the website - so one can surmise that it is either embedded into the page or derived by client-side code. However, the URL can still be intercepted as the Yle Areena web client requests it. Below is code that does hijacks `XMLHttpRequest.open` in order to do this.

```typescript
// Yle Areena uses XMLHttpRequest, not window.fetch, to request the m3u8 manifest
const originalXhrOpen = window.XMLHttpRequest.prototype.open;

window.XMLHttpRequest.prototype.open = function () {
  // Function (not arrow) syntax exposes the arguments keyword for referring to
  // the function's arguments without having to explictly specify all of them
  const url = arguments[1];

  if (typeof url === "string" && /https:\/\/.+\.m3u8.+/.test(url)) {
    // If we see an m3u8 URL, remember it
    lastManifestUrl = url;
  }

  // Continue execution of XMLHttpRequest.open
  originalXhrOpen.apply(this, arguments);
};
```

### Obtaining the [media playlist](https://developers.broadpeak.io/docs/foundations-hls#media-playlist){:target="\_blank"} URLs

When needed, the multivariant playlist URL can be re-fetched so that we can extract the media playlist URL for each subtitle track.

```typescript
// Fetch the multivariant playlist
const manifest = await(await fetch(lastManifestUrl)).text();

// URLs in the manifest are relative to the multivariant playlist base path.
// Derive the base path here.
const m3U8UrlObject = new URL(lastManifestUrl);
let dataBaseUrl = `${m3U8UrlObject.origin}/${m3U8UrlObject.pathname}`;
dataBaseUrl = dataBaseUrl.substring(0, dataBaseUrl.lastIndexOf("/"));

// An m3u8 parser is published here: https://github.com/videojs/m3u8-parser
const parser = new Parser();
parser.push(manifest);
parser.end();
const parsedManifest = parser.manifest;
const subGroups = parsedManifest.mediaGroups?.SUBTITLES;

// Iterate over each subtitle track
for (const [category, group] of Object.entries(subGroups)) {
  for (const [label, info] of Object.entries(group)) {
    const subtitleTrackMediaPlaylistUrl = `${dataBaseUrl}/${info.url}`;
    // ...eventually publish the URL + other track data to asbplayer
  }
}
```

It's worth noting that trying to `GET` the original manifest URL by pasting it into your browser's address bar will fail with a 403 (authorization failed) status code, despite the original URL already having some security built in (an `hmac` signature parameter). Apparently, some effort has been put into preventing these URLs from being useful outside of the user's session. For us, this does not matter so much since we can simply `fetch` the URL from the page script and the browser automatically provides any session variables that allows it to succeed. Not all streaming services have this level of security in front of content files.

### Fetching subtitle files

The media playlist is another `m3u8` file that contains relative URLs to the actual subtitle file segments. So fetching actual subtitle file data looks something like this:

```typescript
// Fetch the media playlist
const mediaPlayListResponse = await fetch(mediaPlayListUrl);

// And parse it
const parser = new Parser();
parser.push(await mediaPlayListResponse.text());
parser.end();

// All URLs provided by the playlist are relative so let's again derive the base URL
const dataBaseUrl = mediaPlayListUrl.substring(
  0,
  mediaPlayListUrl.lastIndexOf("/")
);

// Now we queue up fetches for the subtitle data
const promises = parser.manifest.segments
  .filter((s: any) => !s.discontinuity && s.uri)
  .map((s: any) => fetch(`${dataBaseUrl}/${s.uri}`));

for (const p of promises) {
  const subtitleData = await(await p).blob();
  // ...
}
```

### Merging subtitle file segments

Also different from Disney Plus - Yle Areena segments subtitles into far more files - I have observed each segment containing one or two lines of dialog each. It takes significantly longer for asbplayer users to download all the segments at once, but for normal users, using more segments in theory optimizes for shorter buffering time and therefore a better _streaming_ experience. Still, the total amount of data downloaded is much more - in fact, segments obtained from Yle Areena tend to overlap dialog frequently. After merging parsed subtitles, they must be deduplicated.

```typescript
const mergeSubtitleFiles = async (files: File[]) => {
  const allSubtitles = (
    await Promise.all(files.map((f, i) => _parseSubtitleFile(f)))
  )
    .flatMap((nodes) => nodes)
    .sort((a, b) => a.start - b.start);

  return _deduplicateSubtitles(allSubtitles);
};
```

### Put it all together

[Here](https://github.com/killergerbah/asbplayer/commit/d4c2031463d8670d79cd50114470d6799a8a49cf){:target="\_blank"} is the link to the commit with this change. And a video of the whole thing:

<video controls preload width="100%" src="/assets/videos/yle-areena.mp4"></video>

Note how long it takes for the extraction to complete - this is because there are 400+ segments to download!

## Addendum

asbplayer would never have had subtitle detection were it not for the [work](https://github.com/killergerbah/asbplayer/pull/107){:target="\_blank"} of Renji-XD, who contributed the initial subtitle detection feature to asbplayer, with support for Netflix and YouTube. Many thanks to you, and to all the other asbplayer contributors. All page-specific subtitle detection code can be found [here](https://github.com/killergerbah/asbplayer/tree/main/extension/src/pages){:target="\_blank"}.
