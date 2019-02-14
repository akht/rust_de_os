# Writing an OS in Rust

[Writing an OS in Rust (Second Edition)](https://os.phil-opp.com/)

## A Freestanding Rust Binary
最初の一歩は、標準ライブラリを使わない実行可能バイナリを作ること。


### Introduction
OSのカーネルを書くためには、どのOSの機能にも依存しないコードを書かないといけない。
つまり
- スレッド
- ファイル
- ヒープメモリ
- ネットワーキング
- 乱数
- 標準出力
- その他もろもろ

が使えないということ。

Rustの標準ライブラリも使えない。
ただし
- イテレータ
- クロージャ
- パターンマッチ
- Option
- Result
- 文字列フォーマット
- 所有権システム

なんかは使える。

RustでOSのカーネルを作るためには、OSに依存せずに実行できるものを作らないといけない。

"**freestanding**"とか"**bare-metal**"なんて呼ばれているらしい。
それを作っていく。


### Disabling the Standard Library
ふつう、Rustのクレートは標準ライブラリとリンクしていて、

スレッドやファイルやネットワーキングなどの機能はOSに依存している。

また`libc`(Cの標準ライブラリ)にも依存していて、これがOSと緊密に連携している。

今回の目標はOSを書いてみることなので、あらゆるOS依存のライブラリは使わない。

そこで自動で標準ライブラリがインクルードされるのを防ぐために`no_std`アトリビュートを使う。

#### The `no_std` Attribute
cargoで新しいプロジェクトを作り、他はいじらずに`main.rs`の先頭に`#![no_std]`をつける。

これで`cargo build`してみると、`println!`マクロがないと怒られる。

`println!`は、標準出力(OSが提供する特別なファイルディスクリプタ)に書き込むために

標準ライブラリに用意されているマクロなので`no_std`だと使えない。

なので`println!("Hello, world!")`を取り除いて空の`main`関数にする。

`cargo build`すると、
```
> cargo build
error: `#[panic_handler]` function required, but not found
error: language item required, but not found: `eh_personality`
```
みたいなエラーがでる。

`#[panic_handler]`の関数と、`eh_personality`とかいう言語アイテムがないらしい。

この２つのエラーを解決する。

#### Panic Implementation
`panic_handler`アトリビュートは、panicが起こったときに呼ばれる関数を定義するやつ。

`no_std`だと自分で用意する必要がある。

書かれているとおりに`panic`関数を定義する。

`!`型のことを`never type`といって、それを返す関数を`diverging function(発散する関数)`というらしい。

#### The `eh_personality` Language Item
`Language items(言語アイテム)`というのはコンパイラ向けの特別な関数や型のこと。たとえばCopyトレイトがそう。

ここでは`eh_personality`言語アイテムが必要なんだけど、今回は自分で実装しない。
(型チェックが効かなかったり、実装がunstableで、要するに煩雑なのかな)

代わりの方法で回避する。

この`eh_personality `言語アイテムは、panic時の[スタックのアンワインド](https://www.bogotobogo.com/cplusplus/stackunwinding.php)に関わるものらしい。

いろいろ込み入っていて、OS specificなライブラリを使っていたりもするので今回は踏み込まない。ただ単にアンワインドを無効にする。

#### Disabling Unwinding
RustにはpanicをabortするオプションがあるのでCargo.tomlに設定する。

これで`eh_personality`がなくても大丈夫になる。

`cargo build`してみると今度は
```
> cargo build
error: requires `start` lang_item
```
というエラーがでる。


#### The `start` attribute
プログラムを走らせたときいきなり`main`関数が呼ばれると思っている人がいるが実際はそうじゃなく、ほとんどの言語にはランタイムがあって(たとえばJavaのGCだったり、Goのgoroutineだったりを提供する)、それらは`main`関数が呼ばれる前に初期化されている必要がある。

ふつうのRustバイナリの場合は、まず`crt0`(c runtime zero)というCのランタイムライブラリ(スタートアップルーチン)が呼ばれ、こいつがスタックを用意してくれたりレジスタをあれこれやってくれるらしい。(※よくわかってない)

それからCのランタイムがRustランタイムのエントリーポイントを呼び出し(これが`start`言語アイテム)、そこから`main`関数が呼ばれる。

今回はRustランタイムと`crt0`が使えない。`start`言語アイテムを自分で作ることもできない(`crt0`が必要なため)

なので`crt0`エントリーポイントをオーバーライドしてしまう。


#### Overwriting the Entry Point
ふつうのエントリーポイントを使わないことをRustコンパイラに教えるために`#![no_main]`アトリビュートをつける。そして`main`関数を削除する。

`main`関数は、それを呼び出すランタイムが存在しないと意味をなさないので不要になる。

代わりに、OSのエントリーポイントをオーバーライドする。

エントリーポイントの名前はOSごとに慣習があるが、今回はLinuxのデフォルトエントリーポイントに使われている`_start`という名前で関数を作ってオーバーライドする。

[名前マングリング(名前修飾)](https://ja.wikipedia.org/wiki/%E5%90%8D%E5%89%8D%E4%BF%AE%E9%A3%BE)をさせないために`#![no_mangle]`をつける。

`no_mangle`をつけないと`_start`がマングリングされて`_ZN3blog_os4_start7hb173fedf945531caE`みたいなシンボルになってしまい、リンカがエントリーポイントとして認識できなくなる。

`extern "C"`をつけて、Cの呼出規約を使うことをコンパイラに示す。

`_start`は値を返す必要がないためdiverging functionにする。エントリーポイントはOSやブートローダーから直接呼ばれるもので、関数から呼ばれることはないため値を返さない。値を返すかわりにOSの`exit`(システムコール)したりする。

今はとりあえず無限にループしとけばいい的なことが書かれている。
