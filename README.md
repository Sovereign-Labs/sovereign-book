## Viewing the Book

The Sovereign SDK book is built using `mdbook`. To view its contents:

1. `cargo install mdbook`
2. `mdbook serve --open`

You can find full documentation for `mdbook` [here](https://rust-lang.github.io/mdBook/).

## Editing the Book

To edit an existing chapter, simply modify the corresponding Markdown file in the `src` directory.

To change the book structure, modify `SUMMARY.md`.

To edit the default theme, modify `theme/variables.css`.

## Deploying

To deploy this book, you just need to update https://github.com/Sovereign-Labs/sovereign-labs.github.io. In the
following instructions, we assume that repository is checked out and that it shares a parent with this one:
```ascii
# The directory structure should look like this
parent/
  sovereign-book/
  sovereign-labs.github.io/
```
1. (In this repository) `mdbook build`
2. `cp -r ./book/* ../sovereign-labs.github.io/`
3. `cd ../sovereign-labs.github.io/`
4. `git add .`
5. `git status` (To review the changes)
6. `git commit -m "Update book"`
7. `git push`
