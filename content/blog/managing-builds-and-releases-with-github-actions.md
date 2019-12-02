+++
author = "Ricardo Rocha"
date = "2019-12-02T18:00:00+00:00"
description = ""
draft = true
tags = ["dev", "git", "github", "github actions"]
title = "Managing Builds and Releases with GitHub Actions"
+++

I've been a longtime user of Travis CI, specifically for a [pet project](https://github.com/ezgliding/goigc)
managing gliding flights parsing and analysis - that in the past had iterations in
Java and Python and has for several years been written in Go.

With recent talk about GitHub Actions and also seeing that a couple of
new projects in the containers area seem to be using it, i decided to give this
a go. The first results are pretty good and in this post i'll describe what's
available and how to use this functionality to:
* Build cross platform Go binaries, plus test and coverage
* Build and publish Docker containers into registries
* Automate GitHub release creation using tags, including generation of changelogs

## Workflow

GitHub Actions allow the creation of workflows with each step automating some
custom process. These can be building and packaging software, deploying to some
external service, generating code quality reports, among many others.
They are stored under `.github/workflows` and defined in yaml, and you can
have as many as you want.

Workflows are based on events such as pushing new code, new branches or tags to a
repository, opening or closing pull requests, commenting on a PR, etc. In my
example i want to trigger for all branch updates, tags starting with `v` - my
version tag convention - and i want to skip the workflow for doc file changes.
{{< highlight yaml >}}
on:
  push:
    branches:
      - '*'
    tags:
      - 'v*'
    paths-ignore:
      - 'CONTRIBUTING.md'
      - 'README.md'
      - 'docs/**'
{{< /highlight >}}

One of the best features of GitHub Actions is the [Marketplace](https://github.com/marketplace?type=actions),
where you can find reusable components for most common tasks. An example is
the [GH Release](https://github.com/marketplace/actions/gh-release) action to
automate the creation of GitHub releases everytime a new tag is pushed.
```yaml
  - name: Release cross platform binaries
    uses: softprops/action-gh-release@v1
    if: startsWith(github.ref, 'refs/tags/v')
    with:
      files: |
        _dist/**
      body_path: CHANGELOG.md
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

This step relies on [softprops/action-gh-release@v1](https://github.com/marketplace/actions/gh-release)
and i pass it the list of assets that go with it (the previously
built binaries for each platform) as well as the generated changelog. I found
this simpler than relying on the official [create-release](https://github.com/marketplace/actions/create-a-release)
action which requires a [separate step](https://github.com/actions/upload-release-asset)
to upload assets.

Notice that this step has a condition so that it is only triggered for version
tags, and ignored for other tags as well as branch updates.

The build step it depends on relies on the wonderful [gox](https://github.com/mitchellh/gox)
tool which does cross compilation for multiple platforms. Another option would
be to rely on Github Actions's [matrix strategy](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions#jobsjob_idstrategy) to truly run the builds on multiple platforms, but this is rarely needed when
building Go binaries.
```yaml
  - name: Make build for cross platform binaries
    run: |
      make build-cross
```

## Makefile Actions

In the last step above we simply call a make target. The decision to keep most
of the build steps in a Makefile while there are equivalent GitHub Actions is
to still be able to replicate these steps offline. Wrapping them as individual
steps in the workflow is trivial.

Check the full Makefile [here](https://github.com/ezgliding/goigc/blob/master/Makefile),
heavily based on [this one](https://github.com/helm/helm/blob/master/Makefile)
from the Helm project. Targets include `build`, `build-cross`, `test`, `test-coverage`,
`changelog`, `docker` and `docker-push`. Here's the `build-cross` target from
above.
```bash
.PHONY: build-cross
build-cross: LDFLAGS += -extldflags "-static"
build-cross: $(GOX)
	GO111MODULE=on CGO_ENABLED=0 $(GOX) -parallel=3 \
      -output="_dist/{{.OS}}-{{.Arch}}/$(BINNAME)_{{.OS}}_{{.Arch}}" \
      -osarch='$(TARGETS)' $(GOFLAGS) -tags '$(TAGS)' -ldflags '$(LDFLAGS)' \
      ./cmd/goigc
```

## Generating Changelog

The goal is to generate the changelog before creating the release, and include
the result in the body text. I spent some time choosing an option for this,
there are again [multiple GitHub Actions](https://github.com/marketplace?utf8=%E2%9C%93&type=actions&query=changelog)
for this task.

In the end i still relied on the [github_changelog_generator](https://github.com/github-changelog-generator/github-changelog-generator)
with some built-in logic to track the previous tag, here's the Makefile target.
```bash
changelog:
	@./scripts/changelog.sh
```
```bash
cat scripts/changelog
...
github_changelog_generator -u ezgliding -p goigc --since-tag $(git describe --abbrev=0 --tags `git rev-list --tags --skip=1 --max-count=1`)
```

## Visualization

The great integration with the rest of GitHub is also very useful.

{{< figure src="/images/blog/github-actions-goigc.png"
    title="" width="100%" >}}

You can easily [browse the workflows](https://github.com/ezgliding/goigc/actions), 
check logs and retrieve any artifacts from the different steps.

It's been a positive transition and i don't think i'll move back.
