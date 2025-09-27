---
layout: post
title: "イベントサブスクリプションのAPI設計"
date: 2025-9-27 18:21:00 +0900
categories: engineering
lang: ja
---

イベントサブスクリプションを実装するには様々なやり方があります。これまでみてきたいくつかのデザインについて書いてみたいと思います。

## Spring の `@EventListener`

Spring のイベントサブスクリプションはシンプルで Java アノテーションをベースにした設計を採用しています。サブスクリプション解除は完全に Spring に丸投げしても良いです。

```java
@Service
public final class ListeningSpringBean {
    // ...

    @EventListener
    public void onSomeEvent(SomeEvent event) {
        // イベントをハンドル
    }
}
```

## React の useEffect

React では関数を状態変更にサブスクライブすることができます。React は Component のライフサイクルを管理しているお陰、サブスクリプション解除は考えなくても良いです。

```typescript
const Component = () => {
  const [flag, setFlag] = useState<boolean>(false);

  useEffect(() => {
    // flagが切り替えたときの処理
  }, [flag]);

  // ...
};
```

ちなみに、`useEffect`に渡す関数の戻り値はイフェクトが次実行される前に実行されるので、`useEffect`を使って他のイベントへのサブスクライブを管理することができます。

```typescript
const Component = () => {
  useEffect(() => {
    const listener = (event: MessageEvent) => {
      // イベントをハンドル
    };
    document.addEventListener("message", listener);
    return () => document.removeEventListener("message", listener);
  }, []);

  // ...
};
```

手動で `document.removeEventListener`を呼ぶ必要があるのが残念なところですが、API の設計に
後処理が組み込まれているため、非常にミスしにくいです。

## Unity の Monobehaviour ライフサイクルメソッド

Unity の`MonoBehaviour`のライフサイクルメソッドは自動的にライフサイクルイベントにサブスクライブされます。

```csharp
public class SomeComponent : MonoBehaviour {
    void OnEnable() {
        // Behaviorがアクティブになった時の処理
    }

    void OnDisable() {
        // Behaviorが非アクティブになった時の処理
    }

    // …などなど
}
```

## CSharp のイベント

CSharp はイベントが言語の機能として組み込まれています。Unity のコードベースでは、以下の使い方が想像できます。

```csharp
public class Publisher {
    public event Action OnSomethingHappened;

    // ...
}

public class SomeComponent : MonoBehaviour {

    // ...

    void OnEnable() {
        // BehaviorがアクティブになったときにOnSomethingHappenedにサブスクライブ
        _publisher.OnSomethingHappened += HandleSomethingHappened;
    }

    void OnDestroy() {
        // Behaviorが破棄されたときにOnSomethingHappenedへのサブスクライブを解除
        _publisher.OnSomethingHappened -= HandleSomethingHappened;
    }

    void HandleSomethingHappened() {
        // イベントが発生した時の処理
    }
}
```

ところが、上のコードにはバグがあります！`OnEnable`は複数実行される可能性があるのに対し、`OnDestroy`は一回しか実行されません。防御的なサブスクリプション解除で直せます。

```csharp
public class SomeComponent : MonoBehaviour {
    // ...

    void OnEnable() {
        // ブスで防御的なサブスクリプション解除とサブスクライブ
        _publisher.OnSomethingHappened -= HandleSomethingHappened;
        _publisher.OnSomethingHappened += HandleSomethingHappened;
    }

    // ...
}
```

# 設計の比較

以上の例は使いやすい順で並べてあります。

- Spring の`@EventListener`はイベントサブスクリプションの管理をミスることが不可能です。なぜかというと、Spring が代わりに管理してくれるからです。
- 一方、CSharp イベントのサブスクリプション管理は完全に開発者の責任なので、非常にミスしやすいです。

もちろん Spring の API はシングルトン Bean の概念に結びついており、CSharp のイベントと比べて非常に限られた目的のために設計されたのを承知しています。とはいえ、設計を比べることで何か発見があるかもしれません。

## CSharp イベントの欠点

まずは、CSharp イベントがなぜ使い勝手が悪いのか考えてみましょう。

### パブリッシャーへの参照が必要

イベントにサブスクライブするには`event`が必要なので、パブリッシャーへの依存が発生します。

### サブスクリプションがミスしやすい

上の Unity の例のように、間違って同じ`event`に数回サブスクライブするミスが可能です. さらに、`event`の状態が可視化されていないので、ミスの回避やデバッグは難しいです。

### サブスクリプション解除もミスしやすい

サブスクリプション解除はサブスクリプション時のパラメーターを繰り返し渡さないといけません。

- `event`への参照。
- サブスクライバーである関数への参照。

従って、開発者はサブスクリプション解除のパラメーターを間違えないように気をつけないといけません。

## フレームワークの利点

上記を踏まえて、すでにフレームワークをベースに開発を行っている開発者は、フレームワークが提供してくれるイベントアブストラクションを使うべきだと思っています。事実、CSharp イベントの欠点は Spring の`@EventListener`で見受けられません。

- パブリッシャーへの参照は不要
- サブスクライブしすぎることが不可能で、サブスクリプション解除を忘れることも不可能です。それはなぜかというと、Spring が Bean のライフサイクルに結びついているサブスクリプションを管理してくれるからです。

## こだわった API 設計の利点

イベントサブスクリプションが手元にあるツールによって解決されていなくても、自分で使いやすいイベントサブスクリプションの API を設計してみてはいかがでしょうか。上の比較を元に、イベント API をどう設計すべきか、いくつかのポイントを提案します。

- もしサブスクリプション解除が手動で行わないといけない場合, サブスクリプション時のパラメーターを覚えなくても良いように設計すると良いです。
- アプリケーションレベルのイベントはパブリッシャーを特定しなくてもサブスクライブすることができるように設計すると良いです。

例えば、CSharp イベントを採用するより、上のガイドラインに従って以下のインターフェース設計が考えられます。

```csharp
interface IEventPublisher
{
    /// <summary>
    /// <c>subscriber</c>を型が<c>TEvent</c>のイベントにサブスクライブします。
    /// </summary>
    /// <param name="subscriber">サブスクライブされる関数</param>
    /// <returns>
    /// サブスクリプションを解除するためのIDisposable。
    /// </returns>
    IDisposable Subscribe<TEvent>(Action<TEvent> subscriber);

    /// <summary>
    /// 肩が<c>TEvent</c>のイベントをパブリッシュします。
    /// </summary>
    void Publish<TEvent>(TEvent event);
}
```

もしくは、オープンソースのものを使っても問題ないでしょう。

# なぜこんな話をするのか

CSharp イベントの管理ミスが原因のバグを一つデバッグするのに数時間がかかってもおかしくないです。では、最初から使い勝手の良い API を提供するだけで、どれぐらいの時間が節約できるでしょうか。問題が発生する前に問題を解決するのがベストです。間違いなく、プロジェクトの初期段階の優れた API 設計がその方法の一つです。
