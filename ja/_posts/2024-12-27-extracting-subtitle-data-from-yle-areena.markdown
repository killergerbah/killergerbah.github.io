---
layout: post
title: "「Yle Areena」の動画の字幕データ抽出"
date: 2025-01-02 02:23:00 +0900
categories: engineering
lang: ja
---

## 動画配信サービスの字幕データ抽出

[asbplayer](https://github.com/killergerbah/asbplayer){:target="\_blank"}のユーザーからよくお願いされるのは、特定の動画配信サービスの字幕データ抽出です。こういったリクエストに対応するのは、そんなに時間がかかるものでもありませんし、楽しいリバースエンジニアリング問題だったりしますので、ほとんどの場合は、喜んでやってみます。それに、asbplayer の良いところは、動画配信サービスとの連携が非常に簡単で、字幕データの探知と抽出を実装すれば良いだけです。

どの動画配信サービスでも、字幕データを抽出するには、大体以下の手順を踏みます：

- ブラウザのネットワークモニタリングツールを使い、`.har`ファイルをよく見、そこから字幕関連のやりとりを突き止めます。多くの場合、`vtt`形式のファイルの`GET`がでてきます。
- さらにその URL の出所を突き止めます。`m3u8`や`mpd`といったビデオマニフェストファイルなど、あるいはそのサイト専用のカスタムなマニフェストに字幕ファイルの URL が記載されたりします。
- 遡ってマニフェストの URL の出所を探します。
- こうやって突き止めたリクエストやリスポンスから字幕データの URL を抽出し保存します。大体の場合、HTTP のやりとりに使われている`fetch`、`JSON.parse`, `JSON.stringify`, `XHRWebRequest`といった関数に割り込み、必要なデータを取ります。
- 最後に字幕データの URL を`document.dispatchEvent`を通して asbplayer に渡します。

## Yle Areena

最後に配信サービスの字幕抽出を頼まれたのは、フィンランドで人気の「Yle Areena」という動画配信サービスです。結局、他のサイトとは作業がさほど変わりませんでしたが、まだ記憶に残っていますので、この経験について書いてみたいと思います。

### [マルチバリアントプレイリスト](#補足)の取得

Yle Areena は動画配信のために`m3u8`のマニフェストファイルが使われています。asbplayer はすでに Disney Plus との連携で`m3u8`ファイルを扱っていますので、Yle Areena との連携を書くに当たり、僕は`m3u8`について何も知らなかったというわけではありませんでした。Disney Plus と同じく、Yle Areena は`m3u8`ファイルを使い、セグメントという複数のファイルに字幕データを分割します。ただし、Disney Plus と違い、`m3u8`ファイルの URL は直接サーバーから渡されません。代わりに、URL がペイジに埋め込められているか、クライアント側のコードによって計算されているかと思われます。なので、`m3u8`の URL を手に入れるためには、HTTP リクエストを処理する関数に割り込み、ペイジが`m3u8`をリクエストする直前に URL を盗み取るという、違うやり方を採用しなければなりませんでした。そのために`XMLHttpRequest.open`に割り込むコードが以下のようになります。

```typescript
// Yle Areenaはm3u8を取得するためには、window.fetchではなく、XMLHttpRequestが使われているようです
const originalXhrOpen = window.XMLHttpRequest.prototype.open;

window.XMLHttpRequest.prototype.open = function () {
  // アローではなく、functionシンタックスを使うとargumentsキーワードが参照できるようになります
  // これで、全ての引数を特定しないで済みます
  const url = arguments[1];

  if (typeof url === "string" && /https:\/\/.+\.m3u8.+/.test(url)) {
    // m3u8のURLがでてきたら、保存します
    lastManifestUrl = url;
  }

  // XMLHttpRequest.openの実行を続けます
  originalXhrOpen.apply(this, arguments);
};
```

### [メディアプレイリスト](#補足)の取得

字幕トラックごとのメディアプレイリストの URL を取得するためには、保存したマルチバリアントプレイリストを再リクストできます。

```typescript
// マルチバリアントプレイリストをfetch
const manifest = await(await fetch(lastManifestUrl)).text();

// マルチバリアントプレイリストに書かれているメディアプレイリストのURLは相対URLです
// そのベースURLを計算します
const m3U8UrlObject = new URL(lastManifestUrl);
let dataBaseUrl = `${m3U8UrlObject.origin}/${m3U8UrlObject.pathname}`;
dataBaseUrl = dataBaseUrl.substring(0, dataBaseUrl.lastIndexOf("/"));

// オープンソースのm3u8ファイルのパーサーはこちらです： https://github.com/videojs/m3u8-parser
const parser = new Parser({
  url: lastManifestUrl,
});
parser.push(manifest);
parser.end();
const parsedManifest = parser.manifest;
const subGroups = parsedManifest.mediaGroups?.SUBTITLES;

for (const [category, group] of Object.entries(subGroups)) {
  for (const [label, info] of Object.entries(group)) {
    const subtitleTrackMediaPlaylistUrl = `${dataBaseUrl}/${info.url}`;
    // ...最終的には、字幕のメディアプレイリストURLと他の字幕関連のデータをasbplayerに渡します
  }
}
```

ついでの話ですが、マニフェスト URL をブラウザのアドレスバーにコピペして読み込もうとしたら、認証エラーの 403 のリスポンスが返されます。これで URL がユーザーセッション外で使えないようにセキュリティがかかっているのが分かります。ただし、それが asbplayer のような拡張機能が URL を使用することへの妨害にはなりません。コンテントスクリプトで URL を普通に `fetch` すれば、ブラウザが勝手にセッションを特定するパラメータを付けてくれますので、200 のリスポンスが返ってきます。動画配信サービスの中では、Yle Areena が特別に高度なセキュリティを実装したらしいです。

## 実際の字幕ファイルの取得

字幕トラックのメディアプレイリストには、実際の字幕ファイル URL が参照されています。それらを `fetch` すると次のようになります。

```typescript
// メディアプレイリストをfetch
const mediaPlayListResponse = await fetch(mediaPlayListUrl);

// fetchしたものをパース
const parser = new Parser();
parser.push(await mediaPlayListResponse.text());
parser.end();

// 載っているURLは全部相対URLなので、ベースURLを計算します
const dataBaseUrl = mediaPlayListUrl.substring(
  0,
  mediaPlayListUrl.lastIndexOf("/")
);

// 計算された絶対URLを全部同時にfetch
const promises = parser.manifest.segments
  .filter((s: any) => !s.discontinuity && s.uri)
  .map((s: any) => fetch(`${dataBaseUrl}/${s.uri}`));

for (const p of promises) {
  const subtitleData = await(await p).blob();
  // ...
}
```

## 字幕ファイルのマージ

Yle Areena が Disney Plus とまた違うのは、字幕が分割したセグメントの数が圧倒的に多いことです。asbplayer ユーザーはセグメントを全部ダウンロードするには時間がかかりますが、普通のユーザーにとっては字幕をもっと分割したほうがバッファリング時間が短くなりますのでストリーミングの UX の最適化が狙いだと考えられます。結果として、最終的にダウンロードするデータの量重くなりますし、実際には、セリフが重複しているセグメントがほとんどなので、少なくともデータ量が 2 倍になると予測できます。とにかく、字幕をマージをした後、字幕の重複排除が必要です。

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

## まとめ

ソースコードの diff は[こちら](https://github.com/killergerbah/asbplayer/commit/d4c2031463d8670d79cd50114470d6799a8a49cf){:target="\_blank"}です。コードが Yle Areena で動いているところが観れる動画は以下です：

<video controls preload width="100%" src="/assets/videos/yle-areena.mp4"></video>

字幕を全部ダウンロードするには 30 秒近くもかかります。セグメントが 400 以上もあるからです！

## 最後に感謝を

YouTube と Netflix の字幕抽出を実装した Renji-XD の[貢献](https://github.com/killergerbah/asbplayer/pull/107){:target="\_blank"}がなければ、asbplayer には字幕抽出の機能がずっとないままだったかもしれません。Renji-XD と asbplayer に貢献してくれた他のみなさまには感謝しています。動画配信サービス全部の字幕抽出コードは[こちら](https://github.com/killergerbah/asbplayer/tree/main/extension/src/pages){:target="\_blank"}です。

## 補足

**マルチバリアントプレイリスト**と**メディアプレイリスト**は、HLS というメディア配信プロトコルで使われる用語です。詳しくは[こちら](https://developers.play.jp/entry/2023/05/26/171431)を参考にしてください。
