---
layout: post
title: "Fast and easy Go binaries delivery"
---

I have several little apps made in Go, which I deliver as binaries.

~1 year ago, I made a [shell script](https://github.com/goreleaser/old-go-releaser)
to automate the `go build` for some OSs and archs, create a `tar.gz` archive
and create a github release.

But, I also wanted to release them to a homebrew tap repo. I also wanted
to customize some stuff in some apps.

To improve the entire thing, I created a new app called
[goreleaser](https://github.com/goreleaser/releaser).

It has a pipeline that:

- build the binaries for the platforms you want
- archive them with other files you want in `tar.gz` package
- creates a github release with a good description and log
- updates/creates a homebrew recipe in a tap of your choosing

All you need to do is to create a `goreleaser.yml` in you repository
root and call goreleaser on your CI.

#### Simplest possible `goreleaser.yml`:

```yaml
repo: user/repo
binary_name: my-binary
```

#### `travis.yml` example:

```yaml
language: go
after_success:
  test ! -z "$TRAVIS_TAG" && curl -s https://raw.githubusercontent.com/goreleaser/get/master/latest | bash
```

## More options

An example with all custom options possible:

```yaml
repo: user/repo
binary_name: my-binary
files:
  - LICENSE.txt
  - README.md
  - CHANGELOG.md
build:
  main: ./cmd/main.go
  oses:
    - darwin
    - freebsd
  arches:
    - amd64
brew:
  repo: user/homebrew-formulae
  caveats: "Caveats section to add to the formulae"
```

The code of course is on [github](https://github.com/goreleaser/releaser),
as well as a README with all the info you need.

## How a release looks like?

Pretty much like this:

[![gorelease example release](/public/images/goreleaser-release-example.png)](https://github.com/goreleaser/releaser/releases/tag/v0.1.4)

And here is an example of [homebrew tap formulae](https://github.com/getantibody/homebrew-antibody/blob/master/antibody.rb)

## In the wild

- [goreleaser releases itself](https://github.com/goreleaser/releaser/releases)
- [antibody](https://github.com/getantibody/antibody/releases)
- [all my go binary apps](https://github.com/caarlos0/homebrew-formulae)

## Final words

Well, it is basically a weekend project that I will continue to maintain,
as it automates a lot of my work.

Hope you all enjoy it!
