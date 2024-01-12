# kkentzo.github.io

## Setup development

Do not use a `.ruby-gemset` file on MacOS. `nokogiri` fails to
compile. Need to fix...

We need `asdf` and the `ruby` plugin to be installed in the system
(`asdf plugin add ruby`).

Then:

    $ asdf install
    $ gem install jekyll bundler
    $ bundle install

## Run local server

    $ bundle exec jekyll serve [--drafts]

## View jekyll docs locally

    $ bundle exec jekyll docs

## Theme

This site uses the [whiteglass](https://github.com/yous/whiteglass)
theme. Visit the page for more info.
