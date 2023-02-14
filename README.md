[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/saliton/RustUint40/blob/main/RustUint40.ipynb)

# Rustで整数を5バイトに押し込む簡単なお仕事

## はじめに

ビッグデータを処理するような場合、データの読み込みが律速になることが多いです。その場合、データのストレージ上のサイズが小さくなると読み込み時間が減り、全体の処理時間が半分になったりします。そんな時、整数が40bitで十分ならば、40bitで保持したいですね。というわけで、Rustで整数を40bitでストレージに読み書きする方法について調べました。[以前の記事](https://qiita.com/saliton/items/76781d1462bf04db7557)でGo言語で同じことをやっています。

> ※ インデックス算出の際に時間計測に含めるべきではない演算を行っていたので、それを排除した内容に差し替えました。　　

まずは、Rustをインストールします。


  ```shell
!wget https://static.rust-lang.org/rustup/rustup-init.sh
!sh rustup-init.sh -y
!cp /root/.cargo/bin/* /usr/local/bin
```

    --2022-01-22 16:44:50--  https://static.rust-lang.org/rustup/rustup-init.sh
    Resolving static.rust-lang.org (static.rust-lang.org)... 18.64.236.58, 18.64.236.29, 18.64.236.83, ...
    Connecting to static.rust-lang.org (static.rust-lang.org)|18.64.236.58|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 21102 (21K) [text/x-sh]
    Saving to: ‘rustup-init.sh’
    
    rustup-init.sh      100%[===================>]  20.61K  --.-KB/s    in 0.004s
    
    2022-01-22 16:44:50 (5.52 MB/s) - ‘rustup-init.sh’ saved [21102/21102]
    
    info: downloading installer
    info: profile set to 'default'
    info: default host triple is x86_64-unknown-linux-gnu
    info: syncing channel updates for 'stable-x86_64-unknown-linux-gnu'
    info: latest update on 2022-01-20, rust version 1.58.1 (db9d1b20b 2022-01-20)
    info: downloading component 'cargo'
    info: downloading component 'clippy'
    info: downloading component 'rust-docs'
    info: downloading component 'rust-std'
    info: downloading component 'rustc'
    info: downloading component 'rustfmt'
    info: installing component 'cargo'
    info: installing component 'clippy'
    info: installing component 'rust-docs'
     18.9 MiB /  18.9 MiB (100 %)   3.7 MiB/s in  5s ETA:  0s
    info: installing component 'rust-std'
     25.0 MiB /  25.0 MiB (100 %)   8.8 MiB/s in  3s ETA:  0s
    info: installing component 'rustc'
     53.2 MiB /  53.2 MiB (100 %)  10.3 MiB/s in  5s ETA:  0s
    info: installing component 'rustfmt'
    info: default toolchain set to 'stable-x86_64-unknown-linux-gnu'
    
      stable-x86_64-unknown-linux-gnu installed - rustc 1.58.1 (db9d1b20b 2022-01-20)
    
    
    Rust is installed now. Great!
    
    To get started you may need to restart your current shell.
    This would reload your PATH environment variable to include
    Cargo's bin directory ($HOME/.cargo/bin).
    
    To configure your current shell, run:
    source $HOME/.cargo/env


検証用のプロジェクトを作成して、カレントディレクトリを移動します。


  ```shell
!cargo new measure
%cd measure
```

    Created binary (application) `measure` package
    /content/measure


環境設定ファイルを生成します。


```Rust
%%writefile Cargo.toml

[package]
name = "measure"
version = "0.1.0"
authors = []
edition = "2018"

[dependencies]
rand = "0.8.4"
```

    Overwriting Cargo.toml


## 64bit整数の8バイト配列化

> `to_le_bytes()`がおすすめ

まずは64bitの整数をバイト配列にする方法を調べます。(以下では整数のエンコーディングはリトルエンディアンにします。特にunsafeがらみではビッグエンディアンのCPUでは間違った動作になりますのでご注意ください。)

愚直だとこうでしょうか。
```
for j in 0..8 {
    buf[j] = (v >> (8 * j)) as u8;
}
```


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);

    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8; 8]; SIZE] = [[0; 8]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let v = vs[idx];
            let b = &mut buf[idx];
            for j in 0..8 {
                b[j] = (v >> (8 * j)) as u8;
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..8 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.64s
    Running `target/release/measure`
    7.969 sec
    [205, 178, 63, 42, 187, 0, 0, 0]
    63749253


速い。Go言語では同じ処理で2分以上かかっていました。オプティマイザが優秀そうです。

ここで、参考のため、時間測定外の処理をなくして計測してみます。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8; 8]; SIZE] = [[0; 8]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let v = vs[idx];
            let b = &mut buf[idx];
            for j in 0..8 {
                b[j] = (v >> (8 * j)) as u8;
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.52s
    Running `target/release/measure`
    0.000 sec


計算結果を出力しないと、最適化で処理が削除されてしまうようです。また、計算結果の全てに触らないと部分的に削除されることもあるようなので、念のため全てに触るようにしてあります。


それでは内側のfor文を展開してみましょう。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8; 8]; SIZE] = [[0; 8]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let v = vs[idx];
            let b = &mut buf[idx];
            b[0] = v as u8;
            b[1] = (v >> 8) as u8;
            b[2] = (v >> 16) as u8;
            b[3] = (v >> 24) as u8;
            b[4] = (v >> 32) as u8;
            b[5] = (v >> 40) as u8;
            b[6] = (v >> 48) as u8;
            b[7] = (v >> 56) as u8;
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..8 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.56s
    Running `target/release/measure`
    7.861 sec
    [205, 178, 63, 42, 187, 0, 0, 0]
    63749253


ほぼ同じです。すでに最適化によりfor文が展開済みなのかもしれません。


Rustでは整数に対してto_le_bytes()というそのものズバリの関数が用意されていました。それを使用してみます。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8; 8]; SIZE] = [[0; 8]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            buf[idx] = vs[idx].to_le_bytes();
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..8 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.54s
    Running `target/release/measure`
    3.801 sec
    [205, 178, 63, 42, 187, 0, 0, 0]
    63749253


かなり速くなりました。

次に、整数をバイト配列に変換する別の方法が[某書籍](https://www.amazon.co.jp/dp/B087BZQ48R)で紹介されていたので、試してみます。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8; 8]; SIZE] = [[0; 8]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            unsafe {
                buf[idx] = std::mem::transmute::<u64, [u8; 8]>(vs[idx]);
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..8 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.57s
    Running `target/release/measure`
    3.957 sec
    [205, 178, 63, 42, 187, 0, 0, 0]
    63749253


処理時間、ほぼ同じです。

どうせunsafeを使うなら、最後に、ポインターを使う技を試してみましょう。当初、どう型を定義してよいかわからず、コンパイラーに随分怒られました。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let buf: [[u8; 8]; SIZE] = [[0; 8]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            unsafe {
                let ptr: *mut u64 = buf[idx].as_ptr() as *mut u64;
                *ptr = vs[idx];
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..8 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.55s
    Running `target/release/measure`
    3.915 sec
    [205, 178, 63, 42, 187, 0, 0, 0]
    63749253


誤差の範囲内で同じです。`to_le_bytes()`をおすすめします。

## 8バイト配列から64bit整数への変換

> `from_le_bytes()`がおすすめ



まずは愚直な方法です。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 8]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    // 計測開始
    let start = Instant::now();
    let mut v: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            for j in 0..8 {
                v += (b[j] as u64) << (8 * j);
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", v);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.60s
    Running `target/release/measure`
    11.566 sec
    8845746833903097824


配列への変換に比べるとかなり遅いです。分解するよりも合成する方が計算が必要な分、遅くなるのでしょうか。

内側のfor文を展開します。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 8]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    // 計測開始
    let start = Instant::now();
    let mut v: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            v += b[0] as u64;
            v += (b[1] as u64) << 8;
            v += (b[2] as u64) << 16;
            v += (b[3] as u64) << 24;
            v += (b[4] as u64) << 32;
            v += (b[5] as u64) << 40;
            v += (b[6] as u64) << 48;
            v += (b[7] as u64) << 56;
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", v);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.66s
    Running `target/release/measure`
    18.503 sec
    8845746833903097824


展開した方が処理時間が長くなる結果になりました。展開するよりも効率の良い最適化がなされるようになったのかもしれません。

次に、関数を使いましょう。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 8]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    // 計測開始
    let start = Instant::now();
    let mut v: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            v += u64::from_le_bytes(buf[idx]);
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", v);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.58s
    Running `target/release/measure`
    5.717 sec
    8845746833903097824


速くなりました。流石に専用の関数は違いますね。

別の関数による方法。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 8]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    // 計測開始
    let start = Instant::now();
    let mut v: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            unsafe {
                v += std::mem::transmute::<[u8; 8], u64>(buf[idx]);
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", v);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.59s
    Running `target/release/measure`
    5.691 sec
    8845746833903097824


同じです。

次はunsafeを使った方法。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 8]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    // 計測開始
    let start = Instant::now();
    let mut v: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            unsafe {
                v += *(buf[idx].as_ptr() as *mut u64);
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", v);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.59s
    Running `target/release/measure`
    5.787 sec
    8845746833903097824


特に速くはなりませんでした。`from_le_bytes()`をお勧めします。

## 40bit整数の5バイト配列化

> `*(buf[idx..].as_ptr() as *mut u64) = v`が最速

それでは、本題の40bit整数を扱う方法を検討しましょう。
まずは愚直な方法です。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);

    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8; 5]; SIZE] = [[0; 5]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let v = vs[idx];
            let b = &mut buf[idx];
            for j in 0..5 {
                b[j] = (v >> (8 * j)) as u8;
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.57s
    Running `target/release/measure`
    21.764 sec
    [205, 178, 63, 42, 187]
    63749253


驚くことに、内側のforループが8回から5回に減っているにもかかわらず、64bit整数の8バイト配列化よりも処理時間が長くなりました。どうやら8回限定の超絶最適化が存在するようです。

内側のforループを展開するとどうなるでしょうか。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }

    let mut buf: [[u8; 5]; SIZE] = [[0; 5]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let v = vs[idx];
            let b = &mut buf[idx];
            b[0] = v as u8;
            b[1] = (v >> 8) as u8;
            b[2] = (v >> 16) as u8;
            b[3] = (v >> 24) as u8;
            b[4] = (v >> 32) as u8;
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.54s
    Running `target/release/measure`
    21.827 sec
    [205, 178, 63, 42, 187]
    63749253


これは変わりませんでした。

次に関数利用です。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8;5]; SIZE] = [[0; 5]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let dst = &mut buf[idx];
            let src = vs[idx].to_le_bytes();
            for j in 0..5 {
                dst[j] = src[j];
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.59s
    Running `target/release/measure`
    10.318 sec
    [205, 178, 63, 42, 187]
    63749253


かなり速くなりました。

別の関数ではどうでしょうか。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8;5]; SIZE] = [[0; 5]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            unsafe {
                let dst = &mut buf[idx];
                let src = std::mem::transmute::<u64, [u8; 8]>(vs[idx]);
                for j in 0..5 {
                    dst[j] = src[j];
                }
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.54s
    Running `target/release/measure`
    10.411 sec
    [205, 178, 63, 42, 187]
    63749253


同じでした。

次はポインター利用です。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8;5]; SIZE] = [[0; 5]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let v = vs[idx];
            let b = &mut buf[idx];
            unsafe {
                *(b.as_ptr() as *mut u32) = v as u32;
                b[4] = (v >> 32) as u8;
            }
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.58s
    Running `target/release/measure`
    10.428 sec
    [205, 178, 63, 42, 187]
    63749253


同じです。

型変換に関数を使ってみます。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }
    let mut buf: [[u8; 5]; SIZE] = [[0; 5]; SIZE];

    // 計測開始
    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let v = vs[idx];
            let b = &mut buf[idx];
            unsafe {
                *(b.as_ptr() as *mut u32) = v as u32;
                b[4] = std::mem::transmute::<u64, [u8; 8]>(v)[4];
            }
        }
    }
    let end = start.elapsed();
    // 計測終了

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i][j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.62s
    Running `target/release/measure`
    10.443 sec
    [205, 178, 63, 42, 187]
    63749253


同じでした。

## 5バイト配列から40bit整数への変換

> `u64::from_le_bytes() & 0xFF_FFFF_FFFF`がおすすめ

まずは愚直な方法


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 5]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            let mut v = 0;
            for j in 0..5 {
                v += (b[j] as u64) << (8 * j);
            }
            total += v;
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.63s
    Running `target/release/measure`
    9.718 sec
    1666958173003018432


for文の展開


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 5]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            let mut v = b[0] as u64;
            v += (b[1] as u64) << 8;
            v += (b[2] as u64) << 16;
            v += (b[3] as u64) << 24;
            v += (b[4] as u64) << 32;
            total += v;
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.61s
    Running `target/release/measure`
    9.725 sec
    1666958173003018432


こちらはfor文を展開しても処理時間は変わりませんでした。

次に関数利用


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 8]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  0 as u8, 0 as u8, 0 as u8])
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            total += u64::from_le_bytes(buf[idx]) & 0xFF_FFFF_FFFF;
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.57s
    Running `target/release/measure`
    6.793 sec
    1666958173003018432


速くなりました。

40bitに切り詰めるのに別の方法を使ってみます。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 5]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = &buf[idx];
            total += u64::from_le_bytes([b[0], b[1], b[2], b[3], b[4], 0, 0, 0]);
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.58s
    Running `target/release/measure`
    10.015 sec
    1666958173003018432


遅くなりました。

さらに別の方法。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 5]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            total += u32::from_le_bytes([b[0], b[1], b[2], b[3]]) as u64 + ((b[4] as u64) << 32);
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.60s
    Running `target/release/measure`
    9.995 sec
    1666958173003018432


遅いです。

さらに別の方法。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 5]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            unsafe {
                let ptr: *mut u32 = b.as_ptr() as *mut u32;
                total += (*ptr as u64) + ((b[4] as u64) << 32);
            }
        }
    }
    let end = start.elapsed();


    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.59s
    Running `target/release/measure`
    9.483 sec
    1666958173003018432


遅いです。

ポインターを使った方法を試します。

まずは0xFF_FFFF_FFFFマスクを使った方法。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 8]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  0 as u8, 0 as u8, 0 as u8]);
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            unsafe {
                let ptr: *mut u64 = b.as_ptr() as *mut u64;
                total += *ptr & 0xFF_FFFF_FFFF;
            }
        }
    }
    let end = start.elapsed();


    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.60s
    Running `target/release/measure`
    6.843 sec
    1666958173003018432


速くなりました。

以下、その他の切り詰め方法。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 5]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8]);
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            let b = [b[0], b[1], b[2], b[3], b[4], 0, 0, 0];
            unsafe {
                total += *(b.as_ptr() as *mut u64);
            }
        }
    }
    let end = start.elapsed();


    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.59s
    Running `target/release/measure`
    10.022 sec
    1666958173003018432


遅い。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 5]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            total += u64::from_le_bytes([b[0], b[1], b[2], b[3], b[4], 0, 0, 0]);
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.59s
    Running `target/release/measure`
    10.024 sec
    1666958173003018432


遅い。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut buf: Vec<[u8; 5]> = Vec::new();
    for _ in 0..SIZE {
        buf.push([rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8,
                  rng.gen_range(0..256) as u8])
    }

    let start = Instant::now();
    let mut total: u64 = 0;
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let b = buf[idx];
            total += (b[0] as u64) + ((b[1] as u64) << 8) + ((b[2] as u64) << 16) + ((b[3] as u64) << 24) + ((b[4] as u64) << 32);
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.59s
    Running `target/release/measure`
    9.969 sec
    1666958173003018432


遅い。

## 40bit整数の配列の5Nバイト配列化

> `*(buf[idx*5..].as_ptr() as *mut u64) = v`が最速

40bit整数の配列を5Nバイト配列に変換するには以下の図の戦略が使えます。

![bytes.png](bytes.png)

まずは素朴な実装。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }

    let mut buf: [u8; 5 * SIZE] = [0; 5 * SIZE];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let v = vs[idx];
            let b = &mut buf[(idx*5)..];
            for i in 0..5 {
                b[i] = (v >> (8 * i)) as u8;
            }
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i*5+j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.52s
    Running `target/release/measure`
    21.127 sec
    205
    63749253


遅い。

内側のforループの展開。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }

    let mut buf: [u8; 5 * SIZE] = [0; 5 * SIZE];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let v = vs[idx];
            let b = &mut buf[(idx*5)..];
            b[0] = v as u8;
            b[1] = (v >> 8) as u8;
            b[2] = (v >> 16) as u8;
            b[3] = (v >> 24) as u8;
            b[4] = (v >> 32) as u8;
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i*5+j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.53s
    Running `target/release/measure`
    21.086 sec
    205
    63749253


展開しないもとの同じです。Go言語よりも遅いです。

関数利用。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }

    let mut buf: [u8; 5 * SIZE] = [0; 5 * SIZE];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            let src = vs[idx].to_le_bytes();
            let b = &mut buf[(idx*5)..];
            b[0] = src[0];
            b[1] = src[1];
            b[2] = src[2];
            b[3] = src[3];
            b[4] = src[4];
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i*5+j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.52s
    Running `target/release/measure`
    21.153 sec
    205
    63749253


同じでした。

最後にポインター利用。


```Rust
%%writefile src/main.rs
use std::time::Instant;
use rand;
use rand::prelude::*;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut vs: [u64; SIZE] = [0; SIZE];
    for i in 0..SIZE {
        vs[i] = rng.gen_range(0..(1<<40));
    }

    let buf: [u8; 5 * SIZE + 3] = [0; 5 * SIZE + 3];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            unsafe {
                *(buf[idx*5..].as_ptr() as *mut u64) = vs[idx];
            }
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );

    // 時間測定外の処理
    println!("{:?}", buf[0]);
    let mut total: usize = 0;
    for i in 0..SIZE {
        for j in 0..5 {
            total += buf[i*5+j] as usize;
        }
    }
    println!("{}", total);
}

```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Finished release [optimized] target(s) in 0.01s
    Running `target/release/measure`
    7.748 sec
    205
    63749253


速い。やっとRustの面目躍如。ポインターを使ってやっとGo言語を越せました。

## 5Nバイト配列から40bit整数の配列への変換

> `v = *(ans[idx*5..].as_ptr() as *mut u64) & 0xFF_FFFF_FFFF`が最速

まずは素朴な実装。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut ans:[u8; SIZE*5] = [0; SIZE*5];
    for i in 0..SIZE {
        for j in 0..5 {
            ans[i*5+j] = rng.gen_range(0..256) as u8;
        }
    }
    let mut buf:[u64; SIZE] = [0; SIZE];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for i in 0..SIZE {
            let i5 = i * 5;
            let mut v = 0;
            for j in 0..5 {
                v += (ans[i5+j] as u64) << (8 * j);
            }
            buf[i] = v;
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    let mut total: u64 = 0;
    for i in 0..SIZE {
        total += buf[i];
    }

    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.49s
    Running `target/release/measure`
    21.323 sec
    54987966921384494


内側のforループの展開。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut ans:[u8; SIZE*5] = [0; SIZE*5];
    for i in 0..SIZE {
        for j in 0..5 {
            ans[i*5+j] = rng.gen_range(0..256) as u8;
        }
    }
    let mut buf:[u64; SIZE] = [0; SIZE];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for i in 0..SIZE {
            let i5 = i * 5;
            let mut v = ans[i5] as u64;
            v += (ans[i5+1] as u64) << 8;
            v += (ans[i5+2] as u64) << 16;
            v += (ans[i5+3] as u64) << 24;
            v += (ans[i5+4] as u64) << 32;
            buf[i] = v;
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    let mut total: u64 = 0;
    for i in 0..SIZE {
        total += buf[i];
    }

    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.52s
    Running `target/release/measure`
    7.934 sec
    54987966921384494


めずらしくforループの展開が効果がありました。

関数利用。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut ans:[u8; SIZE*5] = [0; SIZE*5];
    for i in 0..SIZE {
        for j in 0..5 {
            ans[i*5+j] = rng.gen_range(0..256) as u8;
        }
    }
    let mut buf:[u64; SIZE] = [0; SIZE];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for i in 0..SIZE {
            let i5 = i * 5;
            buf[i] = u64::from_le_bytes([ans[i5], ans[i5+1], ans[i5+2], ans[i5+3], ans[i5+4], 0, 0, 0]);
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    let mut total: u64 = 0;
    for i in 0..SIZE {
        total += buf[i];
    }

    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.53s
    Running `target/release/measure`
    7.918 sec
    54987966921384494


変わりません。

別の集計方法。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut ans:[u8; SIZE*5] = [0; SIZE*5];
    for i in 0..SIZE {
        for j in 0..5 {
            ans[i*5+j] = rng.gen_range(0..256) as u8;
        }
    }
    let mut buf:[u64; SIZE] = [0; SIZE];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for i in 0..SIZE {
            let i5 = i * 5;
            buf[i] = (u32::from_le_bytes([ans[i5], ans[i5+1], ans[i5+2], ans[i5+3]]) as u64) + ((ans[i5+4] as u64) << 32);
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    let mut total: u64 = 0;
    for i in 0..SIZE {
        total += buf[i];
    }

    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.50s
    Running `target/release/measure`
    7.912 sec
    54987966921384494


ほぼ同じです。

0xFF_FFFF_FFFFのマスクによる集計方法。


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut ans:[u8; SIZE*5] = [0; SIZE*5];
    for i in 0..SIZE {
        for j in 0..5 {
            ans[i*5+j] = rng.gen_range(0..256) as u8;
        }
    }
    let mut buf:[u64; SIZE] = [0; SIZE];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            unsafe {
                buf[idx] = *(ans[idx*5..].as_ptr() as *mut u64) & 0xFF_FFFF_FFFF;
            }
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    let mut total: u64 = 0;
    for i in 0..SIZE {
        total += buf[i];
    }

    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.51s
    Running `target/release/measure`
    6.419 sec
    54987966921384494


速い。これが最速です。

念のため、


```Rust
%%writefile src/main.rs
use rand;
use rand::prelude::*;
use std::time::Instant;

fn main() {
    const COUNT: usize = 10_000_000_000;
    const SIZE: usize = 100_000;
    let mut rng: rand::rngs::StdRng = rand::SeedableRng::seed_from_u64(0);
    let mut ans:[u8; SIZE*5] = [0; SIZE*5];
    for i in 0..SIZE {
        for j in 0..5 {
            ans[i*5+j] = rng.gen_range(0..256) as u8;
        }
    }
    let mut buf:[u64; SIZE] = [0; SIZE];

    let start = Instant::now();
    for _ in 0..COUNT/SIZE {
        for idx in 0..SIZE {
            unsafe {
                buf[idx] = (*(ans[idx*5..].as_ptr() as *mut u32) as u64) + ((ans[idx*5+4] as u64) << 32);
            }
        }
    }
    let end = start.elapsed();

    println!(
        "{}.{:03} sec",
        end.as_secs(),
        end.subsec_nanos() / 1_000_000
    );
    let mut total: u64 = 0;
    for i in 0..SIZE {
        total += buf[i];
    }

    println!("{}", total);
}
```

    Overwriting src/main.rs



  ```shell
!cargo run --release
```

    Compiling measure v0.1.0 (/content/measure)
    Finished release [optimized] target(s) in 0.49s
    Running `target/release/measure`
    7.199 sec
    54987966921384494


これは遅かった。
