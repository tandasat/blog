# Blog

Set up and build instructions.
```
$ sudo apt-get install ruby-full build-essential zlib1g-dev
$ sudo gem install jekyll bundler
$ sudo chown -R $USER:$USER /var/lib/gems/
$ cd blog
$ sudo bundle install
$ bundle exec jekyll serve --force_polling
or
$ bundle exec jekyll serve --force_polling --drafts --future
```
