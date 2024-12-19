# Contributor Guidelines

This repository follows the usual Github workflow of pull request to make changes to the repository. Once you create a pull request against this repository you will see preview of your changes deployed by Netlify. If you prefer to check changes locally then follow the [local development](#local-development) guidelines in this document.

## Local Development

### Using Hugo

- Make sure You have installed:
  - GO [here](https://go.dev/doc/install) and include it in PATH
  - NodeJS [here](https://nodejs.org/en/download/package-manager) and include it in PATH
  - PostCSS plugin [here](https://gohugo.io/functions/css/postcss/#setup)
- Install Hugo by following the instructions [here](https://gohugo.io/installation/).

- Start a local server from the root of this repository:
  ```bash
  hugo server
  ```

### Using Docker Compose

- Install docker-compose by following the instructions [here](https://docs.docker.com/compose/install/).

- Start a local server from the root of this repository:

```bash
docker-compose up -d
```

Now go to [http://localhost:1313](http://localhost:1313) on your browser. Once you make any changes to the code the changes are compiled in real-time and you can see those in the browser.

## Writing a blog

- Create a folder with the current year under `content/en/blog` if it does not exists.
- Create a file of markdown format under this path: `content/en/blog/202x` named appropriately.
- At the top of the file create metadata section to look like this:

```yaml
---
date: 202x-xx-xx
title: Title of the blog
linkTitle: Title for the link
description: >
  First line desc
  second line desc
author: Alpha Beta ([@nan](https://twitter.com/nan))
---
```

- Write the blog in general markdown.
- And create a PR to this repository. You can also view your changes locally before creating a PR using the instructions mentioned in [local development](#local-development) section.
- Read more about organizing blogs [here](https://www.docsy.dev/docs/adding-content/content/#organizing-your-blog-posts).
- To see all the different elements supported by the docsy theme for blogs, see this [example blog](https://example.docsy.dev/blog/2018/10/06/second-blog-post/) and its associated [source code](https://github.com/google/docsy-example/blob/main/content/en/blog/news/second-post.md).
