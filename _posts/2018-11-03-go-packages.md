---
layout: post
title: go/packagesについて
tags:
  - Go
---

## 概要
Go 1.11のリリースから、パッケージ解析等の目的でGoのパッケージをロードするためのライブラリ[`x/tools/go/packages`](https://godoc.org/golang.org/x/tools/go/packages)がGo Toolsに追加された（[リリースノート](https://golang.org/doc/go1.11#gopackages)）。このライブラリはGo 1.11から導入された[modulesシステム](https://golang.org/cmd/go/#hdr-Modules__module_versions__and_more)をサポートしている。まだ標準ライブラリには入っていないが、これはmodules非対応の[`go/build`](https://golang.org/pkg/go/build/)を代替するものである。

ところで、Go Toolsにはパッケージ読み込みのためのライブラリ（[`x/tools/go/loader`](https://godoc.org/golang.org/x/tools/go/loader)）が既に存在する。公式のアナウンス等は無くこのパッケージの立ち位置は不明瞭だが、`x/tools/go/loader`に関する[issueコメント](https://github.com/golang/go/issues/14120#issuecomment-383994980)や、`loader.Program`を使用する`ssautil.CreateProgram`が[deprecatedになった](https://github.com/golang/tools/commit/5b5e9c877a65dcf1f386adf9542d43cd797cc43a)ことから、今後は`x/tools/go/loader`ではなく`x/tools/go/packages`を使うべきだろう。

## 使い方

パッケージのロードには、まず[`packages.Config`](https://godoc.org/golang.org/x/tools/go/packages#Config)構造体に諸設定を記述する。もっとも重要なのは`Mode`メンバで、ローダーによる解析のレベルを[`LoadMode`](https://godoc.org/golang.org/x/tools/go/packages#LoadMode)型の値を代入して指定する。以下に挙げるが、下に行くほどレベルが深く、ローダーによって返却される`packages.Package`構造体に含まれる情報も増える。

- `LoadFiles` : パッケージに所属する*.goファイルの一覧を得る
- `LoadImports`: インポートの依存解析を行う
- `LoadTypes`: 宣言レベルでの型解析を行う
- `LoadSyntax`: 指定したパッケージについてASTの構築・型解析を行う
- `LoadAllSyntax`: 指定したパッケージ及びそれらの依存パッケージにおいてASTの構築・型解析を行う

設定を[`packages.Load`](https://godoc.org/golang.org/x/tools/go/packages#Load)に与えて呼び出す。第２引数以降で読み込むパッケージを指定するが、これにはGoのビルドシステムの引数で用いられる`[packages]`パターンを基本的に用いる。`[packages]`パターンの詳細は`go help packages`に記述がある。実際、ライブラリ内部では`go list`コマンドを実行している。

`[packages]`パターン以外にも、`"patternName=string"`の書式で詳細なクエリを指定することができる。現在公式に実装されているクエリは`file`と`pattern`で、前者は`"file=path/to/file.go"`の形式で`path/to/file.go`を含むパッケージを指定する。後者は`"pattern=string"`の形式で、`string`が`[packages]`パターンとして`go list`に渡される。これはエスケープの用途で用いる。第３のクエリとして`"name=identifier"`クエリがあり、パッケージ名での指定ができる（例：`"name=rand"`で`math/rand`と`crypto/rand`を読み込む）。これを書いた時点では、`name`クエリは実装途中であるとドキュメントに記載されていたが、既に[動くものが実装されている](https://github.com/golang/tools/commit/63d31665e311d0da81db6a27060589b038fad816#diff-75b4895cfda9cd788ec45840ff5948c6)。

`packages.Load`は読み込んだパッケージのリストを[`*packages.Package`](https://godoc.org/golang.org/x/tools/go/packages#Package)のスライスとして返却する。パッケージの順番及び数は`packages.Load`の第２引数以降で指定したパターンと必ずしも一致しないので注意する。

## サンプル

以下のプログラムは`testdata/t.go`を[`x/tools/go/packages`](https://godoc.org/golang.org/x/tools/go/packages)によって[`runtime`](https://golang.org/pkg/runtime/)パッケージとともに読み込み、SSA形式に変換してから[`x/tools/go/interp`](https://godoc.org/golang.org/x/tools/go/packages)のAPIによってインタプリタ実行する。

main.go

```golang
package main

import (
	"go/types"
	"log"

	"golang.org/x/tools/go/packages"
	"golang.org/x/tools/go/ssa"
	"golang.org/x/tools/go/ssa/interp"
	"golang.org/x/tools/go/ssa/ssautil"
)

func main() {
	target := "testdata/t.go"

	// interp.Interpretで実行する対象は、依存パッケージを含めすべて
	// SSA形式に変換されている必要があるため、LoadAllSyntaxモードを指定する
	cfg := &packages.Config{
		Mode:  packages.LoadAllSyntax,
		Tests: false,
	}

	// runtimeモジュールはinterp.Interpretによる実行の際に必要。
	// 大抵依存パッケージの推移的閉包に含まれているため、明示的に指定しなくても動くことがある
	pkgs, err := packages.Load(cfg, "name=runtime", target)
	if err != nil {
		log.Fatalf("failed to load package: %v", err)
		return
	}

	// SSA形式に変換
	ssaProg, ssaPkgs := ssautil.AllPackages(pkgs, 0)
	ssaProg.Build()

	// mainパッケージを探す
	var mainPkg *ssa.Package
	for _, ssaPkg := range ssaPkgs {
		if ssaPkg.Pkg.Name() == "main" {
			mainPkg = ssaPkg
		}
	}

	// 実行
	interp.Interpret(mainPkg, 0, &types.StdSizes{WordSize: 8, MaxAlign: 8}, "hoge", []string{})
}

```

testdata/t.go

```golang
package main

import "fmt"

func main() {
	fmt.Println("hello golang")
}
```

実行結果

```sh
$ go run main.go
hello golang
```

## 参考

- `gopackages`コマンドのソース [https://github.com/golang/tools/blob/master/go/packages/gopackages/main.go](https://github.com/golang/tools/blob/master/go/packages/gopackages/main.go)