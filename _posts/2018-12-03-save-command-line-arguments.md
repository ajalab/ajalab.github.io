---
layout: post
title: Pythonのコマンドライン引数を保存して使いまわす
tags:
  - Python
---

機械学習や数値実験を行うプログラムは、しばしば多数のパラメータを扱う必要があります。
これらをコマンドライン引数で指定する形にしていると毎回入力するのが煩雑です[^1]。
また、パラメータは実験を再現・考察するために当然記録しておくものですが、再実験の際に記録済みのパラメータを読み込んだり、部分的に変更して実行するといった操作ができると便利です。
そのために、`argparse`でパースしたコマンドライン引数を保存・読み込み・変更できるような仕組みを考えます。

[`argparse.ArgumentParser`には`fromfile_prefix_char`というオプションがあります](https://docs.python.org/3.7/library/argparse.html#fromfile-prefix-chars)。これはまさに今回の目的を念頭においているものです。しかしコマンドライン上でファイルを指定するための特殊記号を新たに予約する必要があったり、引数を指定するファイルは1行ずつオプション名と値を書く形式をとらなければならない等、取り回しが不便です。

`ArgumentParser.parse_args`のオプション引数である`namespace`を利用するとシンプルに実装することができます。

以下のような、`argparse` モジュールでコマンドライン引数をパースするシンプルなPythonアプリについて考えます。

```python
#!/usr/bin/env python3
# app.py

import argparse


def build_parser():
    parser = argparse.ArgumentParser()

    # positional (required) arguments: a
    parser.add_argument(
        'a',
        type=int,
    )

    # keyword (optional) arguments: --one, --two
    parser.add_argument(
        '--one',
        type=str,
        default='default',
    )
    parser.add_argument(
        '--two',
        type=str,
        default='default',
    )
    return parser


def main():
    parser = build_parser()
    args = parser.parse_args()
    print(f'a: {args.a}\none: {args.one}\ntwo: {args.two}')


if __name__ == "__main__":
    main()
```

```shell
$ ./app.py 10 --one foo
a: 10
one: foo
two: default
```

パスを指定してコマンドライン引数を保存する`--dst`と、保存したコマンドライン引数を読み込む`--src`の2つを新たにコマンドライン引数として追加します。`--src`の結果に応じて引数のロードがなされるので、最初にこの2つのオプションについてだけ引数を取得するパーサを別途に用意します。
適当な名前が思いつかないので`meta_parser`とでもしておきます。

```python
def build_meta_parser():
    parser = argparse.ArgumentParser(add_help=False)
    parser.add_argument(
        '--src',
        type=str,
    )
    parser.add_argument(
        '--dst',
        type=str,
    )
    return parser


def build_parser(meta_parser):
    parser = argparse.ArgumentParser(parents=[meta_parser])

    # positional (required) arguments
    parser.add_argument(
        'a',
        type=int,
    )

    # keyword (optional) arguments
    parser.add_argument(
        '--one',
        type=str,
        default='default',
    )
    parser.add_argument(
        '--two',
        type=str,
        default='default',
    )
    return parser
```

各関数の1行目では、ヘルプメッセージ（--helpまたは-hを指定したときに出力される文字列）にすべてのオプションを表示させるために`add_help`や`parents`を指定しています。

ArgumentParser.parse_argsの返り値は`argparse.Namespace`というクラスのインスタンスです。これを保存・読込するための関数は以下のようになります。今回は保存形式をJSONとします。

```python
def save_args(args, dst):
    if dst is None:
        return
    with open(dst, 'w') as f:
        json.dump(vars(args), f)


def load_args(src):
    if src is not None and Path(src).exists():
        with open(src, 'r') as f:
            ns = json.load(f)
        return argparse.Namespace(**ns)
```

`--src`や`--dst`の有無によってコマンドライン引数を読み込み・保存します。
コードの全体を以下に示します。

```python
#!/usr/bin/env python3
# app2.py

import argparse
import json
from pathlib import Path


def build_meta_parser():
    parser = argparse.ArgumentParser(add_help=False)
    parser.add_argument(
        '--src',
        type=str,
    )
    parser.add_argument(
        '--dst',
        type=str,
    )
    return parser


def build_parser(meta_parser):
    parser = argparse.ArgumentParser(parents=[meta_parser])

    # positional (required) arguments
    parser.add_argument(
        'a',
        type=int,
    )

    # keyword (optional) arguments
    parser.add_argument(
        '--one',
        type=str,
        default='default',
    )
    parser.add_argument(
        '--two',
        type=str,
        default='default',
    )
    return parser


def save_args(args, dst):
    if dst is None:
        return
    with open(dst, 'w') as f:
        json.dump(vars(args), f)


def load_args(src):
    if src is not None and Path(src).exists():
        with open(src, 'r') as f:
            ns = json.load(f)
        return argparse.Namespace(**ns)


def main():
    meta_parser = build_meta_parser()
    parser = build_parser(meta_parser)

    # parse --src and --dst
    margs, rest = meta_parser.parse_known_args()
    namespace = load_args(margs.src)

    # parse arguments for the app
    args = parser.parse_args(args=rest, namespace=namespace)

    print(f'a: {args.a}\none: {args.one}\ntwo: {args.two}')

    save_args(args, margs.dst)


if __name__ == "__main__":
    main()
```
`main`の4行目でまず`--src`、`--dst`についてのみ`parse_known_args`でパースします。
パースされなかった引数のリストは返り値の2番目にあたります。
ポイントは`main`における

```python
    args = parser.parse_args(args=rest, namespace=namespace)
```

の行で、第1引数で`--dst`、`--src`以外についてのコマンドライン引数のリストを、第2引数でロードしたコマンドライン引数を格納したオブジェクトを指定します。
`parse_args`はデフォルトで空の`Namespace`をもとにコマンドライン引数のキーと値を格納しますが、`namespace`で格納先のオブジェクトを指定することができます。
`parse_args`はコマンドライン引数が明示的に指定されている場合はオブジェクトのアトリビュートを上書きしますが、指定されていない（デフォルト値）場合は上書きしません[^2]。

挙動を確認します。

```shell
$ ./app2.py --help
usage: app2.py [-h] [--src SRC] [--dst DST] [--one ONE] [--two TWO] a

positional arguments:
  a

optional arguments:
  -h, --help  show this help message and exit
  --src SRC
  --dst DST
  --one ONE
  --two TWO
```

```shell
$ ./app2.py 100 --one Foo --dst args.json
a: 100
one: Foo
two: default
$ ./app2.py 200 --src args.json
a: 200
one: Foo
two: default
$ ./app2.py 300 --one Bar --src args.json
a: 300
one: Bar
two: default
$ ./app2.py 400 --two Baz --src args.json
a: 400
one: Foo
two: Baz
```

```shell
$ cat args.json
{"src": null, "dst": null, "a": 100, "one": "Foo", "two": "default"}
```
`parser`の`parents`に`meta_parser`を指定した関係で保存される引数にゴミが残っていますが、気になる場合は削除する処理を挟むと良いでしょう。

これで元のアプリのパーサ部分にさほど大きな変更を施すことなく、コマンドライン引数のロード・セーブ処理を挟むことができました。
この方法はコマンドライン引数の指定が必須であるとき（positionalな引数や`required`指定されたオプション引数）にそれらの入力を省くことはできませんが、オプション引数の指定は常に任意であるべき[^3]ですから、ほとんどのユースケースに適応することができると思います。

[^1]: ChainerRLが提供しているDQNのサンプル（[https://github.com/chainer/chainerrl/blob/master/examples/atari/dqn/train_dqn.py](https://github.com/chainer/chainerrl/blob/master/examples/atari/dqn/train_dqn.py)）
[^2]: parse_argsの実装（[https://github.com/python/cpython/blob/3.7/Lib/argparse.py#L1774](https://github.com/python/cpython/blob/3.7/Lib/argparse.py#L1774)）参考。
[^3]: requiredオプションに関する注記（[https://docs.python.org/3.7/library/argparse.html#required](https://docs.python.org/3.7/library/argparse.html#required)）