source "https://rubygems.org"

# Use the github-pages gem so the local build matches GitHub Pages' build
# environment exactly (it pins Jekyll and the allowed plugins).
gem "github-pages", group: :jekyll_plugins

# Plugins used by the site and the Minimal Mistakes theme.
group :jekyll_plugins do
  gem "jekyll-include-cache"
end

# Windows and JRuby do not include zoneinfo files, so bundle the tzinfo-data gem.
gem "tzinfo-data", platforms: [:mingw, :mswin, :x64_mingw, :jruby]

# Ruby 3.0+ no longer ships webrick in the stdlib; Jekyll's local server needs it.
gem "webrick", "~> 1.8"
