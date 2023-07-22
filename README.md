# Blog

Set up and build instructions.
```
$ sudo apt-get install ruby-full build-essential zlib1g-dev
$ sudo gem install jekyll bundler
$ sudo bundle install
$ cd blog
$ bundle exec jekyll serve --force_polling
or
$ bundle exec jekyll serve --force_polling --drafts
```
