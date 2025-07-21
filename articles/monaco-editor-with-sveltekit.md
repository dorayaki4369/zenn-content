---
title: "Monaco EditorをSvelteKitで使ったメモ"
emoji: "📝"
type: "tech"
topics: ["monaco", "svelte"]
---

SvelteKitでManaco Editorを少し使ってみたので、備忘録として記事にしました。

Monaco EditorはVS Codeから作られたブラウザで動作するテキストエディタで、シンタックスハイライトや予測変換などといったVS Codeで使える機能を、手軽に自分のWebアプリケーションに組み込めるようになります。

今回は、Monaco EditorをSvelteコンポーネントとしてラップし、Web Worker周りの実装をしました。

## サンプル

サンプルアプリはGitHub Pagesで公開しており、以下のURLからアクセスできます。

[https://dorayaki4369.github.io/monaco-editor-with-sveltekit/](https://dorayaki4369.github.io/monaco-editor-with-sveltekit/)

リポジトリは以下になります。

[https://github.com/dorayaki4369/monaco-editor-with-sveltekit](https://github.com/dorayaki4369/monaco-editor-with-sveltekit)

サンプルアプリの機能は以下のとおりです。

1. Monaco Editorによるコード編集
2. 言語切り替え
3. Light/Darkテーマ切り替え

## 環境

使用した主要なライブラリは以下のようになります。

+ svelte ^5.0.0
+ sveltekit ^2.22.0
+ monaco-editor ^0.52.2
+ monaco-yaml ^5.4.0
+ typescript ^5.0.0
+ vite ^7.0.4
+ pnpm 10.13.1
+ Node.js 22.17.1

## プロジェクトの作成

まず、Svelteのドキュメントにある[Creating a project](https://svelte.dev/docs/kit/creating-a-project)を参考にプロジェクトを作成します。

```shell
npx sv create monaco-editor-with-sveltekit
cd monaco-editor-with-sveltekit
pnpm install
```

その後、[monaco-editor](https://github.com/microsoft/monaco-editor)をインストールすれば完了です。
今回はyamlにも対応するため、[monaco-yaml](https://github.com/remcohaszing/monaco-yaml)もインストールしています。

```shell
pnpm add monaco-editor monaco-yaml
```

## Editorコンポーネントの作成

Monaco Editorをラップする`Editor.svelte`コンポーネントを作成します。
完成系ですが、以下のようになりました。

https://github.com/dorayaki4369/monaco-editor-with-sveltekit/blob/main/src/lib/components/Editor.svelte

実装ポイントとして、このコンポーネントは全体的に以下のような構成で実装しました。

1. global stateのthemeをインポート
2. propsの定義
3. 内部で使用する変数の定義（初期化はしない）
4. Monaco Editor初期化関数
5. Monaco Editor初期化関数を実行する`onMount`
6. プロパティ変更時に実行されるMonaco Editorの状態を変えるスクリプト

重要なのは、**Monaco Editorはクライアントサイドで確実に初期化されるようにすること**と、**プロパティの変更をMonaco Editorに反映させる関数の実装**です。

```svelte
<script lang="ts">
  import type * as monaco from "monaco-editor";
  import { onMount } from "svelte";

  // global stateのthemeインポート
  import { theme } from "$lib/theme.svelte";

  // propsの定義
  let {...} = $props<...>();

  // DOMエレメントやeditor、monacoなど、内部で使用する変数の定義
  let container: HTMLElement;
  let editor: monaco.editor.IStandaloneCodeEditor;
  let m: typeof import("monaco-editor");

  // プロパティ変更時に実行されるMonaco Editorの状態を変えるスクリプト
  $effect(() => {...});

  ...

  // Monaco Editor初期化関数
  async function innitializeMonacoEditor() {
    ...
  }

  // Monaco Editor初期化関数の実行&dispose関数の実装
  onMount(() => {
    initializeMonacoEditor();

    return () => editor?.dispose();
  });
</script>

<div id="editor" bind:this={container} {...props} class={props.class}></div>
```

それぞれのポイントは以下のとおりです。

### 1. global stateの`theme`をインポート

このサンプルアプリでは、　ユーザーによって任意に変えられるLight/Darkテーマの切り替え機能の状態管理を、別モジュールの`$lib/theme.svelte.ts`ファイルにて行なっています。
`theme`変数を読み込み、Monaco Editorのテーマを切り替えられるようにします。

### 2. propsの定義

これはSvelteの[Rune](https://svelte.jp/blog/runes)を使ったプロパティの実装になります。

`value`(テキスト内容)をbindingすることで、このエディタで行ったコードの変更が親コンポーネントにも伝わるようにしています。

https://github.com/dorayaki4369/monaco-editor-with-sveltekit/blob/ff3ce2ff79341689e38ee1b9ab1f6597bdd05f80/src/lib/components/Editor.svelte#L6-L14

### 3. 内部で使用する変数の定義

内部で使用する変数として、エディタのコンテナとなるDOMエレメントの変数である`container`、エディタの変数である`editor`、`monaco`変数の3つがあります。

これらの変数は、`container`を除き、`onMount`変数で初期化され、`$effect`関数内で使用されます。

### 4. Monaco Editor初期化関数

Monaco Editorの初期化を行う関数を定義します。

この関数は`onMount`から切り出したもので、monacoを動的インポートし、`getWorker`関数の定義、エディタのインスタンス化をしています。
動的インポートをするのは、サーバーサイドでインポート処理が実行されないようにするためです。

`getWorker`関数内でもWeb Workerを動的インポートするようにしています。
これは、`$props`で渡された言語のWeb Workerのみをインポートするようにしているため、効率的にWeb Workerを取得することができるようになっています。

`getWorker`の定義後、`m.editor.create`関数で、monaco editorをインスタンス化し、3で定義した`editor`変数に代入しています。

また、`onDidChangeModelContext`関数に、エディタで更新されたテキスト内容をbindingされた`value`に代入するリスナーを登録しています。
これによって、エディタの最新のテキスト内容を親コンポーネントに伝えることができるようになっています。

### 5. Monaco Editor初期化関数を実行する`onMount`

先ほど定義した初期化関数を`onMount`で実行します。
また、戻り値でMonaco Editorをdisposeする関数を返すことで、このコンポーネントが使用されなくなったタイミングでMonaco Editorもクリーンアップされるようになります。

### 6. プロパティ変更時に実行されるMonaco Editorの状態を変えるスクリプト

これまでの処理でMonaco Editorは初期化されますが、プロパティの変更があったとしてもMonaco Editorにはその変更が伝わらないため、プロパティの変更に応じてMonaco Editorの状態を変える関数を実装する必要があります。

このコンポーネントでは、`$effect`を使用して`language`と`theme`変数が変わった際にMonaco Editorの設定も変えるようにしています。
各関数の最初の行で`$:`と書いていますが、この行がないとプロパティが変化しても関数が実行されなかったので注意が必要です。（値に依存している関数だとコンパイラに判定されない？）

https://github.com/dorayaki4369/monaco-editor-with-sveltekit/blob/ff3ce2ff79341689e38ee1b9ab1f6597bdd05f80/src/lib/components/Editor.svelte#L21-L45

## まとめ

本記事ではSvelteKitとMonaco Editorを組み合わせてみたところを記事にしました。
今回の実装だと対応言語は少ないですが、ブラウザ上で手軽にVS Codeと同じエディタが動くのはとても良いですね！

ゆくゆくはLSPサーバーに接続して対応言語を増やしてみたいと思います。