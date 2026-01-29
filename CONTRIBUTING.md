# Contributor Guidelines

This repository follows the usual Github workflow of pull request to make changes to the repository. Once you create a pull request against this repository you will see preview of your changes deployed by Netlify. If you prefer to check changes locally then follow the [local development](#local-development) guidelines in this document.

## Local Development

### Prerequisites

- Make sure You have installed:
  - GO [here](https://go.dev/doc/install) and include it in PATH
  - NodeJS [here](https://nodejs.org/en/download/package-manager) and include it in PATH
  - PostCSS plugin [here](https://gohugo.io/functions/css/postcss/#setup)
- Install Hugo by following the instructions [here](https://gohugo.io/installation/).

> [!NOTE]
> Currently recommended versions are:
> - hugo: **0.155.0+** 
> - docsy: **0.13.0+**

### Using Hugo

Start a local server from the root of this repository:

```bash
hugo server
```

### Using Docker

1. Create a cache directory for Hugo:
    ```bash
    mkdir -p $HOME/.cache/hugo_cache
    ```

2. Start a local server from the root of this repository:
    ```bash
    docker run --rm -v .:/site -v $HOME/.cache/hugo_cache:/cache -u $(id -u):$(id -g) -w /site -p 1313:1313 ghcr.io/gohugoio/hugo:latest server --bind="0.0.0.0"
    ```

### Using Docker Compose

1. Install docker-compose by following the instructions [here](https://docs.docker.com/compose/install/).

2. Export variables for user and group id:

    ```bash
    export UID=$(id -u)
    export GID=$(id -g)
    ```

3. Create a cache directory for Hugo:

   To prevent ownership and permission problems, create the Hugo cache directory and ignore the error if the directory
   already exists:

    ```bash
    mkdir -p $HOME/.cache/hugo_cache
    ```

4. Start a local server from the root of this repository:

    ```bash
    docker-compose up -d
    ```

Now go to [http://localhost:1313](http://localhost:1313) on your browser. Once you make any changes to the code the
changes are compiled in real-time and you can see those in the browser.

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
