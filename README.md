This is the source code for my blog on [musigma.blog](https://musigma.blog/).

## Installation on macOS
0. Install bundler: `gem install bundler`
0. Install jekyll: `bundle install`
0. Test the site by executing: `bundle exec jekyll serve`
0. New posts go in the `_posts` directory.
0. New posts follow the naming convention `YYYY-MM-DD-slug.md`
0. The date for a post can be inserted in vim by running: `:r !date "+date: \%F \%T \%z"`

## Installation on GNU/Linux
0. Install dependencies: `sudo pacman -Si ruby nodejs`
0. Install more dependencies: `gem install bundler && bundle install`
0. Don't forget to add `~/.gem/ruby/*/bin` to your `PATH`
0. Sit and ponder how `bundle install` put `jekyll` in `/usr/bin` without prompting me.
0. Recall that you're probably using `NOPASSWD` in `sudo`, so `bundle` probably invoked it.
0. Use `jekyll serve` to preview site. Or, use `bundle exec jekyll serve` as on macOS.

## Deployment
0. Build site: `bundle exec jekyll build`
0. Deploy site: `scp -r _site/* musigma.blog:`
0. Determine if an automated pipeline is worth it?
