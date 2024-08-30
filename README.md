# pages-mimic-github-flavored-markdown

This repo aims to make the markdown rendered from GitHub Pages looks as close as possible to that rendered by GitHub.

## Dependency versions

| Dependency                                                                       | Version |
| -------------------------------------------------------------------------------- | ------- |
| [pages-gem](https://github.com/github/pages-gem)                                 | 232     |
| [jekyll-commonmark-ghpages](https://github.com/github/jekyll-commonmark-ghpages) | 0.5.1   |
| [github-markdown-css](https://github.com/sindresorhus/github-markdown-css)       | 5.6.1   |
| [anchor-js](https://github.com/bryanbraun/anchorjs)                              | 5.0.0   |

## GitHub Pages and GitHub process markdown files differently

There are mainly two reasons: 1. Different markdown processors. 2. Different CSS styling.

### Markdown converter

From
https://github.com/github/pages-gem/blob/v232/lib/github-pages/configuration.rb#L135-L151:

```ruby
# Ensure we're using Kramdown or GFM.  Force to Kramdown if
# neither of these.
#
# This can get called multiply on the same config, so try to
# be idempotentish.
def restrict_and_config_markdown_processor(config)
  config["markdown"] = "kramdown" unless \
    %w(kramdown gfm commonmarkghpages).include?(config["markdown"].to_s.downcase)

  return unless config["markdown"].to_s.casecmp("gfm").zero?

  config["markdown"] = "CommonMarkGhPages"
  config["commonmark"] = {
    "extensions" => %w(table strikethrough autolink tagfilter),
    "options" => %w(unsafe footnotes),
  }
end
```

GitHub Pages supports `kramdown` and `GFM` as markdown processor, where `GFM` is just `CommonMarkGhPages`. Although `GFM` outputs HTML closer to GitHub, it still behaves differently in some cases. For example, alerts are not supported by `GFM`.

GitHub uses [markup](https://github.com/github/markup) to determine which markup library to use to render a content file. However, this only includes the initial step of rendering and the rest steps are done on GitHub.com, which means that the whole rendering pipeline is not open-sourced. To render the markdown file exactly like GitHub, one can use [GitHub API](https://docs.github.com/en/rest/markdown/markdown?apiVersion=2022-11-28#render-a-markdown-document).

## Prior knowledge

If `markdown: GFM` is set in `_config.yml`, it will be replaced by
`CommonMarkGhPages`
([jekyll-commonmark-ghpages](https://github.com/github/jekyll-commonmark-ghpages)),
the CommonMark generator for Jekyll used by GitHub Pages.

From
https://github.com/github/jekyll-commonmark-ghpages/blob/v0.5.1/lib/jekyll-commonmark-ghpages.rb#L27-L57:

```ruby
def code_block(node)
  lang = if node.fence_info && !node.fence_info.empty?
           node.fence_info.split(/[\s,]/)[0]
         end

  content = node.string_content

  if lang && lexer = ::Rouge::Lexer.find_fancy(lang, content)
    block do
      out("<div class=\"language-#{lang} highlighter-rouge\">")
      out(::Rouge::Formatters::HTMLLegacy.new(css_class: 'highlight').format(lexer.lex(content)))
      out("</div>")
    end
    return
  end

  if option_enabled?(:GITHUB_PRE_LANG)
    out("<pre#{sourcepos(node)}")
    out(' lang="', lang, '"') if lang
    out('><code>')
  else
    out("<pre#{sourcepos(node)}><code")
    if lang
      out(' class="language-', lang, '">')
    else
      out('>')
    end
  end
  out(escape_html(content))
  out('</code></pre>')
end
```

[Rouge](https://github.com/rouge-ruby/rouge), a pure Ruby syntax highlighter, is
used for syntax highlighting. Its HTML output is compatible with stylesheets
designed for [Pygments](https://github.com/pygments/pygments), a generic syntax
highlighter written in Python. CSS files created from Pygments built-in styles
can be found [here](https://github.com/richleland/pygments-css).

In additional,
[github-markdown-css](https://github.com/sindresorhus/github-markdown-css) is
needed to replicate the GitHub Markdown style.
