This is the source code for my blog on [musigma.blog](https://musigma.blog/).

## Installation on macOS
0. Install bundler: `gem install bundler`
0. Install jekyll: `bundle install --path vendor/bundle`
0. Test the site by executing: `bundle exec jekyll serve`

## Installation on GNU/Linux
0. Install dependencies: `sudo pacman -Si ruby nodejs`
0. Install more dependencies: `gem install bundler && bundle install --path vendor/bundle`
0. Test the site by executing: `bundle exec jekyll serve`

## Deployment
0. Build site: `bundle exec jekyll build`
0. Deploy site: `tar -czf - -C _site . | ssh musigma.blog 'tar -xzf -'`
