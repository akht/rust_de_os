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
