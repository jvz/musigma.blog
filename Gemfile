#source 'https://rubygems.org'
#
# FIXME: using the unversioned 'json' gem here uses json 2.x while jekyll requires json 1.x
require 'json'
require 'open-uri'
versions = JSON.parse(open('https://pages.github.com/versions.json').read)

gem 'github-pages', versions['github-pages']

#gem 'github-pages', '134'
