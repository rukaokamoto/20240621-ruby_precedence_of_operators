---
marp: true
# dark theme
class: invert
---
<!-- headingDivider: 1 -->

#  Rubyの演算子の優先度について

# Author

rukaokamoto

![height:200](profile.jpg)
github

![height:200](qr_github.png)

# License

CC BY-NC-SA 4.0

[![CC BY-NC-SA 4.0](https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-nc-sa/4.0/)

Copyright (C) 2024 rukaokamoto

# About this contents

GitHub Pages: [https://rukaokamoto.github.io/20240621-ruby_precedence_of_operators/](https://rukaokamoto.github.io/20240621-ruby_precedence_of_operators)

Repository: [https://github.com/rukaokamoto/20240621-ruby_precedence_of_operators - GitHub](https://github.com/rukaokamoto/20240621-ruby_precedence_of_operators)

![height:200](qr_github_pages.png)


# どのようなレコードが返ってくるか？

```rb
Model.where(status_code: -8888 || 1...Float::INFINITY)
```

# 期待していた取得できるレコード

- status_codeが-8888または1以上のレコード
- status_codeが-8888のレコード

```rb
Model.where(status_code: -8888 || 1...Float::INFINITY)
```

# 実際に取得されたレコード

-8888以上のレコード

```rb
Model.where(status_code: -8888 || 1...Float::INFINITY)
```

# このコードを見た時の感想_1

```-8888 || 1...Float::INFINITY``` で ```-8888``` がtrueなので-8888しか取得出来てないのでは？

# このコードを見た時の感想_2

Rubyだと以下のように式が評価されるため。

```rb
irb(main):001> a = 1
=> 1
irb(main):002> b = 2
=> 2
irb(main):003> p a || b
1
=> 1
irb(main):004> a = nil
=> nil
irb(main):005> p a || b
2
=> 2
```

# 修正内容

```rb
Model.where("status_code = ? OR status_code >= ?", -8888, 1)
```

# 現状のコードと比較

-8888しか取得出来なかったものから1以上も取得出来るようになったので、レコード数は増えてる筈なのに何故か減ってる。

```rb
irb(main):014> Model.where(status_code: -8888 || 1...Float::INFINITY).count
=> 88888
irb(main):016> Model.where("status_code = ? OR status_code >= ?", -8888, 1).count
=> 8888
```

# 取得出来ているstatus_codeを確認する

何故か-888が混じってる。

```rb
irb(main):026> Model.where(status_code: -8888 || 1...Float::INFINITY).pluck(:status_code).uniq
=> [-8888, 8888, -888]
irb(main):027> Model.where("status_code = ? OR status_code >= ?", -8888, 1).pluck(:allocation_st
atus).uniq
=> [-8888, 8888]
```

# ここでやっと発行されているSQLを確認

```rb
[3] pry(main)> Model.where(status_code: -8888 || 1...Float::INFINITY).to_sql
=> "SELECT `allocations`.* FROM `allocations` WHERE `allocations`.`status_code` >= -8888"
[4] pry(main)> Model.where("status_code = ? OR status_code >= ?", -8888, 1).to_sql
=> "SELECT `allocations`.* FROM `allocations` WHERE (status_code = -8888 OR status_code >= 1)"
```

# わかったこと

元々のコードは-8888以上のレコードを取得している。
修正したものは正しかったが、元々のコードが-8888しか取れてないと思い込んでいることがそもそも間違いだった。

# なぜそうなっているのか

```rb
-8888 || 1...Float::INFINITY
```
↓以下のように評価されていた
```
(-8888 || 1)...Float::INFINITY
```
だから、```-8888...Float::INFINITY``` -8888以上になっていた。

# 演算子の優先順位について

通常、```||```の方が```...```よりも優先度が高い。

もし、```...```の方が優先度が高い場合、つまり```-8888 || (1...Float::INFINITY)```の場合
この場合は、最初に予想していたstatus_codeが-8888しか取得されていないという予想が真実になる。

# 結論

to_sqlメソッドでActiveRecordで発行されるSQLは毎回確認しましょう。