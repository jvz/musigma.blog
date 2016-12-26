This is the source code for my blog on [musigma.org](http://musigma.org/).

## Installation on macOS
0. Install rbenv: `brew install rbenv`
0. Enable rbenv oh-my-zsh plugin.
0. Install ruby: `rbenv install 2.3.3`
0. Refresh: `rbenv rehash`
0. Install bundler: `gem install bundler`
0. Refresh again: `rbenv rehash`
0. Install jekyll: `bundle install`
0. Test the site by executing: `bundle exec jekyll serve`
0. New posts go in the `_posts` directory.
0. New posts follow the naming convention `YYYY-MM-DD-slug.md`
0. The date for a post can be inserted in vim by running: `:r !date "+date: \%F \%T \%z"`

## Installation on GNU/Linux
0. Install dependencies: `sudo pacman -Si ruby nodejs`
0. Install more dependencies: `gem install bundler && bundle install`
0. Don't forget to add `~/.gem/ruby/2.3.0/bin` to your `PATH`
0. Sit and ponder how `bundle install` put `jekyll` in `/usr/bin` without prompting me.
0. Recall that you're probably using `NOPASSWD` in `sudo`, so `bundle` probably invoked it.
0. Use `jekyll serve` to preview site. Or, use `bundle exec jekyll serve` as on macOS.
