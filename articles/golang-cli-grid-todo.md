---
title: "goでcli作成: マンダラートをmarkdownテーブルで表示、yamlで内容を管理"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "cli"]
published: false
---

マンダラートを使ってアイデア出しをする時に、出したアイデアをテキストファイルで管理したいなーと思い、yamlファイルで作った内容をmarkdownテーブル表示のマンダラートに変換するcliをgoで作りました。

# 開発の経緯

開発などのアイデア出しの手法について色々模索しているのですが、マンダラートという手法があります。
https://ja.wikipedia.org/wiki/%E3%83%9E%E3%83%B3%E3%83%80%E3%83%A9%E3%83%BC%E3%83%88

マンダラートでは、下記のような手法でアイデアを整理します
- 3×3の9マスを書き、その中心のマスに考えたいことを記載
- 周りのマスには中心に書いたことに関連する事柄を記載
- 周りのマスのうち1つを選び、そのマスの記載内容を別の紙の中心のマスに転記し、同様に繰り返す

既存のアプリを使ったり、エクセルとかで管理しても良いのですが、自分の好きなエディタで書きたかったのと、出したアイデアの変更履歴管理したい（git）という理由から自分でツールを作ってみました。

goを選んだ理由は、
- cliが作りやすいというイメージがあるから
- 同僚から以前猛烈に「goはいいよー」と勧められたのを思い出したから
- とりあえずgoで何か作ってみたいから

で、特に合理的な理由がある訳ではないです。goでcliを作ってみたいという方の参考になれば幸いです。

# ソース

cliを作るlibraryとして、cobraを使いました。
https://github.com/spf13/cobra

またyamlの処理には以下のlibrary`goccy/go-yaml`を使いました
https://github.com/goccy/go-yaml

## cobraでmain.goファイル生成

cobraコマンドをインストールして、cli用のprojectを作成します。
```bash
$ mkdir yourProject
$ cd yourProject
$ go get -u github.com/spf13/cobra/cobra
$ cobra init --pkg-name yourProject
```

:::message
デフォルトだとライセンスがApache 2.0になったり、アカウント名が入らないので、それらを指定する場合は引数を追加します。
```bash
$ cobra init --pkg-name yourProject --author account --license mit
```
:::

この時点で、`cobra`によるファイル構成は以下のような感じになります。
```bash
$ tree
.
├── LICENSE
├── cmd
│   └── root.go
├── main.go
```

`main.go`がcliのentry pointで、中身は`cmd/root.go`の`Execute()`関数を実行しているだけです。`yourProject`配下で下記コマンドを実行すると、
```bash
$ go install
```
`yourProject`コマンドが実行形式ファイルとして、`$GOPATH/bin`以下に配置されます。`cmd/root.go`の`Run`関数に`fmt.Println`とかを仕込んでおけば、コマンド実行の結果を確認できます。
```go:cmd/root.go
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Hello world")
	},
```
```bash
$ go install
$ $GOPATH/bin/yourProject
Hello world
```

## サブコマンドの実装1: yamlファイルを生成

yamlファイルを0から作るのは面倒なので、雛形を生成するサブコマンド`generate`を作成します。
```bash
$ cobra add generate
$ tree
.
├── LICENSE
├── cmd
│   ├── generate.go
│   └── root.go
├── main.go
```

![](https://storage.googleapis.com/zenn-user-upload/n1bvffi96rve1qx3js0p9gklroa6)
最終的には上記のように、9×9マスのグリッドを作成したいので、下記の型を作成しました。

```go:todo/todo.go
// 3*3 panel = (3*3 item) * 9
type Todo struct {
	Goal  string
	Panel [9]Panel
}

// 1 panel = 3*3 item
type Panel struct {
	Cell [9]string
}
```
3×3マスを1パネルとしてその要素`Cell`を`string`の配列で定義、パネルを3×3個並べて最終的に9×9セルのグリッドを作成します。各要素には文字列を表示しますが、パネル4以外のパネルの中心には、パネル4に表示された文字列を表示します。例えば、パネル1の中心には、パネル4の1に表示された文字列を表示。`Todo`の`Goal`には、パネル4のセル4に表示する文字列を格納します。

yamlファイルを作成する処理は以下の通りで、空の`Todo`を作ってエンコードして、yamlファイルを生成します。
```go:cmd/generate.go
	Run: func(cmd *cobra.Command, args []string) {
		var yamlFile string
		if len(args) == 0 {
			yamlFile = "todo.yaml"
		} else {
			yamlFile = args[0]
		}

		todo := todo.Todo{
			Goal:  "",
			Panel: [9]todo.Panel{},
		}

		data, err := yaml.Marshal(todo)
		if err != nil {
			panic(err)
		}

		err = ioutil.WriteFile(yamlFile, data, 0644)
		if err != nil {
			log.Fatal(err)
		}
	},
```

これで入力に使えるフォーマットのyamlファイルを生成できます。
```
$ go install
$ gridtodo generate
$ cat todo.yaml
goal: ""
panel:
- cell:
  - ""
  - ""
  - ""
  - ""
  - ""
  - ""
  - ""
  - ""
  - ""
- cell:
  - ""
...
（長いので省略）
```

## サブコマンドの実装2: yamlファイルをmarkdown tableに変換

入力したyamlファイルをmarkdown tableとして出力するコマンド`show`を作成します。
```bash
$ cobra add show
$ tree
.
├── LICENSE
├── cmd
│   ├── generate.go
│   ├── root.go
│   └── show.go
├── main.go
└── todo
│   ├── md.go
    └── todo.go
```

yamlファイルをデコードして、markdown table形式に変換して、標準出力に表示。
```go:cmd/show.go
	Run: func(cmd *cobra.Command, args []string) {
		if len(args) == 0 {
			fmt.Fprintln(os.Stderr, "provide yaml filename")
			return
		}

		yamlFile := args[0]

		buf, err := ioutil.ReadFile(yamlFile)
		if err != nil {
			log.Fatal(err)
		}

		grid := todo.Todo{
			Goal:  "",
			Panel: [9]todo.Panel{},
		}

		err = yaml.Unmarshal([]byte(buf), &grid)
		if err != nil {
			panic(err)
		}

		md := todo.Convert(grid)
		fmt.Println(md.Table())
	},
```

```go:todo/md.go
package todo

import (
	"fmt"
	"os"
)

// Markdown store strings to create grid-todo markdown table
type Markdown struct {
	Row [9]string
}

// Table show markdown table with header
func (md Markdown) Table() string {
	return "||||||||||\n|-|-|-|-|-|-|-|-|-|\n" +
		md.Row[0] + "\n" + md.Row[1] + "\n" + md.Row[2] + "\n" + md.Row[3] + "\n" + md.Row[4] + "\n" +
		md.Row[5] + "\n" + md.Row[6] + "\n" + md.Row[7] + "\n" + md.Row[8] + "\n"
}

// CreateMarkdownRow create 1 row of markdown table format from Panel
// create 1st row
// from
// [1, 2, 3, 4, 5, 6, 7, 8, 9]
// [11, 12, 13, 14, 15, 16, 17, 18, 19]
// [21, 22, 23, 24, 25, 26, 27, 28, 29]
// =>
// |1|2|3|10|11|12|20|21|22|
// |4|5|6|14|15|16|24|25|26|
// |7|8|9|17|18|19|27|28|29|
func CreateMarkdownRow(row int, panelRow int, grid Todo) string {
	if row < 0 || row > 2 {
		fmt.Fprintln(os.Stderr, "Error: row should be 0, 1, or 2")
		return ""
	}
	offset := row * 3
	panelOffset := panelRow * 3
	centerid := panelOffset

	panel := grid.Panel
	center := panel[4]
	left := panel[0+panelRow*3]
	middle := panel[1+panelRow*3]
	right := panel[2+panelRow*3]

	goal := Escape(grid.Goal)
	l := left.EscapeCell()
	m := middle.EscapeCell()
	r := right.EscapeCell()
	c := center.EscapeCell()

	b := make([]byte, 0, 128)
	for i := 0; i < 3; i++ {
		b = append(b, "|"...)
		if row == 1 && i == 1 {
			b = append(b, c[centerid]...)
			centerid++
		} else {
			b = append(b, l[i+offset]...)
		}
	}
	for i := 0; i < 3; i++ {
		b = append(b, "|"...)
		if row == 1 && i == 1 {
			if panelRow == 1 {
				b = append(b, []byte(goal)...)
			} else {
				b = append(b, c[centerid]...)
				centerid++
			}
		} else {
			b = append(b, m[i+offset]...)
		}
	}
	for i := 0; i < 3; i++ {
		b = append(b, "|"...)
		if row == 1 && i == 1 {
			b = append(b, c[centerid]...)
			centerid++
		} else {
			b = append(b, r[i+offset]...)
		}
	}
	b = append(b, "|"...)

	return string(b)
}

// Convert convert given todo grid string (9*9 string) into markdown table with 9 rows
func Convert(grid Todo) Markdown {
	md := Markdown{}

	for i := 0; i < 3; i++ {
		md.Row[0+i*3] = CreateMarkdownRow(0, i, grid)
		md.Row[1+i*3] = CreateMarkdownRow(1, i, grid)
		md.Row[2+i*3] = CreateMarkdownRow(2, i, grid)
	}

	return md
}
```

yamlからの入力文字列は長さ9の配列に格納されているので、3*3のmarkdown table形式に変換します。
```go
// 変換のイメージ
// [1, 2, 3, 4, 5, 6, 7, 8, 9]
// [11, 12, 13, 14, 15, 16, 17, 18, 19]
// [21, 22, 23, 24, 25, 26, 27, 28, 29]
// =>
// |1|2|3|10|11|12|20|21|22|
// |4|5|6|14|15|16|24|25|26|
// |7|8|9|17|18|19|27|28|29|
```
配列を3つずつ処理しますが、markdown tableの1行目に表示する要素は、3つの配列の最初から3つめまでの要素を順番に並べたものになります。同様に、2行目に表示するのは4番目から6番目の要素、3行目に表示する要素は7番目から9番目の要素です。各パネルの真ん中には、中心のパネルの要素を表示します。また、縦線`|`はmarkdownでのcellの区切りになるので、エスケープしています。

実際にテスト用に作ってみたyamlファイルを出力すると以下のようになります（vscodeのmarkdownプレビューの拡張機能を使って表示）。
```bash
$ gridtodo show test.yml
||||||||||
|-|-|-|-|-|-|-|-|-|
|🐶|🐶&#124;|🐶|🐱|🐱|🐱|⭐|⭐|⭐|
|🐶|犬|🐶|🐱|猫|🐱|⭐|星|⭐|
|🐶|🐶|🐶|🐱|🐱|🐱|⭐|⭐|⭐|
|🎵|🎵|🎵|犬|猫|星|🚗|🚗|🚗|
|🎵|音|🎵|音|😃<br>😃|車|🚗|顔|🚗|
|🎵|🎵|🎵|走|球|青|🚗|🚗|🚗|
|🏃|🏃|🏃|⚽|⚽|⚽|🔵|🔵|🔵|
|🏃|走|🏃|⚽|球|⚽|🔵|青|🔵|
|🏃|🏃|🏃|⚽|⚽|⚽|🔵|🔵|🔵|
```
文字列として縦線を入力した場合、ちゃんと`$#124;`にエスケープされていることが分かります。また、改行コード`\n`を指定すると、markdownの中で改行出来るようにしています。

![](https://storage.googleapis.com/zenn-user-upload/3ky52rh1atp1i7vq6m5spovml5ty)

入力用に用意したyamlファイルは下記。
```yml:test.yml
goal: "😃\n😃"
panel:
- cell:
  - "🐶"
  - "🐶|"
  - "🐶"
  - "🐶"
  - "🐶"
  - "🐶"
  - "🐶"
  - "🐶"
  - "🐶"
- cell:
  - "🐱"
  - "🐱"
  - "🐱"
  - "🐱"
  - "🐱"
  - "🐱"
  - "🐱"
  - "🐱"
  - "🐱"
- cell:
  - "⭐"
  - "⭐"
  - "⭐"
  - "⭐"
  - "⭐"
  - "⭐"
  - "⭐"
  - "⭐"
  - "⭐"
- cell:
  - "🎵"
  - "🎵"
  - "🎵"
  - "🎵"
  - "🎵"
  - "🎵"
  - "🎵"
  - "🎵"
  - "🎵"
- cell:
  - "犬"
  - "猫"
  - "星"
  - "音"
  - "顔"
  - "車"
  - "走"
  - "球"
  - "青"
- cell:
  - "🚗"
  - "🚗"
  - "🚗"
  - "🚗"
  - "🚗"
  - "🚗"
  - "🚗"
  - "🚗"
  - "🚗"
- cell:
  - "🏃"
  - "🏃"
  - "🏃"
  - "🏃"
  - "🏃"
  - "🏃"
  - "🏃"
  - "🏃"
  - "🏃"
- cell:
  - "⚽"
  - "⚽"
  - "⚽"
  - "⚽"
  - "⚽"
  - "⚽"
  - "⚽"
  - "⚽"
  - "⚽"
- cell:
  - "🔵"
  - "🔵"
  - "🔵"
  - "🔵"
  - "🔵"
  - "🔵"
  - "🔵"
  - "🔵"
  - "🔵"
```

1つ未解決の問題として、yamlでダブルクォートなしで改行コードを入れた場合、エスケープ時にエスケープされず`\n`のまま表示されてしまうというのがあります。
```yaml
goal: 😃\n😃 // ダブルクォートなしだと、\nをエスケープしても表示が😃\n😃となる。\nが<br>に変換されない
```
とりあえずダブルクォートで囲んでいる場合は正しく改行されているので、ちゃんと調査していませんが、解決方法があれば実装するかもしれません。

# まとめ

cobraのおかげかと思いますが、goでcliを作るのは気軽に出来るので良いですね。個人的には`go fmt`で強制的にソースがフォーマットされるのが、実装時に非常に快適でした。ソースをどうフォーマットするのかに全くこだわりがないからかもしれませんが、これは他の言語にもデフォルトで是非いれて欲しいなーと思う機能です。

改善点としては、長めの文字列を入れるとmarkdown tableのレイアウトが崩れるかもしれないので、例えばデフォルト5文字とかで自動改行させるようにするとか、入力文字数を制限する等の機能はあってもよいかなーと思いました。

# 参考にした記事

https://zenn.dev/tomi/articles/2020-12-13-go-cli
https://towardsdatascience.com/how-to-create-a-cli-in-golang-with-cobra-d729641c7177