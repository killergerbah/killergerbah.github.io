---
layout: post
title: "少し変わったやり方で、スワイプで閉じれるHTML要素を実装してみました"
date: 2024-12-23 11:11:00 +0900
categories: engineering
lang: ja
---

僕はもともと[asbplayer](https://github.com/killergerbah/asbplayer){:target="\_blank"}をスマホ用に作りませんでしたが、今となっては大分使えるようになってきました。スマホでの使用を可能にするためには、asbplayerの機能をアクセスできるオーバーレイを実装し、それを動画要素の上に乗せました。ただし、オーバーレイが動画プレイヤーのUIと被ったり、単に邪魔だったりする可能性があるため、スマホの画面の狭さへの配慮が重要で、オーバーレイを簡単に閉じれるようにしたほうが良いかと考えました。

最初は、閉じるための「X」ボタンをオーバーレイに加えました：

<video controls width="100%" src="/assets/videos/swipeable-panel-in-asbplayer-0.mp4"></video>

とりあえずの解決方法としては良かったですが、もっとボタンをオーバーレイに詰めようと思ったとき、オーバーレイの領域をなるべく効率的に使う必要がでてきました。それで、ボタンではなく、スワイプで閉じれるようにすれば良いのではないか、という解決方法に至りました。オーバーレイの領域の一部が空くのだけではなく、普通のスマホユーザーにとって、「スワイプで閉じる」という操作には馴染みがあるでしょう。

たまたまそのとき、[scroll snap](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_scroll_snap){:target="\_blank"}というCSSプロパティーのことを知ったばかりでした。「スワイプで閉じる」という操作は、「スクロールして視野から飛ばす」のと同じようなものだと考えると、もしかしたらこのCSSプロパティーを使えるのではないかと思いました。他の方法を調べようともせず、早速やってみることに取り組みました。

オーバーレイを上にスコールできるようにするためには、オーバーレイの下に何かを置く必要があります。この要素を「スクロールバッファー」だと呼びます。オーバーレイはすでに`iframe`の中に入っているので、実装の段取りは次の通りになります。

- `iframe`の中で、スクロールバッファーの要素をオーバーレイの要素の下に置きます。
- この二つの要素とも、スナップスクロール可能な子要素にするために、`scroll-snap-align`のプロパティーを付けます。
- `iframe`の`body`に`snap-scroll-type`のプロパティーを付け、以上の子要素の親要素にします。
- スクロールバーを[scrollbar-width](https://developer.mozilla.org/en-US/docs/Web/CSS/scrollbar-width)のCSSプロパティーで隠します。
- [scrollend](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollend_event)のイベントにサブスクライブし、`iframe`がスクロールの最終点に着いたタイミングに反応し、タッチのインプットが下の要素に届くように`iframe` をDOMから外します。

数時間で、91行のコードによってちゃんと動くようになりました：

<video controls width="100%" src="/assets/videos/swipeable-panel-in-asbplayer-2.mp4"></video>

スクロールバッファを赤にすると、以下の通りになります：

<video controls width="100%" src="/assets/videos/swipeable-panel-in-asbplayer-1.mp4"></video>

グーグルでこの問題を調べてみると、タッチのイベントでスワイプのジェスチャーを探知し、CSSで要素を画面から飛ばしたりといったような、主にJSを使った解決方法がでてきます。asbplayerで多く使われているMUIも、`Swipeable Drawer`というコンポーネントがあります。一見、ぴったりのソリューションには見えますが、CSS寄りの方法を使うことで[`Swipeable Drawer`のコードの2KB](https://v4.mui.com/components/drawers/#swipeable){:target="\_blank"}をasbplayerのビルドに加えないで済んだらしいです。さらに、`snap-scroll`のCSSのほうが、ブラウザのレンダリングエンジンによって動かされていて、JSがほとんど間に入らないため、すらすらと動きます。