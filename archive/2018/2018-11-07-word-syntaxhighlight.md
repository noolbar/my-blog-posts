
---
date:2018-11-07T02:20:28.9140730+09:00
title:Microsoft Office Wordにシンタックスハイライトしたコード表示をさせたい
slide: false
---

## CUI環境をもっとみんなに使ってもらいたい。
Windowsユーザには馴染みのないコマンドコンソール画面ですが､同じことを何度もやらせる場合にはとても便利です。便利なものは積極的な利用を促進したい。

しかし、初めて利用する人にただ真っ黒な画面を出されても、どうしたら良いのかわからず立ち尽くしてしまいます。そんな人でもコピペするだけで実行できるようにサンプルを提示させたいのですが、細かい説明とともに取扱説明書のような形にまとめたらユーザフレンドリになるんじゃないかと考え、Wordの本文中にコードを表示ことを目指していきます｡

ただ貼り付けるだけではただの文字になるため､ハイライトのないコードは見ていて辛いものがあります。
最近のエディタにあるようなシンタックスハイライトを行って表示できるようにします。

今回のサンプルは以下に保存します｡

<https://github.com/noolbar/my-blog-ms-word-highlight.git>

# Why do we use XSL for?
どのようにWordの中に記述しようか考えながら、機能をざっと見ていたのですが､クイックパーツにあるフィールドのInsetTextにXSL変換のオプションがありました｡

XSLを用いることによってXML文書の表示スタイルを指定することができ、汎用的なプログラムを書いたりすることもできます｡

syntaxhighlightには以下のようなものがあります｡
<https://github.com/docbook/xslt10-stylesheets.git>

.netのxslプロセッサはxsl v1.0のみに対応であるため制限がありますが、docbookはv1.0に合わせて記載されています｡対応できそうです。しかし、これは実行環境に依存したスクリプトが必要で適用は無理そうです｡

しかも立て続けて問題があり、Wordに`Include`や`script`を含むxslを実行すると以下のようなエラーが表示されます｡

```
この文章に適用されているxsl変換(xslt)ファイルは、スクリプトまたは他の文書への参照が含まれていますが､署名されていません。
The xsl transformation (xslt) file that is applied to this document contains scripts or references to other documents. but it is not signed. word will apply the xslt file, but the scripts or references will not run
```

MSDNを見てみると.net環境でXSLの実行は`XslCompiledTransform`を推奨しており、Wordもこれを使っているような気がします｡
セキュリティに関する部分を拾い読みしていくと以下のようなものが見つかります｡

<https://docs.microsoft.com/ja-jp/dotnet/standard/data/xml/migrating-from-the-xsltransform-class>
既定で、XslCompiledTransform クラスでは XSLT document() 関数と埋め込みスクリプトのサポートが無効になっています。

<https://docs.microsoft.com/ja-jp/dotnet/api/system.xml.xmlresolver?view=netframework-4.7.2>
認証の資格情報を提供します。
読み取る XML データを格納しているファイルには、制限付きアクセス ポリシーがあります。 ネットワーク リソースにアクセスするために認証が必要な場合は、Credentials プロパティを使用して必要な資格情報を指定します。

セキュリティポリシーの問題で､Wordの実行環境は改善できそうにありません。

# powershellでの変換
逆に考えて指定してやればできそうなので、関数と埋め込みスクリプトに関してはPowerShellで実行できるように指定してやればできそうです。

<https://docs.microsoft.com/ja-jp/dotnet/standard/data/xml/script-blocks-using-msxsl-script>

以下のサイトのコードを試しにコピペして実行させてみました｡
<https://dobon.net/vb/dotnet/process/standardoutput.html>

```xsl
<?xml version="1.0" encoding="utf-8"?>
<xsl:stylesheet version="1.0"
  xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
  xmlns:msxsl="urn:schemas-microsoft-com:xslt"
  xmlns:user="urn:my-scripts">
  <msxsl:script language="CSharp" implements-prefix="user">
  <![CDATA[
public string Main(){
  //Processオブジェクトを作成
  System.Diagnostics.Process p = new System.Diagnostics.Process();

  //ComSpec(cmd.exe)のパスを取得して、FileNameプロパティに指定
  p.StartInfo.FileName = System.Environment.GetEnvironmentVariable("ComSpec");
  //出力を読み取れるようにする
  p.StartInfo.UseShellExecute = false;
  p.StartInfo.RedirectStandardOutput = true;
  p.StartInfo.RedirectStandardInput = false;
  //ウィンドウを表示しないようにする
  p.StartInfo.CreateNoWindow = true;
  //コマンドラインを指定（"/c"は実行後閉じるために必要）
  p.StartInfo.Arguments = @"/c dir c:\ /w";
  //起動
  p.Start();

  //出力を読み取る
  string results = p.StandardOutput.ReadToEnd();

  //プロセス終了まで待機する
  //WaitForExitはReadToEndの後である必要がある
  //(親プロセス、子プロセスでブロック防止のため)
  p.WaitForExit();
  p.Close();

  //出力された結果を表示
  return results;
}
  ]]>
  </msxsl:script>
  <xsl:output method="html" encoding="utf-8" indent="yes"/>
  <xsl:param name="l10n.gentext.default.language">ja</xsl:param>
  <xsl:template match="data">
      <xsl:value-of select="user:Main()"/>
  </xsl:template>
</xsl:stylesheet>
```

``` powershell
$xslsettings = new-object System.Xml.Xsl.XsltSettings($true, $true)
$resolver = new-object System.Xml.XmlUrlResolver
$xslt = new-object System.Xml.Xsl.XslCompiledTransform
$xslt.Load((resolve-path(".\res\test.xsl")), $xslsettings, $resolver)
$xslt.Transform((resolve-path(".\res\sample.xml")),($(pwd).path + "/sample.html"))
```

実行できました!!

このスクリプトブロックで実行できるのは.net環境で動くコードのみのようです。
keywordをハイライトさせるぐらいであれば、スクリプトに書けば良さそう。

C#で手頃なSyntaxhighlightがないようなので、JavaScriptの`Prism.js`を使ってみます｡
node環境があれば良いですが､インストールが許されない環境もありますので、ここではWindowsにデフォルトで実行することができるWSHを使いたいと思います｡
ただし、WSHのデフォルトの動作は、スクリプトエンジンが古いためそのままでは実行できません｡

探してみるとWSHの実行時にMicrosoft Edgeのエンジンを指定する方法がありました｡
`Prism.js`での実行結果をローカルファイルに書き出して扱えば、xslからでも行けそうです。

https://qiita.com/standard-software/items/82cfad075decd42aba92
https://qiita.com/segur/items/bcfeedf66a54372e67e6
http://inemaru.hatenablog.com/entry/2018/04/05/055557

こんな漢字でWSHスクリプトを指定してやれば、実行できます｡
``` 
%windir%\System32\cscript.exe //nologo //E:{1B7CD997-E5FF-4932-A7A6-2A9E636DA385} runWSH.jse.js
``` 

これに合わせて、WHSスクリプトを書きました｡このスクリプトは`Shift-JIS`で保存する必要があります｡`Prism.js` には手を加えずに動作させることができます｡`Prism.js`はこの使い方を予見していた?

さて、ここまでの一連の流れに沿って行えばで機能が完成です。

``` md
Wordからxsltを実行したいがセキュリティポリシーの問題でscriptや他の文書の読み込みできないので､PowerShellから`XslCompiledTransform`を使い`msxsl:script`を含めるxsltの実行できる環境を作る。
`msxsl:script`から`Prism.js`を呼び出したいが直接呼び出せないので､まずコマンドプロントから`cscript.exe`を使いWSHスクリプト実行し`Prism.js`を呼び出だす。結果を受け取りhtmlで出力する。
```

･･･PowerShellからxsltの実行がいらない!!

いらない部分を削ってまとめると**WHSスクリプトで`Prism.css`を適用する**でよさそうです。

WHSスクリプトで作成されるhtmlは以下のようになります。

``` html
<span class="token keyword">var</span> data <span class="token operator">=</span> <span class="token number">1</span><span class="token punctuation">;</span>
```

以下のようなコードでハイライトのためにhtmlヘッダと`Prism.css`を挿入します。

``` powershell
param($codefile)

if (-not(Test-Path $codefile)) {exit};

$basePath = (Resolve-Path($codefile + "\..\..\output\")).Path;
if (-not(Test-Path $basePath)) {mkdir -p $basePath}

$outputfile = ($basePath + (ls $codefile).BaseName + ".html");
$code = cat $codefile

cmd.exe /c "%windir%\System32\cscript.exe //nologo //E:{1B7CD997-E5FF-4932-A7A6-2A9E636DA385} runWSH.jse.js `"$code`" `"$outputfile`""

if (-not(Test-Path ($basePath + "prism.css"))){cp ./res/prism.css $basePath}
$htmlstring = @"
<html>
<head>
<link href="./prism.css" rel="stylesheet" />
</head>
<body>
$(cat $outputfile)
</body>
</html>
"@

$htmlstring > $outputfile
```

`Prism.js`のWebサイトからダウンロードしてきたcssを適用させておしまいと思いきや、標準で準備されている方法でWordにhtmlを挿入する方法がありません。

`Convert HTML to DOCX`や`Convert HTML to RTF`で検索してみるといろいろな方法が見つかります｡

+ HtmlをRichTextBoxで変換
+ pandoc
+ textutil

<!-- stylesheetを適用する方法
https://so-zou.jp/software/tech/programming/c-sharp/control/web-browser/styles.htm#create-style-sheet
https://teratail.com/questions/111095

Converting between RTF and HTML
https://code.msdn.microsoft.com/Converting-between-RTF-and-aaa02a6e

Displaying HTML in a WPF RichTextBox
https://www.codeproject.com/Articles/1097390/Displaying-HTML-in-a-WPF-RichTextBox

HtmlToXamlConverter
https://www.nuget.org/packages/HtmlToXamlConverter/1.0.5727.24510

chromium
http://cefsharp.github.io/ -->

例えば、以下のようなコードで変換できるとありますが､取得できるのは文字列だけで､ハイライトはできませんでした。

https://stackoverflow.com/questions/10410361/convert-html-to-rtf-rich-text-using-a-web-browser-control-in-asp-net
https://stackoverflow.com/questions/150208/how-do-i-convert-html-to-rtf-rich-text-in-net-without-paying-for-a-component

``` powershell
cd "D:\tmp\word-syntax-highlight"

Add-Type -Assembly System.Windows.Forms

$htmlstring = @"
<html>
<head>

</head>
<body>
<span class="token keyword">var</span> data <span class="token operator">=</span> <span class="token number">1</span><span class="token punctuation">;</span>
</body>
</html>
"@

$richTextBox = new-object System.Windows.Forms.RichTextBox;

$webBrowser = new-object System.Windows.Forms.WebBrowser;
$webBrowser.CreateControl();
$webBrowser.DocumentText = $htmlstring;
while ($webBrowser.DocumentText -ne $htmlstring){
   [System.Windows.Forms.Application]::DoEvents();
}
$webBrowser.Document.ExecCommand("SelectAll", $false, $null);
$webBrowser.Document.ExecCommand("Copy", $false, $null);
$richTextBox.Paste(); 
```

htmlからDocxまでスクリプトで一括変換して、完成品を出力できれば良いのですが､他のソフトウェアをインストールしたり､外部ライブラリを使用するのであればそもそも`Node.js`を使えばよいとなるため、レギュレーション違反(?)と思い今回は手動で対応します｡

htmlをWebブラウザに表示させてWordにコピペします。これを`rtf`や`docx`形式などで保存することで､ドキュメント内で利用することができるようになります｡どちらでもだいたい同じように使用することができます。

ちなみに、VSCodeがインストールされているのであればエディタ画面をコピーして、Wordに貼り付けるだけでそのまま画面がペーストされるので、今回のコードは**完全に不要**です。


## Wordからの使用
Word形式で保存したコードは、独立したファイルとして管理して、使用するファイルから利用してみます｡
コードを保守する側としてもWordファイルとコードが分離していることで、ドキュメントの管理が楽になります。
ここでは、Wordの標準の方法を使い外部のファイルを本文中に挿入します｡

同じフォルダに`child-document.docx`を配置してこれを`parent-docment.docx`から使用することを考えます｡

挿入->クイックパーツ->フィールドを選択でフィールドを挿入します｡(ショートカットはctrl+F9(windowsの場合)で`{}`が現れるので、ここにフィールド名を入力します｡)

![insert-quick-parts-field.png](https://raw.githubusercontent.com/noolbar/my-blog-ms-word-highlight/master/res/insert-quick-parts-field.png)

参照関係を作るには、以下の2つがよく利用されます｡

+ InsertText : テキスト、XML、Docxなどを外部から読み込んで挿入できます｡
+ Link : OLEでexcelファイルを読み込めます。Docxではブックマークを指定して読み込めます。

パスは絶対パスで書く必要があるので､相対パスで書きたい場合には以下のようにします｡
`{LINK  Word.Document.12 "{Filename \p}\\..\\child-document.docx"}`

ちなみに、XMLのタグの名称に`()`が入っていると全角でも半角でも失敗してしまいます。

##スタイルの適用
引用したあとにスタイルを適用することで､フォントや枠などを全体で統一させることができます。
今回のようなコードを表示させるには、段落の｢次の段落を改行しない｣｢段落を改行しない｣にチェックを入れて、等幅フォントになるようにしておけばよいでしょう。
![paragraph-setting.png](https://raw.githubusercontent.com/noolbar/my-blog-ms-word-highlight/master/res/paragraph-setting.png)

スタイルについて今まで使ったことがなければ、学会などで配布しているテンプレートファイルを使うと手間が少しだけ省けるので参考にして見るとよいかと思います。

## まとめ
この記事は最初xsltを使って__MS WordでVBAを使わずにフィボナッチ数列を計算してみた__みたいな題名で書き始めていたため､xsltの導入から書き始めたのですが､結局使わずに終わってしまいました。
Wordのバージョンによって細かい部分がことなっているものもあるかと思いますが､みなさんも試していろいろ遊んでみてください｡