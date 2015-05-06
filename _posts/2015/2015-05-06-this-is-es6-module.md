---
title: "ES6 moduleのtop levelにある`this`の値は何になるのか?"
author: azu
layout: post
date : 2015-05-06T18:41
category: JavaScript
tags:
    - ECMAScript
    - JavaScript
    - module

---

## 知りたい事

- ES6 moduleのtop levelにある`this`の値は何になるのか?
- [Babelで top level this が undefinedになって困った件 - console.lealog();](http://lealog.hateblo.jp/entry/2015/04/27/203147 "Babelで top level this が undefinedになって困った件 - console.lealog();")
     - [demo](https://babeljs.io/repl/#?experimental=true&evaluate=true&loose=false&spec=false&playground=false&code=var%20that%20%3D%20this%3B)

 
----

## Babel 前提

- Babelは入力されたコードをES6 moduleとして扱う

> Babel assumes that all input code is an ES6 module

- [FAQ · Babel](https://babeljs.io/docs/faq/#why-is-this-being-remapped-to-undefined- "FAQ · Babel")

(結論的には`--blacklist strict`でこの挙動は無効化できる)

## ES6 moduleの前提知識

- Moduleは[Module Environment Records](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-module-environment-records "Module Environment Records")という所に情報を記録する(スコープみたいなものに必要な環境情報的なものがある)
- [Module Environment Records](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-module-environment-records "Module Environment Records")はdeclarative Environment Recordの一種で、加えてModule専用の記録領域がある(importのbindingのための場所)
- 基本的な仕組みはdeclarative Environment Recordと同じ

(Recordの所は実装してみないとイマイチピンと来ないので大分ぼやかしてます。
Javaの実装見るとModuleEnvironmentRecordはDeclarativeEnvironmentRecordを継承するみたいなイメージでも良さそう)

- [es6draft/ModuleEnvironmentRecord.java at master · anba/es6draft](https://github.com/anba/es6draft/blob/master/src/main/java/com/github/anba/es6draft/runtime/ModuleEnvironmentRecord.java "es6draft/ModuleEnvironmentRecord.java at master · anba/es6draft")

## ES6 `this`の前提知識

`this`という値がどう解決されるか

- [8.3.3 ResolveThisBinding ( )](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-resolvethisbinding "8.3.3 ResolveThisBinding ( )")
- [8.3.2 GetThisEnvironment ( )](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-getthisenvironment "8.3.2 GetThisEnvironment ( )")

のアルゴリズムによって決定される。

2つをまぜると以下のような感じの構造

1. Let envRec be GetThisEnvironment( ).
    1. Let lex be the running execution context’s LexicalEnvironment. // <= 今いるEnvironment(スコープみたいなもの)と取り出す
    2. Repeat
        1. Let envRec be lex’s EnvironmentRecord.
        2. Let exists be envRec.HasThisBinding().// <= そのEnvironment RecordsのHasThisBinding()を見る
        3. If exists is true, return envRec. // <= trueを返してきたら＊へ
        4. Let outer be the value of lex’s outer environment reference.
        5. Let lex be outer.(Repeatへ戻る)
2. Return envRec.GetThisBinding(). // ＊<= そのEnvironment RecordsのGetThisBinding()が`this`の値になる

擬似コード

```js
function GetThisEnvironment(){
    var lex = currentLexicalEnvironment;
    while(true){
	    var envRec = lex.EnvironmentRecord;
	    if(envRec.HasThisBinding()){
	        return envRec;
	    }
	    lex = envRec.outer;
	    // 最後はglobalがlexに入る
    }
}
var envRec = GetThisEnvironment();
envRec.GetThisBinding(); // thisの値
```

簡単にいえば、そのスコープが`HasThisBinding()`でtrueを返すなら`GetThisBinding();`の返り値が`this`の値になる。

- [es6draft/ExecutionContext.java at 03cc13e31f686584072bcf8ca1a7cebb88edda37 · anba/es6draft](https://github.com/anba/es6draft/blob/03cc13e31f686584072bcf8ca1a7cebb88edda37/src/main/java/com/github/anba/es6draft/runtime/ExecutionContext.java#L491 "es6draft/ExecutionContext.java at 03cc13e31f686584072bcf8ca1a7cebb88edda37 · anba/es6draft")

----

## Moduleのthis

Moduleはどこで作られるかというのは[8.1.2.6 NewModuleEnvironment (E)](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-newmoduleenvironment "8.1.2.6 NewModuleEnvironment (E)")で定義されている。

この内部メソッドを呼び出してるのは、[15.2.1.16.4 ModuleDeclarationInstantiation( ) Concrete Method](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-moduledeclarationinstantiation "15.2.1.16.4 ModuleDeclarationInstantiation( ) Concrete Method")しかないので、

```
NewModuleEnvironment(realm.[[globalEnv]])
```


先ほどのmodule Environment Recordの`outer`にはglobal enviromentが入る。(prototypeチェーンみたいな話)

## ModuleのGetThisEnvironment

先ほど書いたように、`this`の値は、そのEnvironment(スコープ)の`env.GetThisBinding()`と`env.HasThisBinding()`で決まる。

moduleのtop levelのコードから直近のEnvironmentはmodule Environmentなので、module Environmentの定義をみる。

`moduleEnvRecord.HasThisBinding()`がtrueを返すなら、`moduleEnvRecord.GetThisBinding()`が`this`の値になると言える。

8.1.1.5 Module Environment Recordsで

- [8.1.1.5.3 HasThisBinding ()](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-module-environment-records-hasthisbinding "8.1.1.5.3 HasThisBinding ()")
- [8.1.1.5.4 GetThisBinding ()](http://people.mozilla.org/~jorendorff/es6-draft.html#sec-module-environment-records-getthisbinding "8.1.1.5.4 GetThisBinding ()")

に書いてあるように、moduleEnvRecordの`HasThisBinding()`はtrueを返す。
つまり、moduleEnvRecordの`GetThisBinding()`が`this`の値になるので、moduleの`this`はundefeinedになる。
という結論

	8.1.1.5.3 HasThisBinding ()

	Module Environment Records provide a this binding.

	    Return true.

	8.1.1.5.4 GetThisBinding ()

	    Return undefined.


----

## 結論

BabelがコードをES6 modulesとして扱うなら、top levelの`this`がundefinedになるのは仕様。

Babelにissueを立てるならば、なぜ全てをES6 modulesとして扱うのか?が焦点。

また、この挙動は`--blacklist strict`で無効化するオプションが用意されている。

```
$ babel --blacklist strict script.js
```

- [Why is this being remapped to undefined?](https://babeljs.io/docs/faq/#why-is-this-being-remapped-to-undefined- "Why is this being remapped to undefined?")

であってると思うのだけれど、間違っているなら修正したいので[Issues](https://github.com/efcl/efcl.github.io/issues "Issues · efcl/efcl.github.io")か[Pull Requests](https://github.com/efcl/efcl.github.io/pulls "Pull Requests · efcl/efcl.github.io")お願いします。