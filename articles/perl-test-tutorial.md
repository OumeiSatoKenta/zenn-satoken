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

## 参考にしたページ

- [Test::Tutorial](https://perldoc.jp/docs/modules/Test-Simple-0.47/Tutorial.pod)

## 詰まったところで解決したメモ

### Date::ICalのモジュールのインストール

-　`cpan Date::ICal`で失敗する 
-  `cpanm Date::ICal`でインストールできる。root 権限を求められたり、--force をつけるようメッセージが出たら適宜つける。
