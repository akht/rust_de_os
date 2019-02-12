# Writing an OS in Rust

[Writing an OS in Rust (Second Edition)](https://os.phil-opp.com/)

## 標準ライブラリを無効にする

`no_std`のアトリビュートをつけることで標準ライブラリにリンクさせなくする

そうすると`println!`マクロが使えなくなるので、ひとまず`main`関数の中身を空にする

今度は`#[panic_handler]`がないと怒られるので実装する

次に`eh_personality`というlang_itemがないと怒られる
これはスタックのアンワインドに関わるやつだがスタックのアンワインドは難しいから今回は無効にする
いろんな方法があるがCargo.tomlに書くのが簡単


すると`start`というlang_itemがないと怒られる

もろもろあるのは~~あとで~~いつか書く


コンパイル
```
$ cargo rustc -- -C link-arg=-lSystem  
```

ひとまずビルドできるようになった
