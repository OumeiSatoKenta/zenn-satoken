---
title: "rails new をするとOpenSSL::Cipher::CipherErrorが出る。"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## OS, ruby version等
- OS: macOS Montery
- Ruby等
```
$ ruby --version
ruby 2.6.8p205 (2021-07-07 revision 67951) [universal.x86_64-darwin21]
$ which ruby
/usr/bin/ruby
$ rails -v
Rails 6.0.4
$ which rails
/usr/bin/rails
$ gem --version
3.0.3.1
$ which gem
/usr/bin/gem
```

```
$ sudo rails _6.0.4_ new hello_app
/Library/Ruby/Gems/2.6.0/gems/activesupport-6.0.4/lib/active_support/message_encryptor.rb:173:in `auth_data=': couldn't set additional authenticated data (OpenSSL::Cipher::CipherError)
        from /Library/Ruby/Gems/2.6.0/gems/activesupport-6.0.4/lib/active_support/message_encryptor.rb:173:in `_encrypt'
        from /Library/Ruby/Gems/2.6.0/gems/activesupport-6.0.4/lib/active_support/message_encryptor.rb:151:in `encrypt_and_sign'
        from /Library/Ruby/Gems/2.6.0/gems/activesupport-6.0.4/lib/active_support/encrypted_file.rb:76:in `encrypt'
        from /Library/Ruby/Gems/2.6.0/gems/activesupport-6.0.4/lib/active_support/encrypted_file.rb:50:in `write'
        from /Library/Ruby/Gems/2.6.0/gems/activesupport-6.0.4/lib/active_support/encrypted_configuration.rb:29:in `write'
        from /Library/Ruby/Gems/2.6.0/gems/railties-6.0.4/lib/rails/generators/rails/credentials/credentials_generator.rb:30:in `add_credentials_file_silently'
        from /Library/Ruby/Gems/2.6.0/gems/railties-6.0.4/lib/rails/generators/rails/app/app_generator.rb:177:in `credentials'
        from /Library/Ruby/Gems/2.6.0/gems/railties-6.0.4/lib/rails/generators/app_base.rb:155:in `build'
        from /Library/Ruby/Gems/2.6.0/gems/railties-6.0.4/lib/rails/generators/rails/app/app_generator.rb:332:in `create_credentials'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/command.rb:27:in `run'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/invocation.rb:127:in `invoke_command'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/invocation.rb:134:in `block in invoke_all'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/invocation.rb:134:in `each'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/invocation.rb:134:in `map'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/invocation.rb:134:in `invoke_all'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/group.rb:232:in `dispatch'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/base.rb:485:in `start'
        from /Library/Ruby/Gems/2.6.0/gems/railties-6.0.4/lib/rails/commands/application/application_command.rb:26:in `perform'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/command.rb:27:in `run'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor/invocation.rb:127:in `invoke_command'
        from /Library/Ruby/Gems/2.6.0/gems/thor-1.2.1/lib/thor.rb:392:in `dispatch'
        from /Library/Ruby/Gems/2.6.0/gems/railties-6.0.4/lib/rails/command/base.rb:69:in `perform'
        from /Library/Ruby/Gems/2.6.0/gems/railties-6.0.4/lib/rails/command.rb:46:in `invoke'
        from /Library/Ruby/Gems/2.6.0/gems/railties-6.0.4/lib/rails/cli.rb:18:in `<top (required)>'
        from /System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/rubygems/core_ext/kernel_require.rb:54:in `require'
        from /System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/2.6.0/rubygems/core_ext/kernel_require.rb:54:in `require'
        from /Library/Ruby/Gems/2.6.0/gems/railties-6.0.4/exe/rails:10:in `<top (required)>'
        from /usr/local/bin/rails:23:in `load'
        from /usr/local/bin/rails:23:in `<main>'
```