---
layout: post
title: "ブラウザ拡張機能をViteでビルドする方法"
date: 2025-2-16 11:43:00 +0900
categories: engineering
lang: ja
---

# きっかけ

asbplayer を Webpack から Vite に移動するきっかけとなったのは、直接 DX に関係があるようなものではなく、[バグ](https://github.com/killergerbah/asbplayer/issues/644){:target="\_blank"}でした。MUI4 にはクラッシュを起こすようなバグがあり、[かなり無理をして](https://github.com/killergerbah/asbplayer/commit/1714bb4336abc6ff10529fea150ab37b216715de){:target="\_blank"}それを回避することができましたが、数年も更新がない MUI4 に依存するよりも、そのバグがおそらくすでに排除されている最新の MUI6 を使ったほうが良いのではないかと思えるようになりました。そして、その方法を調べたところ、スタイリングのソリューションとしてこれから推奨となる Pigment CSS に対応できているのがフロントエンドのビルドシステムで [Vite 以外のものはない](https://mui.com/material-ui/migration/migrating-to-pigment-css/#supported-frameworks){:target="\_blank"}と知りました。Pigment CSS を使うかどうかは別として、とにかく Vite が JS のビルドシステムとして主流となっていると気づいて、asbplayer をビルドするのに 30 秒もかかる Webpack から Vite に移動しようと思いました。ただし、その作業に取り組むまで知らなかったのは、主なユースケースとしてウェブアプリのビルドに使われている Vite が、拡張機能のような少し変わった JavaScript アプリをビルドするのに最適化されていないことです。その現状に関する情報が少ないようなので、自分の経験を語ってみたいと思います。

# 方法

一般的なウェブアプリを表示するのにブラウザが `html` ファイルをサーバーからダウンロードし、ファイル内で参照されえている JavaScript を実行します。ソースコードをインプットとして吸収し、実行可能なエントリーポイントとなる `index.html` を生成するのに Vite のようなビルドツールを、設定をあまりいじらないでほぼそのまま使えます。それに対して、asbplayer のような大きなブラウザ拡張機能は、実行可能なエントリーポイントが複数あってもおかしくないです。少なくとも、生成しないといけないビルドのアーティファクトとしてコンテントスクリプトとバックグラウンドスクリプトがあり、それぞれのアウトプットに別の条件がつきます。

Vite は ES Modules を生成するのに最適化されており、複数の ES Modules を生成するのにもってこいですが、問題は、このブログを書いている時点では、バックグラウンドスクリプトと違ってコンテントスクリプトは ES Module として使えません。全ての JavaScript アーティファクトを即時実行関数式としてビルドすることができますが、Webpack と違って Vite はビルド実行一回にあたり、即時実行関数式のアーティファクトを一つしか生成できません。実際これを試してみたら、Vite のビルドが Webpack の 2 倍も時間がかかりました！

なので、Vite でのビルド時間の最適化を目指すなら、ES Module として使用可能な部分と、そうではない部分を別にビルドするしかありません。複数のコンフィギュレーションファイルを書いて、数回`vite build`を実行する選択肢もありますが、僕はインラインのコンフィギュレーションを使ってコードで直接`build`を実行する方法を採用しました。それで、ビルドの動作を細かく調整することができました。asbplayer のビルドスクリプトは以下のようなものです。

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

// 一番最初のエントリーポイントを同期的にビルド
// ビルドのアウトプットフォルダーをまず削除するように設定してあります
await build(configForNonModuleEntryPoint(firstEntry[0], firstEntry[1], 0));

// 残りのビルドのアウトプットを非同期的に生成し、積み重ねます
const promises: Promise<any>[] = [];

for (let index = 1; index < entries.length; ++index) {
  const [name, path] = entries[index];
  promises.push(build(configForNonModuleEntryPoint(name, path, index)));
}

// ES Moduleの部分 は一発で生成します
promises.push(build(configForModuleEntryPoints(moduleEntryPoints)));
await Promise.all(promises);
```

これで asbplayer のビルド時間が Webpack の半分になりました！生産性の面でかなり得しました。

## 結論

この経験を経て拡張機能プロジェクトのビルドに Vite を勧めるかと聞かれたら、Vite が選択肢として一番良いとは言い切れません。例えば、Webpack を `esbuild` と一緒に使うことができるらしくて、Vite よりも Webpack でビルドを短くできるかもしれません。それに、以上のスクリプトから分かる通り、コンテントスクリプトが多ければ多いほど、`build`の実効数が増えて、ビルド時間が大幅に遅くなってしまう可能性もあります。自分のプロジェクトに合わせて判断するしかありません。
