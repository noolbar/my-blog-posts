---
date: 2018-05-03T16:39:06+09:00
title: PureScriptでLチカをしよう(1)
tags: purescript 電子工作
---

## PureScriptを組込みで使いたい
Haskellに影響を受けているPureScriptという言語をご存知でしょうか.純粋関数型言語で記述し,強力な型システムを持っており,Haskellと異なり正格評価で動作するAltJSです.

ここでは,何回かに分けてPureScriptを組込み目的で使用するための準備を行い,評価ボードでLチカをすることを目標とします.AltJSとして開発されたPureScriptは,JavaScriptに変換されて使用されることを想定していますので,まずはJavaScript以外の言語にちゃんと変換されることを確認していきます.

## ここがすばらしい
半導体の技術進歩によって開発者は,高機能なハードウェアを様々な場面に使えるようになり,組込みシステムで行うことができることは増えていく一方です.しかし,組込みシステムの開発は,目まぐるしく変化するweb開発などと比べて従来から変わらないレガシーな開発を行うケースがまだまだ多く,ソフトウェア開発部分だけを考えても開発環境の制限などから実績のある開発言語を用いてのが現状です.

異なる業界を比べることに違和感を感じる方もいらっしゃるかもしれません.リリース後のソフトウェアをの差し替えが比較的容易な環境と難しい環境では,後者のほうが十分なテストを行う必要性から開発が遅れてしまいます.
しかし,リリース後に改修を行う方法は必ずしも万能ではありません.Web開発で求められていることは組込みソフトウェアでも共通して求められていることであり,すべての開発者が本当に求めているのは,高品質なソフトウェアを効率よく作成する手法です.

PureScriptでは状態に影響を受けたり変更するコードと状態に関係なくいつ実行しても結果の変わらないコードを明確に分けています.この様な文法を取ることで見通しを良くしてテスト性を上げています.そんな言語が組込みシステムで使えるのであれば,皆さん使ってみたいと思わないでしょうか.

日本語での文法解説を行ったものは少ないのですが[実例によるPureScript](http://aratama.github.io/purescript/)が大変参考になります.

## どうやるの
PureScriptのJavaScript以外への変換について[24 Days of PureScript](https://github.com/paf31/24-days-of-purescript-2016)によれば,[C++](https://github.com/andyarvanitis/purescript-native)と[Erlang](https://github.com/purerl/purescript)への変換があるようです.組込み目的であれは,C++への変換ができればなんとかなりそうなので[`purescript-native`](https://github.com/andyarvanitis/purescript-native)プロジェクトを利用してボードへの組込みをやってみましょう.`README`から特徴を抜粋してみます.

- ランタイムシステムが不要
- ガベージコレクタ(GC)を使用するかどうかを選択できる
- GNU Makeを使って実行ファイルを作成
- 例外の使用について選択可能

## 使えるの
実用的なコードを記載していくのは難しいでしょう.現状ではPC上で実行されるネイティブコードを出力することすら試験的な状況です.ただ,組込みにいろいろな言語を取り入れたいと考える人はかなりいるようです.

[Haskellを組込みやOSに使うためのプロジェクト記事](http://metasepi.org/posts/2013-12-24-jats-ug.html)では[Hongwei Xi](http://www.cs.bu.edu/~hwxi/) (ATSの作者)からのメールを掲載しており,そこで以下のように語っています.

```
Features like GC could make the kernel highly 
unpredictable,scaring away potential users.
```

GCのような機能は,カーネルを非常に予測を難しくする可能性があり,ユーザはその様な不安定性を許容できないと語っています.GCに限らずコンパイル時に動作を予測できないコードを書きたくない開発者は一定数いて,RustなどもGCなしにヒープ領域を管理できるようです.許容できない機能が言語から排除できない場合には,その言語の採用を見送ることになります.

例外についても同様です.非同期処理を採用するケースや問題発生時に個別に復帰せず全体をリセットするケースなどソフトウェアを作成するにはいろいろな戦略があり,無駄のない機能選択ができる言語は理想的です.

PureScriptでは言語だけでなく,周辺のライブラリも小さく保つように設計されており,依存関係の見通しを良くしています.GCはデフォルトで未使用になっており,後で変換したコードで確認してみます.

## やったこと
ネイティブコードの出力について先駆者様が[詳細なノート]([https://github.com/andyarvanitis/purescript-native/issues/15])を残しています.以下の流れを見ながらWindows環境下でコンパイラを作成する流れを見てみましょう.

とりあえずPC上でコンパイラが動作し,コンソールへ"Hello World"を出力できるか確認します.

## 環境

```
os :Windows7 Home version6.1(build 7601 PS1)

> make --version
GNU Make 4.2.1
build for x86_64-pc-msys 
> gcc --version
gcc.exe (Rev2, Built by MSYS2 project) 7.3.0
> choco --version
0.10.10
```

## やってみよう

1. (事前準備)引用元では`mingw`を使用していますが,今回は`msys2`環境で進めました.ビルドツール`make`が使用できれば何でもよいでしょう.すべてのコマンドをシェル上でできればよいですが,私の環境では後述の`stack`の実行がうまくいきませんでした.
    ここではコマンドプロンプトで実行したコマンドは`$`を,シェルで実行したコマンドは`>`を頭につけて記述しておきます.

1. 上記引用先の2,3,4はそのまま言われたとおりに進めます.
    作業ディレクトリ `workDir`,`src`を作成し`src`内に以下の`Main.purs`を保存します.

    ```Haskell:Main.purs
    module Main where

    import Prelude
    import Control.Monad.Eff (Eff)
    import Control.Monad.Eff.Console (CONSOLE, logShow)

    main :: forall e. Eff (console :: CONSOLE | e) Unit
    main = do
      logShow "Hello World"
    ```

    Purescriptのコードについては,ここでは天下り的に受け入れて実行に進めます.ディレクトリ構成は以下のようになっているか確認しましょう.

    ```Bash
    workDir\
      src\
        Main.purs
    ```

1. Chocolatyを使って以下のコマンドで`stack`をインストールします.

    ```Bash
    $ choco install haskell-stack
    ```

    インストールのバーションは下記のとおりです.

    ```Bash
    $ stack --version
    Version 1.7.1, Git revision 681c800873816c022739ca7ed14755e85a579565 (5807 commits) x86_64 hpack-0.28.2
    ```

1. `purescript-native`を使ってPureScriptからC++へ変換するコンパイラ`pcc`を作成します.

    ```Bash
    $ cd ./workDir
    $ git clone https://github.com/andyarvanitis/purescript-native.git
    $ cd ./purescript-native
    $ stack setup
    $ stack build 
    ```

    成功していれば,それらしいところに`pcc.exe`ができているはずなので,パスを通しておきます.
    今回は`workDir/purescript-native/.stack-work/dist/2672c1f3/build/pcc/pcc.exe`にありました.

    ```Bash
    $ pcc --version
    0.10.7
    ```

1. `workDir`に戻り,先程作成したpccを引数無しで実行します.

    ```Bash
    $ cd ..
    $ pcc
    ```
    エラーメッセージが出ずに`Makefile`や`psc-package.json`などのファイルができていればOKです.

1. makeを実行し,実行ファイルを作成します.

    ```Bash
    > make SHELL='sh -x'
    ```

    `output\bin`に実行ファイル`main.exe`ができていれば成功です.

    ```Bash
    > output/bin/main
    "Hello World"
    ```

    うまくいかない場合には,作成されたMakefileを見てみましょう.Windows環境下で作成されたMakefileは頻繁におかしな部分ができるので問題があれば修正します.`make`を実行時に`SHELL='sh -x'`とすることで`make`時の詳細を出力できます.問題の解決に役立つでしょう.

## 見てみる
変換コードの確認でコンパイルされた実行ファイルは以下のようになっています.次回のLチカの参考にします.

```Bash
> size output/bin/main.exe
   text    data     bss     dec     hex filename
  39304    7204    5104   51612    c99c output/bin/main.exe
```

GCを無効にすることをシェル引数で明確にしておけば,Makefileの設定よりも優先して適用されます.自分の意図していない設定がMakefileに記載され,問題が起こることを防げます.

```Bash
> make SHELL='sh -x' GC=NO
```

`C++`に変換されたコードは`workDir\output\Main`に保存されます.
引数`-E`をに与えてマクロを展開したコードを確認しましょう.

```Bash
> g++ -E -O3 -flto -std=c++11 -Wno-logical-op-parentheses -Wno-missing-braces -Wmissing-field-initializers -I output -MMD -MP -c output/Main/Main.cc > Main.cpp
```

`Main.cpp`は以下のようなコードが出力されます.`INITIALIZE_GC();`が消えてGCが未使用になっているのがわかります.

```cpp:Main.cpp
auto main(int, char *[]) -> int {
  using namespace Main;
  ;
  Main::main();
  return 0;
};
```

## 次回
`purescript-native`は試験的なプロジェクトで,安定しているとは言えません.しかし,実際にC++に変換されたPureScriptのコードを得ることができました.
次のステップでは,PureScriptの文法を追いながら,組込みシステムで動作するプログラムを作成していきましょう.

