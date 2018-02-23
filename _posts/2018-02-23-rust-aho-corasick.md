---
layout: post
title: RustでAho-Corasickの実装
---

Rustに入門しました。アルゴリズム実装の手習いも兼ねて、複数パターンマッチングのための[Aho-Corasick法](https://ja.wikipedia.org/wiki/%E3%82%A8%E3%82%A4%E3%83%9B%E2%80%93%E3%82%B3%E3%83%A9%E3%82%B7%E3%83%83%E3%82%AF%E6%B3%95)を実装します。[http://www.cs.uku.fi/~kilpelai/BSA05/lectures/slides04.pdf](http://www.cs.uku.fi/~kilpelai/BSA05/lectures/slides04.pdf) を見ながら書きます。

データ構造

```rust
use std::collections::HashMap;
use std::collections::VecDeque;

type NodeId = usize;
type WordId = usize;

#[derive(Debug)]
struct Node {
    goto: HashMap<char, NodeId>,
    fail: NodeId,
    out: Vec<WordId>,
}

#[derive(Debug)]
pub struct PMA {
    nodes: Vec<Node>,
    words: Vec<String>,
}
```

ノード集合の表現にいわゆるarena patternを使います。ノードの全ての所有権はarena(ここでは`PMA`の`nodes`)が持ち, 各ノードは他のノードへのリンクを`NodeId`で保持します。パターンの集合もオートマトン`PMA`が所有権を管理します。

以下実装を`impl PMA {  }`の中に書きます。まずはグラフ構造に関連するパートです

```rust
impl PMA {
    pub fn new() -> Self {
        PMA {
            nodes: vec![],
            words: vec![],
        }
    }

    fn alloc(&mut self) -> NodeId {
        let id = self.nodes.len();
        let node = Node {
            goto: HashMap::new(),
            fail: 0,
            out: vec![],
        };
        self.nodes.push(node);
        id
    }

    fn get_node(&self, id: NodeId) -> &Node {
        &self.nodes[id]
    }

    pub fn get_words(&self) -> &Vec<String> {
        &self.words
    }

    fn get_edges(&self, id: NodeId) -> Vec<(char, NodeId)> {
        self.nodes[id]
            .goto
            .iter()
            .map(|(&c, &q2)| (c, q2))
            .collect::<Vec<_>>()
    }

    fn get_node_mut(&mut self, id: NodeId) -> &mut Node {
        &mut self.nodes[id]
    }
...
```
`alloc`は新しいノードを作り対応する`NodeId`を返します。`fail`の指す先はデフォルトで根ノードになっています。 `get_edges`は指定したノードに生えている辺をすべて取得して`Vec`で返します。

次にオートマトンを作る部分です。まずはTrie木の構築から
```rust
    pub fn build<S: Into<String>>(&mut self, words: Vec<S>) {
        self.build_goto(words);
        self.build_failure();
    }

    fn build_goto<S: Into<String>>(&mut self, words: Vec<S>) {
        let root = self.alloc();
        let words: Vec<String> = words.into_iter().map(|s| s.into()).collect();

        for (word_id, word) in words.iter().enumerate() {
            let mut q = root;
            for c in word.chars() {
                if let Some(&q_next) = self.get_node(q).goto.get(&c) {
                    q = q_next;
                } else {
                    let q_new = self.alloc();
                    self.get_node_mut(q).goto.insert(c, q_new);
                    q = q_new;
                }
            }
            self.get_node_mut(q).out.push(word_id);
        }
        self.words = words;
    }
```
`String`と`&str`を両方受け取れるようにするために`Into`トレイトを使っています ([参考](http://hermanradtke.com/2015/05/06/creating-a-rust-function-that-accepts-string-or-str.html))。パターン集合は参照で保持していてもよいですが、ライフタイムの取り回しが面倒なので`PMA`が所有権を奪う形にしています。

`self`の借用に関する制限がややこしいです。例えば`build_goto`について、`self.words`への代入を先に済ませておいてからループを
```rust
for (word_id, word) in self.words.iter().enumerate() {
```
とすると``error[E0502]: cannot borrow `*self` as mutable because `self.words` is also borrowed as immutable``で怒られます。

`goto`に失敗する際に巻き戻しするためのfailure linkは以下のように構築します。
```rust
    fn build_failure(&mut self) {
        let mut queue = VecDeque::new();

        for (_, &q) in self.get_node(0).goto.iter() {
            queue.push_back(q);
        }

        while let Some(q1) = queue.pop_front() {
            for (c, q2) in self.get_edges(q1) {
                queue.push_back(q2);

                let mut q = q1;
                while q != 0 {
                    q = self.get_node(q).fail;
                    if let Some(&q_target) = self.get_node(q).goto.get(&c) {
                        q = q_target;
                        break;
                    }
                }
                let out = &self.get_node(q).out.clone();
                let mut node = self.get_node_mut(q2);
                node.fail = q;
                node.out.extend(out);
            }
        }
    }
```
`fail`は単語のsuffixesに対応するノードのうち最長のものを選びます。
`fail(w)`で単語`w`からfail遷移するノードをあらわすとすると、
`fail(wa)`(`a`は文字)に対応するノードは`{ xa | x: suffix of w }`に対応するノードのうち最長のものになります。

最後の方で`out`をcloneしているのは`self`の所有権借用で怒られないようにするためです。もっと良い方法はないものか...

最後に、実際に探索を行う関数です。

```rust
    pub fn find(&self, text: &str) -> Vec<Vec<usize>> {
        let word_len: Vec<usize> = self.words.iter().map(|s| s.chars().count()).collect();
        let mut q: NodeId = 0;
        let mut result: Vec<Vec<usize>> = self.words.iter().map(|_| vec![]).collect();
        for (i, c) in text.chars().enumerate() {
            loop {
                let node = self.get_node(q);
                if let Some(&q_goto) = node.goto.get(&c) {
                    q = q_goto;
                    break;
                }
                if q == 0 {
                    break;
                }
                q = node.fail;
            }
            for &word_id in &self.get_node(q).out {
                result[word_id].push(i + 1 - word_len[word_id]);
            }
        }
        result
    }
}
```

各パターンの出現位置を`Vec`で返します。最初の行で各パターンの長さを計算していますが、utf-8扱いの`char`の個数を数えるコストは馬鹿にならないので構築時にやるべきでしょう。
今回は1文字を`char`として実装したので、見た目の上での出現位置が比較的簡単に計算できますが、実用上は`u8`を単位にした方が使いやすいと思われます。実際、[既存実装(https://github.com/BurntSushi/aho-corasick)](https://github.com/BurntSushi/aho-corasick) ではパターン文字列の型指定に`AsRef[u8]`トレイトを用いています。

実行例
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn work_with_str() {
        let mut pma: PMA = PMA::new();
        pma.build(vec!["he", "she", "his", "hers"]);

        let result = pma.find("ushers");
        assert_eq!(result[0], vec![2]);
        assert_eq!(result[1], vec![1]);
        assert_eq!(result[2], vec![]);
        assert_eq!(result[3], vec![2]);
    }

    #[test]
    fn work_with_string() {
        let mut pma: PMA = PMA::new();
        pma.build(vec![
            "he".to_string(),
            "she".to_string(),
            "his".to_string(),
            "hers".to_string(),
        ]);

        let result = pma.find(&"ushers".to_string());
        assert_eq!(result[0], vec![2]);
        assert_eq!(result[1], vec![1]);
        assert_eq!(result[2], vec![]);
        assert_eq!(result[3], vec![2]);
    }

    #[test]
    fn work_with_japanese() {
        let mut pma: PMA = PMA::new();
        pma.build(vec!["明日", "日", "日曜日"]);

        let result = pma.find("明日は日曜日ですか");
        assert_eq!(result[0], vec![0]);
        assert_eq!(result[1], vec![1, 3, 5]);
        assert_eq!(result[2], vec![3]);
    }
}

```
完全なソースコードは[https://github.com/ajalab/rust_study_algo/blob/master/src/aho_corasick.rs](https://github.com/ajalab/rust_study_algo/blob/master/src/aho_corasick.rs)にあります。

参考

- [Aho Corasick 法 - naoyaのはてなダイアリー](http://d.hatena.ne.jp/naoya/20090405/aho_corasick)
- [Spaghetti Source - 複数パターン検索 (Aho-Corasick)](http://www.prefield.com/algorithm/string/aho_corasick.html)
