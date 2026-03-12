source "https://rubygems.org"

# GitHub Pages 환경과 로컬 환경 모두 호환
gem "jekyll", "~> 4.3"

# 필수 플러그인
gem "jekyll-feed"
gem "jekyll-sitemap"
gem "jekyll-paginate"
gem "kramdown-parser-gfm"
gem "rouge"

# GitHub Actions에서 필요
gem "webrick"          # Ruby 3.x에서 기본 제거됨 → 명시 필요

# Windows/JRuby 전용 (로컬 개발 시)
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end
