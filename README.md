# Swift Journal

A personal collection of Swift notes, organized by topic and readable on GitHub or as a Jekyll site.

- [Browse the topics](index.md)
- [Swift concurrency](topics/concurrency/index.md)

## Add a note

Create a Markdown file in a topic folder, such as `topics/concurrency/my-note.md`. Start it with:

```yaml
---
title: Optionals
---
```

Add a relative link to the note from its topic’s `index.md`. For a new topic, also link its `index.md` from the [home page](index.md).

## Preview locally

With Ruby 3.1 and Bundler installed, run (the GitHub Pages dependency stack can fail on newer Ruby versions):

```sh
bundle install
bundle exec jekyll serve
```

Open the URL printed by Jekyll.

## Publish on GitHub Pages

Push these files to GitHub. In **Settings → Pages**, choose **Deploy from a branch**, select the branch containing the site and **`/ (root)`**, then save. GitHub displays the site URL once the build finishes.
