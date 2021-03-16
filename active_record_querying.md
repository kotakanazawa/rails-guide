[Active Record クエリインターフェイス \- Railsガイド](https://railsguides.jp/active_record_querying.html)

## Active Recordの役割
- DBにクエリを発行してオブジェクトを取り出す

## クエリの種類

### find

```ruby
client = Client.find(10)
```

以下と同様のクエリになる。

```sql
SELECT * FROM clients WHERE(clients.id = 10) LIMIT 1
```

引数に主キーの配列を渡すことで、複数オブジェクトへのクエリを発行することもできる。

```ruby
clients = Client.find([1,10])
```

これと同じになる。

```sql
SELECT * FROM clients WHERE(clients.id = IN (1, 10))
```

ひとつでもレコードが見つからない場合は`ActiveRecord::RecordNotFound`を出す。

### take

レコードをひとつ取り出す。どのレコードが取り出されるかはわからない。いつ使うんだこれ？

```ruby
client = Client.take
```

同等のクエリ。

```sql
SELECT * FROM clients LIMIT 1
```

モデルにレコードが1つも内場合に`nil`を返す。例外は発生しない。

返すレコードの最大値を指定することもできる。

```ruby
clients = Client.take(2)
```

いったいいつ使うのか。

### first

```ruby
client = Client.first
```

同等のクエリ。

```sql
SELECT * FROM clients ORDER BY clients.id ASC LIMIT 1
```

モデルにレコードが1つも内場合に`nil`を返す。例外は発生しない。

## last

firstの反対。主キーの順序にしたがって最後のレコードを返す。

```ruby
client = Client.last
```

同等のクエリ。

```sql
SELECT * FROM clients ORDER BY clients.id DESC LIMIT 1
```

モデルにレコードが1つも内場合に`nil`を返す。例外は発生しない。

### find_by

与えられた条件にマッチする最初のレコードを返す。

```ruby
Client.find_by first_name:'hoge'
```

これと同じ。

```ruby
Client.where(first_name: 'hoge').take
```
ggkkkkk
SQL

```sql
SELECT * FROM clients WHERE (clients.first_name = 'hoge') LIMIT 1
```

### find_each, find_in_batches

```ruby
# このコードはテーブルが大きい場合、メモリを大量に消費する可能性あり
User.all.each do |user|
  NewsMailer.weekly(user).deliver_now
end
```

多くのレコードにたいして反復処理をしたいとき。`User.all.each`みたいに書かないほうがいい。

これだとActive Recordにたいして、テーブル全体を一度に取り出し、1行ごとにオブジェクトを生成する。そして巨大なモデルオブジェクトの配列をメモリに配置する。膨大な数のレコードにたいしてこのようなコードを実行すると、配列全体のサイズがメモリ容量を上回るだろう。

そのため、Railsはメモリを圧迫しないサイズにバッチを分割して処理する2つの方法を提供している。

- find_each
  - レコードのバッチを1つ取り出し、次に各レコードを1つのモデルとして個別にブロックにyieldする
- find_in_batches
  - レコードのバッチを1つ取り出し、バッチ全体をモデルの配列としてブロックにyieldする

### find_each

複数のレコードを一括で取り出し、続いて各レコードを1つのブロックにyieldする。

```ruby
User.find_each do |user|
  NewsMailer.weekly(user).deliver_now
end
```

対象を絞ることもできる。

```ruby
User.where(weekly_subscriber: true).find_each do |user|
  NewsMailer.weekly(user).deliver_now
end
```
`weekly_subscriber`がtrueの人だけにメールを送るバッチ処理。ただしこれは順序指定がない場合に限る。

オプションいろいろ。

`batch_size`オプションは、ブロックに個別に渡される前に1回のバッチで取り出すレコード数を指定する。1回に5000件ずつ処理したい場合はこちら。

```ruby
User.find_each(batch_size: 5000) do |user|
  NewsMailer.weekly(user).deliver_now
end
```

たとえば主キーが2000番以降のユーザーにたいしてメールを送る場合。`start`が使える。

```ruby
User.find_each(start: 2000) do |user|
  NewsMailer.weekly(user).deliver_now
end
```

終わりを指定したい場合は`finish`が使える。


```ruby
User.find_each(start: 2000, finish: 10000) do |user|
  NewsMailer.weekly(user).deliver_now
end
```

### find_in_batches

`find_each`との違いは、`find_in_batches`はバッチを個別にではなくモデルの配列としてブロックにyieldするところ。

```ruby
# 1回あたりadd_invoicesに納品書1000通の配列を渡す
Invoice.find_in_batches do |invoices|
  export.add_invoices(invoices)
end
```

オプションは`find_each`と同じオプションが使える。

## 条件

### where

条件は、文字列、配列、ハッシュのいずれかの方法で与えることができる。ただし、条件を文字列だけで構成すると、SQLインジェクション脆弱性が発生する可能性がある。

数値が変動する場合、このように書く。

```ruby
Client.where("orders_count = ?", params[:orders])
```

?（疑問符）に対応するのが`params[:orders]`

ちなみにこんな書き方は絶対にダメ。

```ruby
Client.where("orders_count = #{params[:orders]}")
```

変数を直接置くと、DBにそのまま渡される。悪意ある人物がエスケープされていない危険な変数を仕込むことができてしまう。

`キー/値`のハッシュを渡すことができる。

```ruby
Client.where("created_at >= :start_date AND created_at <= :end_date",
  {start_date: params[:start_date], end_date: params[:end_date]})
```

疑問符ではなく、`:start_date`を渡して引数で該当する変数をセットする。こちらのほうが可読性が高い。

### ハッシュを使用した条件

可読性が高い。

```ruby
Client.where(locked: true)
```

こんなふうに範囲を指定することもできる。これは実務で使いそうな感じある。

```ruby
Client.where(created_at: (Time.now.midnight - 1.day)..Time.now.midnight)
```

昨日作成されたすべてのクライアントを検索している。


