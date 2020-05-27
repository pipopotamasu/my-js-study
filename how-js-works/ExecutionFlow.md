# How JavaScript works
## 特記事項
![みつを](https://user-images.githubusercontent.com/14048211/82971219-21c81080-a00d-11ea-8697-10b56d53b19a.jpg)

## Motivation
もともとJSのパフォーマンス周りについて調べていたら、JSのネイティブコード関連の話が出てきた。
https://qiita.com/netebakari/items/7c1db0b0cea14a3d4419
<br>しかし、そういえばどういう風にJSが実行(=計算)される(機械語に翻訳されCPUが計算できるようになる)までをちゃんと知っていなかったので、これを機に勉強してみることにした。

## 補足
### JSエンジン
このドキュメントは「V8」というChromeやNode.jsで使われているJSエンジンを前提書かれてます。
JSエンジンとは、JavaScriptのソースコードをCPUが理解できるマシンコードに落とし込み、実行させるもののことです。
FirefoxのSpiderMonkey、SafariのJavaScriptCoreといったJSエンジンとは実行フロー、各部名称は異なります。

### 用語集
いろいろと難しい用語(村上基準)が出てくるので、事前に解説します。
結構抽象度が高い用語もあるので、「V8という文脈」ということを前提に解説してます(特にinterpreter周りとか各プログラミング言語のによって結構担当範囲が変わってくるっぽいので)。

#### (バイト)ストリーム
リソース(ファイル、メモリ、ネットワーク等)とプログラムでデータをやり取りする入出力(読み込み、書き込み、接続、切断)、またはそれを抽象化した各プログラミング言語におけるデータ型。
バイトストリームとはデータを処理する単位(つまり1バイトずつ)。
データはなんらかの文字コードでエンコードされており、UTF-8を例にあげると「f」は「0066」という数値にエンコードされている。

#### バイトストリームデコーダー
ストリーム処理において、エンコードされたデータをデコードする処理をするもの。

#### parser
与えられた(この場合JavaScriptの)テキスト情報(=コード)をASTに出力するもの。
パフォーマンスを向上させる目的(ex: webページの初回読み込み速度の向上)により、V8は２種類のparserを持っている。

**参考**
https://v8.dev/blog/preparser

##### 通常のparser
V8では、すぐに実行されるJavaScriptコードをparseするparser。
例えば、以下のような読み込まれたらすぐに実行されるコードはparserでparseされる。
```foo.js
console.log('foo')
```

後述するpre parserに渡すかどうかの判断もこのparserが担っている。

##### pre parser
すぐに実行されないコードをparseするparser。
例えばクリックイベントに紐づくコードなど。
```bar.js
// このbarの中身はすぐには実行されないので、pre parserでparseされる
function bar () {
  console.log('bar')
}

window.addEventLisner('click', bar);
```

#### AST
Abstract Syntax Tree(抽象構文木)の略。
解析されたプログラミング言語のコードをトークンに分解し、それを他のトークンとの関係性を元にtree構造に表現したもの。
図的にどんなものかは[ググって](https://www.google.com/search?q=abstract+syntax+tree&tbm=isch&ved=2ahUKEwjzwamb59LpAhVDEIgKHZd8AZQQ2-cCegQIABAA&oq=Abstract+syn&gs_lcp=CgNpbWcQARgAMgIIADIECAAQHjIECAAQHjIECAAQHjIECAAQHjIECAAQHjIECAAQHjIECAAQHjIECAAQHjIECAAQHjoECAAQBDoECAAQE1CsvhJYs9wSYMvhEmgBcAB4AIABYogBwAiSAQIxM5gBAKABAaoBC2d3cy13aXotaW1n&sclient=img&ei=27bNXrPoDMOgoASX-YWgCQ&bih=959&biw=1314&client=firefox-b)ください。

#### interpreter
ASTを元に、バイトコードを生成するもの。またV8ではバイトコードを元にマシンコードを生成する機能も担当する。
ちなみにV8のinterpreterは「iginition」と呼ばれている。

**参考**
https://stackoverflow.com/questions/54957946/what-does-v8s-ignition-really-do

#### マシンコード(machine code、機械語)
CPUが直接解釈・実行できる命令コードの体系。
いわゆる0と1の羅列。

#### ネイティブコード(native code)
CPUが理解できる形式(=マシンコード)で書かれたコンピュータプログラム。
C, C++, Rustとかがざっくりどいう風にマシンコード、ひいては実行ファイルを生み出しているかは以下をご覧ください。
http://nenya.cis.ibaraki.ac.jp/TIPS/compiler.html

#### バイトコード(byte code)
仮想的なコンピュータ(Virtual Machine)のために設計された命令コードの体系、また、そのようなコードによって記述された実行可能形式のプログラム。
仮想的なCPUが解釈、実行するマシンコードに相当する言語。

#### JIT compiler
JIT = just in time。
プログラムの実行時に逐次バイトコードからマシンコードを生成するコンパイラ。
V8のJIT compilerを「Turbo fun」と呼ぶ。

## JS Execution Flow
全体としはざっくりこんな感じです。
<img width="717" alt="Screen Shot 2020-05-27 at 11 22 45" src="https://user-images.githubusercontent.com/14048211/82970967-8df64480-a00c-11ea-8c73-05d1c8a91ef5.png">

以下で一つ一つ解説します。

### 1. Fetching source code
ブラウザの場合だと、HTMLパーサーが取得したHTMLを解析しスクリプトタグを見つけます。
その後、「ネットワーク」「ブラウザキャッシュ」「Service worker」のいずれかからJavaScriptソースコードを取得します。

> **どうソースコードを取得するのか？**<br>
> ネットワークを例にとると、ブラウザのプログラムとネットワーク間で「バイトストリーム」というデータ取得経路を作り、エンコード済みのデータ(=JavaScriptソースコード)を取得する。

その後、「バイトストリームデコーダー」によりデータをデコードする。
ある程度のまとまり(=トークン)までデータのデコードが完了したら、parserにトークンを渡す。

例えば
```example.js
function exmaple () {
  // 何かしらの処理
}
```
というJavaScriptのソースコードを読み込む場合、`function`がトークンの単位になる。

### 2. Parsing your code
parserは受け取ったトークンをを元にASTを構築する。
しかし、全てのトークンをASTにするわけではありません。パフォーマンスを向上させるため、「すぐに実行されるコード」のみ先にASTにし(通常のparserが行う)、「すぐに実行されないコード」は実行直前にparseを行います(pre parserが行う)。後者のことをlazy parsingと呼びます。

### 3. Generating Byte code
interpreterは構築されたASTを元にバイトコードを生成します。

### 4. Generating Machine code
さらにinterpreterはバイトコードを元にマシンコードを生成します。これでJavaScriptのコードがCPUで実行できるようになりました。

しかし、まだ終わりではありません。
ここからがV8の本領発揮です。

このバイトコードからマシンコードを生成する時、「Type Feedback」と呼ばれるメタデータも一緒に生成します。これは後述するコード最適化のための情報で、型情報や同じコードの実行回数などの情報を含むものです。

### 5. Optimization
バイトコードとType Feedbackの2つの情報を元に、V8のJIT compiler「Turbo Fan」がより最適化されたマシンコードを生成します。

なぜコードの最適化をするのかというと、一番の理由はJavaScriptは動的型付け言語だからです。
型が動的に変化するため、本来であれば毎回型のチェックをしなければならず、それがプログラムの実行を遅くします。しかしType Feedbackというメタデータを介して、何度も実行されるコードの型情報を取得することで、型チェックをスキップすることができ、より最適化されたマシンコードを生成できるのです。

以下のコードは最適化ができる例です。
`add`functionは数値を受け取り、数値を返すため、数値に最適化されたマシンコードを生成できます。
```optimization.js
function add (a, b) {
  return a + b;
}

add(1, 2);
add(2, 3);
```

### 6. Deoptimization
しかし、必ずしも最適化できるとは限りません。
以下のコードは引数に渡ってくる変数の型を毎回チェックしなければならないため、最適化できません。
```no-optimization.js
function add (a, b) {
  return a + b;
}

add(1, 2);
add('1', 2);
add('1', '2');
add(true, false);
```

この場合、Turbo fanはマシンコードをバイトコードに変換し直します。

## Conclusion
ざっくりこのようにJavaScriptは実行されます。
僕自身、V8のコードを直接読んでこのドキュメントを書いている訳じゃないので、大いに間違っている可能性もあるのであしからず。

# References
- https://dev.to/lydiahallie/javascript-visualized-the-javascript-engine-4cdf?signin=true
- https://www.youtube.com/watch?v=voDhHPNMEzg
- https://v8.dev/blog/preparser
- https://stackoverflow.com/questions/54957946/what-does-v8s-ignition-really-do
- https://v8.dev/blog/launching-ignition-and-turbofan
- https://blog.hiroppy.me/entry/2017/08/03/095304
- https://docs.google.com/presentation/d/1OqjVqRhtwlKeKfvMdX6HaCIu9wpZsrzqpIVIwQSuiXQ/edit#slide=id.g1357e6d1a4_0_58
