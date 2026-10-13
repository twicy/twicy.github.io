# About Me

Hi, I am Yuchen Tang. I am currently a Member of Technical Staff on the Kernel Team, working on the high-performance FlashBlade and FlashArray storage systems. Before that, I worked for two years at Huawei as an Operating System Engineer, and then completed my M.S. in Computer Engineering at Stony Brook University. My interest lies in building solutions for **next generation data centers**, from multiple perspectives, including memory, networks and storage.

Website: <https://twicy.github.io>

This is barely a simple website, and only tells little about myself. Feel free to reach out to me through email: <twicy@umich.edu> and <garretttang@outlook.com>

## Local preview

The `Gemfile` pins the same Jekyll, kramdown and Rouge versions that GitHub Pages uses, so a local build matches the live site. It needs Ruby 3.x (GitHub Pages builds with 3.3; its gems do not install on Ruby 4):

```bash
sudo snap install ruby --classic --channel=3.3/stable   # or: sudo snap refresh ruby --channel=3.3/stable
bundle install
bundle exec jekyll serve   # then open http://localhost:4000
```

If you switch Ruby versions, delete `Gemfile.lock` before `bundle install`.

## Writing posts

- Add a file named `_posts/YYYY-MM-DD-name.md`; it is published at `/blogs/name.html`, listed on the Blogs page, and included in the RSS feed (`/feed.xml`) and sitemap automatically. Minimal front matter:

  ```yaml
  ---
  title: My Post
  topic: GPU and CUDA   # one of `blog_topics` in _config.yml (sets the Blogs page section)
  ---
  ```

  Add `permalink: /blogs/some/path.html` to choose a different URL. Avoid renaming a post's file once it is published: feed readers identify posts by file name and would show it again as new.
- Posts have no automatic title (start the post with `# Title`) and a giscus comment section. Add `comments: false` to the front matter to hide comments.
- Put a post's images in `assets/images/posts/<name>/` (the post file name without the date) and link them with absolute paths, e.g. `![alt](/assets/images/posts/sgemm-p1/sgemm-tile.jpg)`, so they work on the site, in the feed, and in an editor preview. Keep photos at most ~1600 px on the long side.

## Repository layout

```
_config.yml       site settings, navigation, blog topics
index.md          home page (About Me)
_pages/           Projects, Publications, Services, Blogs index (URLs set by `permalink`)
_posts/           blog posts, YYYY-MM-DD-name.md
_layouts/         page templates
_includes/        head, navigation, sidebar, footer, MathJax, comments, icons
assets/
  css/  fonts/    stylesheet and self-hosted fonts
  images/site/            avatar, favicons
  images/projects/        figures for the Projects page
  images/posts/<name>/    one folder per post
  notebooks/              notebooks used to make figures
files/            CV (kept at /files/ so shared links keep working)
```

File names are lowercase with hyphens and no spaces; GitHub Pages is case-sensitive (`.JPEG` ≠ `.jpg`) even when your computer is not.
- Math: `$...$` is inline, `$$...$$` on its own lines is a displayed equation (a blank line around it is not required). MathJax is only loaded on pages that contain math.
- kramdown processes Markdown escapes inside `$...$` before MathJax sees it, so `\{`, `\}` and `\|` lose their backslash and `\\` becomes `\`. Use `\lbrace`, `\rbrace`, `\Vert`, or put the formula in `$$...$$` (kramdown leaves `$$` content untouched). A bare `#` is shown as a literal `#`.

## Credit

The design is not original. Thanks to:

1. [GitHub Pages](https://pages.github.com/) and [Jekyll](https://jekyllrb.com/)
2. [Minimal Mistakes](https://github.com/mmistakes/minimal-mistakes) by Michael Rose (MIT), the theme this site's look comes from
3. [GuangLun2000](https://github.com/GuangLun2000/GuangLun2000.github.io) (Hanlin Cai), whose adaptation of that theme this site started from
4. [PT Serif and PT Sans Narrow](https://www.paratype.com/public/) by ParaType (SIL Open Font License, see `assets/fonts/`)
5. Icons from [Bootstrap Icons](https://icons.getbootstrap.com/) (MIT) and [Simple Icons](https://simpleicons.org/) (CC0); syntax colors from GitHub's [Primer](https://primer.style/) (MIT)
6. [MathJax](https://www.mathjax.org/) for math and [giscus](https://giscus.app/) for comments
