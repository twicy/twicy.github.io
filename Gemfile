source "https://rubygems.org"

# GitHub Pages' gems do not support Ruby 4 yet (commonmarker requires Ruby < 4). On Ruby 4,
# Bundler would silently fall back to a 2022 github-pages that crashes, so fail fast instead.
ruby ">= 3.0", "< 4.0"

# Same Jekyll, kramdown and Rouge versions GitHub Pages builds with
# (https://pages.github.com/versions/), so a local preview matches the live site.
gem "github-pages", group: :jekyll_plugins
gem "webrick" # needed by `jekyll serve` on Ruby 3
