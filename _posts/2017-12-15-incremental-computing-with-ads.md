---
layout: post
title: 抽象データ構造におけるインクリメンタル計算
---

輪講で紹介しました. 普段の研究と違う話題です. スライドを腐らせるのも勿体無いので置いておく.

[Morihata A. (2016) Incremental Computing with Abstract Data Structures. In: Kiselyov O., King A. (eds) Functional and Logic Programming. FLOPS 2016](https://link.springer.com/chapter/10.1007/978-3-319-29604-3_14)

[スライド](https://github.com/ajalab/slides/blob/master/Incremental%20Computing%20with%20Abstract%20Data%20Structures.pdf)

[インクリメンタル計算](https://en.wikipedia.org/wiki/Incremental_computing)とは, ある計算処理について入力の変化が起きたとき, それに伴う出力の変化を前の出力結果を元にして効率よく計算する手法です. 提案手法では, 一般的なデータ構造についてshortcut fusionのアイディアを用いることで, 任意のデータ構造とfold計算からincremental computingを導出する手法を提案しています.

メモ

- Introductionで紹介されているSetに対するinsertの例は正しく動作しないのではないか. 2章で紹介されている対策(foldの代わりにupward accumulationを使う) を適用するのが前提か.
- どの部分で計算が効率化されたのかはっきりしない. 木を走査する回数が減っているような気はする... 時間ができたら実測する.
