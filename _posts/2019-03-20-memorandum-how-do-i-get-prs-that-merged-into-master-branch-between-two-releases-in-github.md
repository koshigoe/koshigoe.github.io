---
layout: post
title:  "Memorandum: How do I get PRs that merged into master branch between two releases in GitHub."
date:   2019-03-20 23:00:00 +09:00
categories:
- Memorandum
- GitHub
---


NOTE: In the strict sense, "between two date times".

For example, if you'd like to see PRs that merged into `master` branch between `2019-03-13T00:00:00Z` and `2019-03-20T00:00:00Z`.

### By GitHub Web

Open `https://github.com/:user/:repository/pulls`, then input following phrase into filter box:

```
base:master merged:2019-03-13T00:00:00Z..2019-03-20T00:00:00Z
```

### By GitHub API

Request Search API (`https://api.github.com/search/issues`) with query like following:

```
q=repo:user/repository+is:pr+base:master+merged:2019-03-13T00:00:00Z..2019-03-20T00:00:00Z
```

### How do I search with tag name instead of date time?

I don't know...

### USE CASE: Auto fill release note into GitHub Release

The [GitHub Actions](https://developer.github.com/actions/) helps us to automate workflows. So, I have automated write release note into GitHub Release.

1. create a GitHub Release (only input tag name and title)
1. a GitHub Action will be triggered by [`release` event](https://developer.github.com/v3/activity/events/types/#releaseevent)
    1. the GitHub Action makes release note and write that into GitHub Release

NOTE: If you are interested in this use case, I can show [my GitHub Actions](https://github.com/koshigoe/koshigoe_gem_playground/blob/72e29607b6fa21196b1ab67595d7e2e3050f1498/.github/action/write_github_release/entrypoint.sh) in [my playground repository](https://github.com/koshigoe/koshigoe_gem_playground).

### Ref

- [About searching on GitHub](https://help.github.com/en/articles/about-searching-on-github)
- [Search issues and pull requests](https://developer.github.com/v3/search/#search-issues-and-pull-requests)
