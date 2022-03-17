## Requirements

1. To build and run the blog locally make sure you have the following dependencies installed on the machine:

- [Ruby](https://www.ruby-lang.org/en/) 2.6.0 or higher
- [Jekyll](https://jekyllrb.com/) 4.2.0 or higher
- [Bundler](https://bundler.io/) 2.2.0 or higher

Check out Jekyll's [installation](https://jekyllrb.com/docs/installation/) page to get detailed instructions.

2. Install dependencies:

```
bundle install
```

## Creating a New Post

1. Add a post using the format `yyyy-MM-dd-{title}.md` to `_posts`.
2. Put images to the `images` directory (if there are any).
3. Build the site:

```
bundle exec jekyll build
```

## Running Locally

1. Execute:

```
bundle exec jekyll serve
```

2. Navigate to http://localhost:4000.
