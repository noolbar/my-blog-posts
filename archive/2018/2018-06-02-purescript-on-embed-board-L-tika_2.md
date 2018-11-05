---
date: 2018-06-02T16:30:00+09:00
title: PureScriptでLチカをしよう(2)
tags: purescript 電子工作
author: noolbar
---

## 前回のあらすじ
[PureScriptでLチカをしよう(1)](https://qiita.com/noolbar/items/0de7ea92200f74e7d984)でPureScriptをC++に変換して実行する事が出来ました.ここから評価ボードへの組込みに向けてコンパイルすることを目標にやっていきましょう.
関数型言語に馴染みのない開発者に向けてPureScriptの単純なコードを示しながら読み進められるように記載を心がけましたが,よくわからない部分やおかしい部分やがあれば,ご連絡いただけるとうれしいです.また,[実例によるPureScript](http://aratama.github.io/purescript/)を参考書として合わせて参照してみてください.

## 市場調査
こんな記事を見ている読者には関係がないかもしれませんが,新しい言語を始めるのは障害が多く難しいものです.特にプロジェクトに取り入れるのはどんな人であれ新しいことに付随する苦労に尻込みするものです.自分を含めそんな人達の意欲を刺激するため,組込みへのいろいろな言語の導入のトレンドを見てみましょう.

組込み目的で関数型言語を使うというプロジェクト[Functional IoT](http://fpiot.metasepi.org/)があり,有用な情報がまとめられています.

ここの著者は過去にHaskellを組込みやOSに使うため[Ajhc](http://ajhc.metasepi.org/)というプロジェクトを行っており,[Functional MCU programming #0: Development environment](https://www.slideshare.net/master_q/functional-mcu-programming-0-development-environment)にHaskellを組込みで使う上での特徴をまとめています.

[LDCでSTM32F103のバイナリを作ろうとしてできなかった](http://kubo39.hatenablog.com/entry/2017/03/23/LDC%E3%81%A7STM32F103%E3%81%AE%E3%83%90%E3%82%A4%E3%83%8A%E3%83%AA%E3%82%92%E4%BD%9C%E3%82%8D%E3%81%86%E3%81%A8%E3%81%97%E3%81%A6%E3%81%A7%E3%81%8D%E3%81%AA%E3%81%8B%E3%81%A3%E3%81%9F)ではSTM32F103をダーゲットにD言語で組み込みを行っています.トラブルをどのように解決していったのかを細かく記載してあり,クロスコンパイルでの問題解決に有用な情報が掲載されています.

上記以外にも組込み開発に書きやすい言語持ってくるのは,IPythonやmRubyなどをRasbPiでプログラムでしてみたなどの記事は大量にあり十分な要望があるとこであるとわかります.

組込みソフトウェアは個人又は数人で行う小規模なケースが多く,ソフトウェアを納入品としないなど客先から言語の指定がない場合には新しい技術の導入に必要な合意を取る人間の数も少なく済みます.多くの人が一丸となってプロジェクトに取り組む大規模ソフトウェア開発はソフトウェア開発の花形であるようなイメージがありますが,小規模で閉鎖的なプロジェクトも良い部分があります.

こういったことを知っておくと今回のような挑戦的な作業を進める上でモチベーションになります.

## はじめに
PureScriptを組込み目的に使うにあたってC++コードを出力させますが,前回触れたように`purescript-native`は試験的なプロジェクトでバーションの追従も完璧でなく,また,実行バイナリのサイズを小さくすることは,優先順位がかなり低いと考えられるため,サイズを小さく詰めていくことは難しいと思われます.
上記のFunctional IoTで紹介されているメールではHaskellを用いたOS開発に触れて以下のように

```
While jhc generates C code, the kind of C code it 
generates may not be suited for kernel.
```

jhc(Ajhcのフォーク元)はCコードを生成しますが、出力されたコードはカーネルに適していない可能性があると言っています.これはクロスコンパイル全体に言えるでしょう.曖昧な表現ですが,変換前と変換後が大きく異なる場合にはより顕著に起こる現象です.Haskellと文法の似ているPureScriptも同様の問題を持っています.今回は良い感じに変換されることを祈り,ダメな場合には手動で直す方針で行きます.

バイナリサイズに関しても同様に考え,ハードウェアはどんどん進化していること,最終的なコードが小さくなるにしろデバックのために大きなファイルサイズを一時的に動作させる必要があること,IoTは少量多品種なものが多くハードウェアの値段がクリティカルな問題ではないことから,この点には目をつぶり,現実的なファイルサイズ及びメモリサイズであれば良しとしましょう.

## 評価ボードの選定
組込み対象となる評価ボードについて[Functional IoT](https://www.slideshare.net/master_q/functional-mcu-programming-0-development-environment)の選定を見てみましょう.

- MCU: ARM Cortex-M 
  - mbed LPC1768 https://os.mbed.com/platforms/mbed-LPC1768/ (NXP Semiconductors)
  - STM32F4DISCOVERY http://www.st.com/en/evaluation-tools/stm32f4discovery.html (STMicroelectronics)
- MCU: TI MSP430
  - MSP-EXP430G2 http://www.ti.com/tool/msp-exp430g2
- MCU: Atmel AVR
  - Arduino Mega 2560 compat board http://www.amazon.co.jp/dp/B00CF2REXC 

選定しているものが少し古いですが,STMicroelectronics社のSTM32シリーズは入手性もよく値段も安いため,`STM32F4DISCOVERY`で行きましょう.ですが,今回はお金がないため評価ボードの購入を見送ることにしました.変なことにお金をかけている自分のせいですので,今回はシミュレータで我慢して進めることにします.`purescript-native`を使用したクロスコンパイルはC++に一度変換してからバイナリを作成するため,他の評価ボードに関しても今回と同様の手順で進めることができるはずです.

## Toolchainの選定
必要な情報について[ねむいさんのぶろぐ](http://nemuisan.blog.bai.ne.jp/?eid=188089)に詳しくまとまっています.
昔は`YAGARTO`でしたが,現在は[GNU Tools for ARM Embedded Processors](https://developer.arm.com/open-source/gnu-toolchain/gnu-rm)をおすすめしているため,コチラを使用していきます.

```bash
$ choco install gcc-arm-embedded 
```

msys2環境から使用するため,ビルドツールのあるフォルダにパスを通しておきましょう.

```bash
> arm-none-eabi-gcc --version | head -n1
arm-none-eabi-gcc.exe (GNU Tools for Arm Embedded Processors 7-2017-q4-major)
 7.2.1 20170904 (release) [ARM/embedded-7-branch revision 255204]

> arm-none-eabi-ld --version | head -n1
GNU ld (GNU Tools for Arm Embedded Processors 7-2017-q4-major) 2.29.51.20171128

> arm-none-eabi-gdb --version | head -n1
GNU gdb (GNU Tools for Arm Embedded Processors 7-2017-q4-major) 8.0.50.20171128-git

> arm-none-eabi-as --version | head -n1
GNU assembler (GNU Tools for Arm Embedded Processors 7-2017-q4-major) 2.29.51.20171128

> git --version
git version 2.17.0

```

今回記載するコードについては,以下に保存しておきます.STMicroelectronics社のコードについては別途保存してください.

https://github.com/noolbar/purescript-native-demo-ltika.git 

## Lチカコード
上記したようにC言語やC++言語以外でのLチカ実行はいろいろな記事があり,[ARM Cortex-M 32ビットマイコンでベアメタル "Safe" Rust](https://qiita.com/tatsuya6502/items/7d8aaf3792bdb5b66f93)などが参考になります.

全体のプログラムの流れは以下のようになっています.
+ ブート処理
+ ペリフェラルの初期化
+ IOポートのOn/Offを繰り返す

PureScriptは,I/Oポートの状態を示すメモリへ書き込むコードを記載することが出来ません.また,ビット演算を行う演算子も準備されていません.ここでは,関数型言語らしいコードにこだわらず,動作させることを優先してPureScriptから外部に記載したC++コードを呼び出して解決することにしましょう.

ブート処理については[スタートアップスクリプトを見てみる](https://yuki-sato.com/wordpress/2016/01/23/%e3%82%b9%e3%82%bf%e3%83%bc%e3%83%88%e3%82%a2%e3%83%83%e3%83%97%e3%82%b9%e3%82%af%e3%83%aa%e3%83%97%e3%83%88%e3%82%92%e8%a6%8b%e3%81%a6%e3%81%bf%e3%82%8b/)が参考になります.

今回はSTMicroelectronics社の[`STM32CubeF4`](http://www.st.com/ja/embedded-software/stm32cubef4.html)にある`LL template`をそのまま使えばよいでしょう.

ペリフェラルの初期化及びIOの操作を呼び出しているC++言語で書かれた以下のプログラムをPureScriptに書き換えてうごかすことにしましょう.ポートDの15をHighにして青のLEDを点滅させています.

```cpp
int main(void)
{
  RCC->AHB1ENR |= RCC_AHB1ENR_GPIODEN;
  
  GPIOD->MODER = 0x40000000;

  volatile unsigned int count = 0;
  while(1)){
    GPIOD->ODR = 1 << 15;
    for (count = 0; count <= 0x0A037A00;) {count++;}
    GPIOD->ODR = 0;
    for (count = 0; count <= 0x0A037A00;) {count++;}
  }
}
```

<!-- ```cpp
include <stm32f407xx.h>

int main(void)
{
  RCC->AHB1ENR |= RCC_AHB1ENR_GPIODEN;
  
  while(1)
  {
    HAL_GPIO_WritePin(GPIOD, GPIO_PIN_15, 1);
    HAL_Delay(1000);
    HAL_GPIO_WritePin(GPIOD, GPIO_PIN_15, 0);
    HAL_Delay(1000);
  }
}
``` -->

## まずは動かしてみる

1. 作成済みのファイルをGitHubからコピーしてきます.

    ```bash
    > git clone https://github.com/noolbar/purescript-native-demo-ltika.git workDir
    ```

1. 上記で紹介したSTM32CubeF4をダウンロードすると`en.stm32cubef4.zip`が入手できますので,必要なファイルを`STM32Cube_FW_F4_V1.21.0`から探して展開します.`LL template`で使用している以下のヘッダーファイルを`bootfile`に保存しましょう.

    `STM32Cube_FW_F4_V1.21.0\Drivers\CMSIS\Include`
    `STM32Cube_FW_F4_V1.21.0\Drivers\CMSIS\Device\ST\STM32F4xx\Include`
    `STM32Cube_FW_F4_V1.21.0\Drivers\STM32F4xx_HAL_Driver\Inc`

    メモリマップも後で使用するのでコピーしておきます.
    `STM32Cube_FW_F4_V1.21.0\Projects\STM32F4-Discovery\Templates_LL\SW4STM32\STM32F4-Discovery\STM32F407VGTx_FLASH.ld`

    `STM32Cube_FW_F4_V1.21.0\Projects\STM32F4-Discovery\Templates_LL\Src\`及び`STM32Cube_FW_F4_V1.21.0\Projects\STM32F4-Discovery\Templates_LL\Inc\`にあるファイル並びに`STM32Cube_FW_F4_V1.21.0\Projects\STM32F4-Discovery\Templates_LL\SW4STM32\startup_stm32f407xx.S`を`bootfile`に展開します.

    `bootfile`の構成に以下ファイルが追加されます.(main.cは不要です)

    ```code
    workDir/
    |-bootfile/
    |---startup_stm32f407xx.s
    |---main.h
    |---stm32_assert.h
    |---stm32f4xx_it.h
    |---stm32f4xx_it.c
    |---system_stm32f4xx.c
    |---STM32F407VGTx_FLASH.ld
    |---STM32Cube_FW_F4_V1.21.0/
    ```

1. C++コードを生成します.`purescript-native`の作成は前回の記事を参考に行い,パスを通しておきましょう.

    ```bash
    > cd workDir
    > make clean
    > make codegen
    ```

    エラーなくコードの生成が出来たでしょうか.
    以降ではgit cloneで持ってきたコードの内容の解説を行っていきながら,Lチカを目指します.とりあえず実行したい場合は次章を飛ばしても問題ありません.

## 外部コードの呼び出し
`FFI`と呼ばれる外部関数インタフェース機能をつかって,PureScriptは他の言語で記載されたライブラリをコード内で使うことができます.

1. 実装となる`bootfile/MPU.cc`を作成します.これは,C++言語で記載します.外部関数の定義については,`purescript-native`が生成する外部関数を定義しているファイルが役に立ちます.例えば`.psc-package\master\console\master\src\Control\Monad\Eff\Console.purs`などを参考にしましょう.

    <!-- https://github.com/paf31/24-days-of-purescript-2016/blob/master/19.markdown
    https://github.com/pure11/pure11/wiki/Memory
    https://www.linkedin.com/pulse/purescript-effects-map-11-global-state-process-wide-coupling-yerkes
    http://andyarvanitis.com/an-ffi-example-for-purescript-c-plus-plus-pcre/
    https://github.com/purescript/documentation/blob/master/language/Differences-from-Haskell.md
    https://github.com/purescript/documentation/blob/master/guides/Eff.md
    http://james-iry.blogspot.jp/2009/07/void-vs-unit.html -->

    まずは,ピンをGPIOとして動作させる設定を行う`externfeSetPeripheral`という関数を定義します.以下のコードを実行させることで実現させます.

    ```cpp
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIODEN;
    ```

    設定値は引数として設定したかったので,以下のように作成しました.

    ```cpp:bootfile/MPU.cc
    #include "stm32f4xx_it.h"
    #include "PureScript/PureScript.hh"
    #include "Main/Main_ffi.hh"
    #include "Data.Unit/Unit.hh"

    namespace MPUFFI {
      using namespace PureScript;

      auto externfeSetPeripheral (const any &flag) -> any {
        return [=]() -> any {
          RCC -> AHB1ENR |= static_cast<const unsigned int>(flag);
          return Data_Unit::unit;
        };
      };
    }
    ```

    `bootfile/MPU.hh`でインクルードします.
    
    ```header:bootfile/MPU.hh
    #ifndef MPUFFI_HH
    #define MPUFFI_HH

    #include "PureScript/PureScript.hh"

    namespace MPUFFI {
      using namespace PureScript;

      auto externfeSetPeripheral (const any &flag) -> any;
    }
    #endif
    ```
      
    ```cpp:bootfile/Main_ffi.hh
    #include "MPU.hh"
    using namespace MPUFFI;
    ```

    ペリフェラルの初期化まで出来るように`src/Main.purs`を記載してみます.

    ```haskell:src/Main.purs
    module Main where
    import Prelude
    import Control.Monad.Eff ( Eff )

    foreign import data MEMORY :: !
    foreign import externfeSetPeripheral :: forall eff. Int -> Eff( memory :: MEMORY | eff ) Unit

    -- #define RCC_AHB1ENR_GPIODEN 0x00000008
    main :: forall eff a.  Eff( memory :: MEMORY | eff ) a
    main = do
      externfeSetPeripheral 0x00000008
    ```

    関数の定義にある.`Eff( memory :: MEMORY | eff ) Unit`の部分が見慣れないコードかと思います.これは,副作用の表現を行っています. 関数の型は「副作用のある計算で、メモリ操作とそれ以外の任意の種類の副作用を備えた任意の環境で実行することができ、型 Unitの値を返す」を表しています.この作用の影響はハンドラを使用して除去するまで伝搬していきます.


1. 同様にしてピンの出力を制御する`externHAL_GPIO_WritePin`や次のステップを遅らせる`externHAL_Delay`についてもPureScriptのコードに持ち込めそうです.
    他の部分で問題になりそうな部分として無限ループがあります.実行を続けるために必要な部分ですが,PureScriptに`while`や`for`などの予約後はありません.これをはどの様にコードを書けばればよいのでしょうか.今回は,無限ループの部分を関数にして自身を呼び出し再帰関数として記述します.

    ```haskell:src/Main.purs
    -- #define RCC_AHB1ENR_GPIODEN 0x00000008
    module main where
    import Prelude
    
    main = do
      unsafeSetPeripheral 0x00000008
      infLoop
      where
        infLoop = do
          infLoop
    ```

1. 他の関数を作成して呼び出しを加えた結果は以下のようになりました.手続き的に記載しているので抵抗感は少ないのではないのでしょうか.ペリフェラルの操作に必要なメモリアドレスは直接記載しています.

    ```haskell:src/Main.purs
    module Main where
    import Prelude

    import Control.Monad.Eff ( Eff )
    foreign import data MEMORY :: !

    foreign import externSystemClock_Config :: forall eff. Eff ( memory :: MEMORY | eff ) Unit
    foreign import externfeSetPeripheral :: forall eff. Int -> Eff( memory :: MEMORY | eff ) Unit
    foreign import externHAL_GPIO_WritePin :: forall eff. Int -> Int -> Int -> Eff( memory :: MEMORY | eff ) Unit
    foreign import externHAL_Delay :: forall eff. Int -> Eff( memory :: MEMORY | eff ) Unit

    -- #define RCC_AHB1ENR_GPIODEN 0x00000008
    -- #define PERIPH_BASE         0x40000000U
    -- #define AHB1PERIPH_BASE     (PERIPH_BASE + 0x00020000U)
    -- #define GPIOD_BASE          (AHB1PERIPH_BASE + 0x0C00U)
    -- #define GPIOD               ((GPIO_TypeDef *) GPIOD_BASE)
    -- #define GPIO_PIN_15         ((uint16_t)0x8000)
    main :: forall eff a.  Eff( memory :: MEMORY | eff ) a
    main = do
      externSystemClock_Config
      externfeSetPeripheral 0x00000008
      infLoop
      where
        infLoop = do
          externHAL_GPIO_WritePin 0x40000000 0x8000 1
          externHAL_Delay 0x05037A00
          externHAL_GPIO_WritePin 0x40000000 0x8000 0
          externHAL_Delay 0x0A037A00
          infLoop
    ```

1. 上記ファイルを追加するようにMakefileを編集します.

    ```makefile
    BOOTFILEiNCLUDE := -I bootfile -I bootfile/STM32Cube_FW_F4_V1.21.0/Drivers/CMSIS/Device/ST/STM32F4xx/Include -I bootfile/STM32Cube_FW_F4_V1.21.0/Drivers/CMSIS/Include -I bootfile/STM32Cube_FW_F4_V1.21.0/Drivers/STM32F4xx_HAL_Driver/Inc
    BOOTFILESRCS :=  $(call rwildcard,bootfile/,*.c) 
    OBJS = $(BOOTFILESRCS:.c=.o) bootfile/startup_stm32f407xx.o
    DEPS = $(SRCS:.cc=.d) $(BOOTFILESRCS:.c=.d)

    %.o: %.cc
      @echo "Creating" $@
      @$(CXX) $(CXXFLAGS) $(INCLUDES) $(BOOTFILEINCLUDE) -MMD -MP -c $< -o $@

    %.o: %.c
      @echo "Creating" $@
      @$(CXX) $(CFLAGS) $(BOOTFILEINCLUDE) -MMD -MP -c $< -o $@

    %.o: %.s
      @echo "Creating" $@
      @$(CXX) -c $< -o $@
    ```

## 実行ファイルを作ってみよう

1. クロスコンパイラを使って生成したコードから実行ファイルを作成してみましょう.実行時に組込み先を指定しないとエラーを起こしますので,これを指定します.

    ```bash
    > make release SHELL='sh -x' GC=NO CXX=arm-none-eabi-gcc CXXFLAGS=-DSTM32F407xx CFLAGS=-DSTM32F407xx
    ``` 

1. 無事コンパイルできて,リンカが実行されると`__sync_synchronize`がないと言われるため,`sync_synchronize.c`というファイルを作成して,いろいろ参考にしてごまかすコードを記載します.git repositoryからcloneした場合には,すでにファイルがあります.

    <!-- TODO メモリオーダリング
    https://stackoverflow.com/questions/14950614/working-of-asm-volatile-memory
    https://gcc.gnu.org/ml/gcc/2012-01/msg00372.html
    https://lists.rtems.org/pipermail/vc/2015-September/009073.html -->

    ```c:bootfile/sync_synchronize.c
    inline void __sync_synchronize()
    {
        asm volatile("" ::: "memory");
    }
    ```

1. いろいろなメモリマップわからないと言われてしまいます.Makefileに以下を追加します.

    ```code
    override LDFLAGS  += -Tbootfile/STM32F407VGTx_FLASH.ld `override LDFLAGS  += -lstdc++ -static`
    ```

    エラーが発生せずに`/output/bin/main`が作成されていれば成功です.

    まだエラーが続く場合の解決のTipsを記載しておきます.実装する必要のある関数が抜けているとき,nmコマンドでシンボルテーブルを確認しながら必要なファイルを加えていきます.

    ```bash
   > arm-none-eabi-nm --demangle --numeric-sort output/bin/main 
    ```

    エラーが出力されないが,動作しない場合にはgccのライブラリを検索するパスの一覧を`-l`で確認しましょう.確認してみると変なファイルが読み込まれている場合がありますので,その場合は修正しておきます.

    ```bash
    > arm-none-eabi-gcc --print-search-dirs
    ```

1. 生成できた実行ファイルを見てみましょう

    ```bash
    > arm-none-eabi-size output/bin/main
      text    data     bss     dec     hex filename
    296160    2784    7600  306544   4ad70 output/bin/main
    ```

    この実行ファイルのサイズであれば,書き込みを行うことができそうです.


## Lチカまでのながれ
PureScriptのコードを実行ファイルまでビルドすることが出来ました.これを書き込んで動作するか見てみたいと思います.上記したように評価ボードではなく,今回はシミュレータで我慢します.今後の開発を考えると,シミュレータでの動作確認は有用ですので良しとしましょう.

1. qemuを使うのがデファクトスタンダードでしょう.https://tnishinaga.hatenablog.com/entry/2016/12/31/130000 でLチカをシミュレータ環境で実行する記事があるのでやってみましょう.http://blog.boochow.com/article/456638901.html も参考になります.

    ```bash
    > qemu-system-gnuarmeclipse.exe --version
    GNU ARM Eclipse 64-bits QEMU emulator version 2.8.0 (v2.8.0-646-g2c99a25-dirty)
    Copyright (c) 2003-2016 Fabrice Bellard and the QEMU Project developers
    ```

1. qemuでシミュレータ環境を確認しましょう.STM32F4-Discoveryがリストにあるのを確認します.

    ```bash
    > qemu-system-gnuarmeclipse.exe -board help

    Supported boards:
      Maple                LeafLab Arduino-style STM32 microcontroller board (r5)
      NUCLEO-F103RB        ST Nucleo Development Board for STM32 F1 series
      NUCLEO-F411RE        ST Nucleo Development Board for STM32 F4 series
      NetduinoGo           Netduino GoBus Development Board with STM32F4
      NetduinoPlus2        Netduino Development Board with STM32F4
      OLIMEXINO-STM32      Olimex Maple (Arduino-like) Development Board
      STM32-E407           Olimex Development Board for STM32F407ZGT6
      STM32-H103           Olimex Header Board for STM32F103RBT6
      STM32-P103           Olimex Prototype Board for STM32F103RBT6
      STM32-P107           Olimex Prototype Board for STM32F107VCT6
      STM32F0-Discovery    ST Discovery kit for STM32F051 line
      STM32F4-Discovery    ST Discovery kit for STM32F407/417 lines
      STM32F429I-Discovery ST Discovery kit for STM32F429/439 lines
      generic              Generic Cortex-M board; use -mcu to define the device

    ```

1. `qemu`で作成したプログラムを実行します.

    ```bash
    > qemu-system-gnuarmeclipse.exe --verbose --board STM32F4-Discovery --gdb tcp::3333 --semihosting-config enable=on,target=native --image ./output/bin/main
    ```

    コンデンサ横のLEDが点灯しているのが確認できれば成功です.

    ![Ltika](https://raw.githubusercontent.com/noolbar/purescript-native-demo-ltika/master/res/Ltika.gif)

    別なコンソールで`gdb`を起動すれば,デバッカを使った実行を追う作業も出来ます.

    ```bash
    > arm-none-eabi-gdb -q ./output/bin/main
    ```

    コマンドでプログラムを実行します.

    ```bash
    (gdb) target remote :3333
    (gdb) load ./output/bin/main
    (gdb) continue
    (gdb) monitor stop
    (gdb) quit
    ```

## 実行ファイルをターゲットCPUへの書き込み
今回はチャレンジできませんでしたが,機会があれば[ねむいさんのぶろぐ](http://nemuisan.blog.bai.ne.jp/?eid=188089)で紹介されている`OpenOCD`を使ってターゲットボードへの書き込みに挑戦してみます.

<!-- http://www.besttechnology.co.jp/modules/knowledge/?OpenOCD --> 

この記事の内容は1円もかからないので,実際にLチカさせてみてください.自分の手元で物が動いている様子を見ると感動するものがあります.
