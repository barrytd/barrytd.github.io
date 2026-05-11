# barrytd.github.io

Personal security-lab blog. Built with Jekyll and the [minima](https://github.com/jekyll/minima) theme, hosted free on GitHub Pages.

## Run locally

```bash
bundle install
bundle exec jekyll serve --livereload
```

Then open http://127.0.0.1:4000/.

If you have not installed Ruby yet, follow the [Jekyll Windows install guide](https://jekyllrb.com/docs/installation/windows/) (use the Ruby+Devkit installer and run `ridk install` afterwards).

## Add a new post

1. Create a file in `_posts/` named `YYYY-MM-DD-slug.md`.
2. Copy the front matter from any existing post in that folder.
3. Drop screenshots into `assets/images/<slug>/`.
4. Reference them with `![alt text]({{ "/assets/images/<slug>/01.png" | relative_url }})`.
5. Commit and push — GitHub Pages rebuilds automatically.

## House style

- **No flag values, no room answers, no cracked password values.** If a room requires retrieving a value, describe how to get it, not what it is. Use *(redacted)* placeholders.
- Use the same plain-English-parenthetical style as the lab READMEs: technical term, then a short non-jargon explanation in parentheses.
- One concept per post section, with a screenshot if it adds context (no flag-revealing screenshots).
- Close every post with a *Key Takeaways* list.

## Deployment

This repo is named `barrytd.github.io`, which means GitHub treats it as a user site and publishes the `main` branch at <https://barrytd.github.io/> automatically. No `gh-pages` branch, no Actions config required.
