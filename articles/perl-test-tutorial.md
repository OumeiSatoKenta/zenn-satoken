---
title: "Perl Test::Tutorialをやってみた。"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## 始めた理由

- perlのユニットテストを書くにあたり、基礎を押さえておきたいため。

## スペック、バージョン

- macOS Monterery
- perl バージョン

```sh
$ perl -v

This is perl 5, version 30, subversion 3 (v5.30.3) built for darwin-thread-multi-2level
(with 2 registered patches, see perl -V for more detail)
```

## 注意

- この記事で公開しているところまでが動作確認した範囲になります。
- 他の作業での不具合は考慮してません。例）cpanm に --forceをつけることなど
- 自分はスクリプトに下の2行のコードを追加しています。非効率なデバッグ作業を防ぐため、つけておきたい。

```perl
use strict;
use warnings;
```

## 参考にしたページ

- [Test::Tutorial](https://perldoc.jp/docs/modules/Test-Simple-0.47/Tutorial.pod)

## 詰まったところで解決したメモ

### Date::ICalのモジュールのインストール

-　`cpan Date::ICal`で失敗する 
-  `cpanm Date::ICal`でインストールできる。root 権限を求められたり、--force をつけるようメッセージが出たら適宜つける。

- 
Global symbol "$ical" requires explicit package name (did you forget to declare "my $ical"?) at manual.pl line 7.

### マニュアルのテスト で失敗する
- `$ical = Date::ICal ... `のところで下のようなエラ−メッセージが出る

```
$ perl manual.pl
1..8
Global symbol "$ical" requires explicit package name (did you forget to declare "my $ical"?) at manual.pl line 7.
Global symbol "$ical" requires explicit package name (did you forget to declare "my $ical"?) at manual.pl line 9.
Execution of manual.pl aborted due to compilation errors.
# Looks like your test exited with 255 before it could output anything.
```
- `my $ical = Date::ICal ... `とすることでエラーが出なくなる
- これは`use strict; use warnings;`をつけていたため。
- 以降の内容では、このエラーについては言及しない。


### "plan tests => keys %ICal_Dates * 8;" でエラーが出る

- 下のエラーが出る
```
$ perl many_values.pl
Experimental keys on scalar is now forbidden at many_values.pl line 23.
Type of arg 1 to keys must be hash or array (not multiplication (*)) at many_values.pl line 23, near "8;"
Execution of many_values.pl aborted due to compilation errors.
```

- 下のように変数にハッシュのサイズを入れるようにするとエラーが出なくなる

```perl
my $ICal_Dates_Size =  keys(%ICal_Dates);
plan tests => $ICal_Dates_Size * 8;
```

### マニュアルのテストで、icalの設定がうまくいかない
- Date::ICalは元々hourでエラーになるようにわざと実装されているらしいが、dayでも失敗する
- テストケースによってはdayでこけないものもある。
- 現状理由がわからない

```
1..32
ok 1 - new(ical => '18990505T232323' )
ok 2 -   and it's the right class
ok 3 -  year()
ok 4 -  month()
ok 5 -  day()
not ok 6 -  hour()
#   Failed test ' hour()'
#   at many_values.pl line 36.
#          got: '14'
#     expected: '23'
ok 7 -  min()
ok 8 -  sec()
ok 9 - new(ical => '19971024T120000' )
ok 10 -   and it's the right class
ok 11 -  year()
ok 12 -  month()
ok 13 -  day()
not ok 14 -  hour()
#   Failed test ' hour()'
#   at many_values.pl line 36.
#          got: '3'
#     expected: '12'
ok 15 -  min()
ok 16 -  sec()
ok 17 - new(ical => '19671225T000000' )
ok 18 -   and it's the right class
ok 19 -  year()
ok 20 -  month()
not ok 21 -  day()
#   Failed test ' day()'
#   at many_values.pl line 35.
#          got: '24'
#     expected: '25'
not ok 22 -  hour()
#   Failed test ' hour()'
#   at many_values.pl line 36.
#          got: '15'
#     expected: '0'
ok 23 -  min()
ok 24 -  sec()
ok 25 - new(ical => '20390123T232832' )
ok 26 -   and it's the right class
ok 27 -  year()
ok 28 -  month()
ok 29 -  day()
not ok 30 -  hour()
#   Failed test ' hour()'
#   at many_values.pl line 36.
#          got: '14'
#     expected: '23'
ok 31 -  min()
ok 32 -  sec()
# Looks like you failed 5 tests of 32.
```

### 「テストをスキップする」で、epochの設定がうまくいかない
- `my $t1 = Date::ICal->new( epoch => 0 );`のところの設定がうまくいってない。
- 現状理由がわからない。
- 自分のコードの実行結果を載せておく。
```
1..7
not ok 1 - Epoch time of 0
#   Failed test 'Epoch time of 0'
#   at skip.pl line 16.
#          got: '1653320528'
#     expected: '0'
not ok 2 - epoch to ical
#   Failed test 'epoch to ical'
#   at skip.pl line 19.
#          got: '20220523T154208Z'
#     expected: '19700101Z'
not ok 3 -  year()
#   Failed test ' year()'
#   at skip.pl line 21.
#          got: '2022'
#     expected: '1970'
not ok 4 -  month()
#   Failed test ' month()'
#   at skip.pl line 22.
#          got: '5'
#     expected: '1'
not ok 5 -  day()
#   Failed test ' day()'
#   at skip.pl line 23.
#          got: '23'
#     expected: '1'
ok 6 - Start of epoch in ICal notation
not ok 7 -  and back to ICal
#   Failed test ' and back to ICal'
#   at skip.pl line 29.
#          got: '3155760000'
#     expected: '0'
# Looks like you failed 6 tests of 7.

[satoukenta:~/workspace/perl_sample/Test/tutorial][main]$ perl skip.pl
1..7
1653320547
not ok 1 - Epoch time of 0
#   Failed test 'Epoch time of 0'
#   at skip.pl line 16.
#          got: '3155760000'
#     expected: '0'
ok 2 - epoch to ical
ok 3 -  year()
ok 4 -  month()
ok 5 -  day()
ok 6 - Start of epoch in ICal notation
not ok 7 -  and back to ICal
#   Failed test ' and back to ICal'
#   at skip.pl line 29.
#          got: '3155760000'
#     expected: '0'
# Looks like you failed 2 tests of 7.
```

### 「テストをスキップする」の条件がうまく働かない
- `if $^O eq 'MacOS';`がうまく働かず、スキップせずにテストが進んでしまう。
- 引数か何か？
- スクリプト実行時に引数を渡すように実装してみた。

```perl
my @nums = @ARGV;

SKIP: {
  skip('epoc to ICal not working on MacOS', 6) if @nums[0] eq 'MacOS';
  ,,,
```

- 下のようにちゃんとスキップするようになった。
```
$ perl skip.pl 'MacOS'
ok 8 # skip epoc to ICal not working on MacOS
ok 9 # skip epoc to ICal not working on MacOS
ok 10 # skip epoc to ICal not working on MacOS
ok 11 # skip epoc to ICal not working on MacOS
ok 12 # skip epoc to ICal not working on MacOS
ok 13 # skip epoc to ICal not working on MacOS
# Looks like you planned 14 tests but ran 13.
# Looks like you failed 2 tests of 13 run.
```