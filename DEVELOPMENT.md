- [Development instructions](#development-instructions)
  * [Setup](#setup)
  * [Run locally](#run-locally)
  * [On a Mac](#on-a-mac)
    + [Sed](#sed)
    + [File Descriptors in "server mode"](#file-descriptors-in--server-mode-)
- [How it is deployed](#how-it-is-deployed)


# Development instructions

## Setup

1. Clone this repo (or your fork) using `--recurse-submodules`, like so:

   ```shell
   git clone --recurse-submodules https://github.com/knative/website.git
   ```

   If you accidentally cloned this repo without `--recurse-submodules`, you'll
   need to do the following inside the repo:

   ```shell
   git submodule init
   git submodule update
   cd themes/docsy
   git submodule init
   git submodule update
   ```

   (Docsy uses two submodules, but those don't use further submodules.)

1. Clone the docs repo next to (_not inside_) the website repo. This allows you
   to test docs changes alongside the website:

   ```shell
   git clone https://github.com/knative/docs.git
   ```

   You may also want to clone the community repo:

   ```shell
   git clone https://github.com/knative/community.git
   ```

1. (Optional) If you want to change the CSS, install
   [PostCSS](https://www.docsy.dev/docs/getting-started/#install-postcss)

1. Install a supported version of [Hugo](https://www.docsy.dev/docs/getting-started/#install-hugo).

## Run locally

You can use `./scripts/localbuild.sh` to build and test files locally.
The script uses Hugo's build and server commands in addition to some Knative
specific file scripts that enables optimal user experience in GitHub
(ie. use README.md files, allows our site to use relative linking
(not
[`rel` or `relref`](https://gohugo.io/content-management/cross-references/#use-ref-and-relref)),
etc.) and also meets Hugo/Docsy static site generator
and template requirements (ie. _index.hmtl files, etc.)

The two local docs build options:

- Simple/static HTML file generation for viewing how your Markdown renders in HTML:

  Use this to generate a static build of the documentation site into HTML. This
  uses Hugo's build command [`hugo`](https://gohugo.io/commands/hugo/).

  From your clone of knative/website, you run `./scripts/localbuild.sh`.

  All of the HTML files are built into the `public/` folder from where you can open,
  view, and test how each file renders.

  Notes:

  - This method does not mirror how knative.dev is generated and therefore is
    only recommend to for testing how your files render. That also means that link
    checking might not be 100% accurate. Hugo builds relative links differently
    (all links based on the site root vs relative to the file in which the link
    resides - this is part of the Knative specific file processing that is done)
    therefore some links will not work between the statically built HTML files.
    For example, links like `.../index.html` are converted to `.../` for simplicity
    (servers treat them as the same destination) but when you are browsing a local HTML
    file you need to open/click on the individual `index.html` files to get where you want
    to go.
  - This method does however make it easier to read or use local tools on the HTML build
    output files (vs. fetching the HTML from the server). For example, it is useful for
    refactoring/moving content and ensuring that complicated Markdown renders properly.
  - Using this method also avoids the MacOs specific issue (see below), where the default
    open FD limit exceeds the total number of `inotify` calls that Hugo wants to keep open.

- Mimic how knative.dev is built and hosted:

  Use this option to locally build knative.dev. This uses Hugo's local server
  command [`hugo server`](https://gohugo.io/commands/hugo_server/).

  From your clone of knative/website, you run `./scripts/localbuild.sh -s true`.

  All of the HTML files are temporarily copied into the `content/en/` folder to allow
  the Hugo server to locally host those files at the URL:port specified in your terminal.

  Notes:

  - This method provides the following local build and test build options:
    - test your locally cloned files
    - build and test other user's remote forks (ie. locally build their PRs `./scripts/build.sh -f repofork -b branchname -s)
    - option to build only a specific branch or all branches (and also from any speicifed fork)
    - fully functioning site links
    - [See all command options in localbuild.sh](https://github.com/knative/website/blob/main/scripts/localbuild.sh)
  - Hugo's live-reload is not completely utilized due to the required Knative specific file processing
    scripts (you need to rerun `./scripts/localbuild.sh -s` to rebuild and reprocess any changes that you
    make to the files from within your local knative/docs clone directory).

    Alternatively, if you want to use Hugo's live-reload feature, you can make temporary
    changes to the copied files within the `content/en/` folder, and then when satisfied, you must
    copy those changes into the corresponding files of your knative/docs clone.
  - Files in `content/en/` are overwritten with a new copy of your local files in your knative/docs
    clone folder each time that you run this script. Note that the last set of built files remain
    in `content/en/` for you to run local tools against but are overwritten each time that you rerun the script.
  - Using this method causes the MacOs specific issue (see below), where the default
    open FD limit exceeds the total number of `inotify` calls that the Hugo server wants to keep open.

## On a Mac

If you want to develop on a Mac, you'll find two obstacles:

### Sed

The scripts assume GNU `sed`. You can install this with
[Homebrew](https://brew.sh/):

```shell
brew install gnu-sed
# You need to put it in your PATH before the built-in Mac sed
PATH="/usr/local/opt/gnu-sed/libexec/gnubin:$PATH"
```

### File Descriptors in "server mode"

By default, MacOS permits a very small number of open FDs. This will manifest
as:

```
ERROR 2020/04/14 12:37:16 Error: listen tcp 127.0.0.1:1313: socket: too many open files
```

You can fix this with the following (may be needed for each new shell):

```shell
sudo launchctl limit maxfiles 65535 200000
# Probably only need around 4k FDs, but 64k is defensive...
ulimit -n 65535
sudo sysctl -w kern.maxfiles=100000
sudo sysctl -w kern.maxfilesperproc=65535
```

# How it is deployed

While the above describes how the content is built locally, https://knative.dev/ is built and served by [Netlify](https://netlify.com/) on their platform.
There are a few differences between the two, and when debugging issues on the website, it can be useful to understand what Netlify is doing (as we've
configured it).

Generally, Netlify runs Hugo/Docsy builds and publishes everything that gets merged into the knative/docs and knative/website repos
(anything in knative/community will get picked up when either of the other two repos trigger a build).

The builds are triggered are through [GitHub webhooks](https://docs.github.com/en/developers/webhooks-and-events/webhook-events-and-payloads).
There are two webhooks sent from knative/docs that are configured to inidicate that they were sent from knative/website:

* One that triggers a "production" build - Any PR that gets merged. (Webhook payload - /website `main` branch)
* One that triggers a "preview" build - Any PR action other than a merge (ie. commit, comment, label, etc). (Webhook payload - /website `staging` branch)

All of our builds (and build logs) are shown here: https://app.netlify.com/sites/knative/deploys (in the order of recent to past)

## Cutting a Release

Update API docs
1. Update the API docs in main branch:
https://github.com/knative/docs/blob/master/docs/reference/api/README.md
Use the following command from the root directory of the `docs` repo:
    ```
    cd ./hack/
    KNATIVE_SERVING_COMMIT=v0.21.0 KNATIVE_EVENTING_COMMIT=v0.21.0 ./gen-api-reference-docs.sh
    ```
    Here is an example PR: https://github.com/knative/docs/pull/3276#issue-579549750

2. Create a new knative/docs branch and release tag (the new release): https://github.com/knative/docs/releases


3. Update the knative/website to include the next release.
Here is an example PR: https://github.com/knative/website/pull/256

    The Netlify build automatically publishes everything
