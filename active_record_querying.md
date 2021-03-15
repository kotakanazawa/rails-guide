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

TODO: 調べる「yieldするとは？」
