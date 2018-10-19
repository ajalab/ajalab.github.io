---
layout: post
title: ブラウザで動くガウス過程回帰のデモ
tags:
  - TypeScript
  - 機械学習
---

[ ![title](/assets/gaussian-process-vue-20181019.png) https://ajalab.github.io/gaussian-process-vue/](https://ajalab.github.io/gaussian-process-vue/)

TypeScriptとVue.jsの手習いを兼ねて作った。ガウス過程に関する主要なコードは[https://github.com/ajalab/gaussian-process-vue/blob/master/src/gaussian_process.ts](https://github.com/ajalab/gaussian-process-vue/blob/master/src/gaussian_process.ts) にある。

## 機能

- ガウス過程による回帰 (ガウスカーネル)
- クリックによる学習データ点の追加
- ノイズ定数$\beta$およびガウスカーネル$k(x, x^\prime)=e^\{-\|\|x-x^\prime\|\|t\}$のハイパーパラメータ$t$ の手動設定
- ハイパーパラメータ$t$の勾配法による最適化

## メモ

パラメータ最適化がたまに失敗する。雑に実装したせいでバグが入っているか、数値的に不安定な処理が入っているからかもしれない。

## 参考

- PRML 6章 カーネル法
- Gaussian Process for Machine Learning 2章 Regression
- [https://kivantium.net/ml-gaussian-process](https://kivantium.net/ml-gaussian-process)