source "https://rubygems.org"

# github-pages gem 이 Jekyll·Liquid·테마 플러그인 버전을 한 묶음으로 고정한다.
# 232 이상으로 유지할 것: 그 아래 묶음은 Liquid 4.0.3 을 가져오고,
# 4.0.3 은 Ruby 3.2 에서 제거된 Object#tainted? 를 호출해 빌드가 깨진다.
# (liquid 4.0.4 에서 해당 호출이 제거됨 → Ruby 3.3 사용 가능)
gem "github-pages", "~> 232", group: :jekyll_plugins
gem "jekyll-include-cache", group: :jekyll_plugins
