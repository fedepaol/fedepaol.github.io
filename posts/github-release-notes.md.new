---
date: 2024-04-06T18:31:52+02:00
title: "Creating release notes for your GitHub Project"
description: "How to create automated release notes for GitHub projects"
categories: ["Automation", "github", "metallb"]
---

# TL;DR

The [kubernetes release notes generator](https://github.com/kubernetes/release/tree/master/cmd/release-notes) is really nice. I managed to use it without Prow with MetalLB, by setting a proper [pull request template](https://github.com/metallb/metallb/blob/main/.github/pull_request_template.md) and a [Github Action workflow](https://github.com/metallb/metallb/blob/main/.github/workflows/classify.yaml) that adds the proper label and validates the PR body.

Finally, a [glue bash script](https://github.com/metallb/metallb/blob/main/website/gen_relnotes.sh) wraps the call to the release notes generator.

# A bit of context

Since I started maintaining MetalLB in 2021, invested significantly in automating various processes. This was made possible with the help
of a vibrant community of contributors that helped in [moving from circleci to github actions](https://github.com/metallb/metallb/pull/1105), [signing the images with cosign](https://github.com/metallb/metallb/pull/1437) and many others.

One thing that was left out was crafting a proper release notes list. 

The manual process was long, tedious and error prone. The releaser (me, mostly) would have to go over all the commits between the last released version and the new version to be released, add a note for each line and classify whether it was a bug or a feature (plus retrieving the corresponding pull request and / or feature).

I must admit that having to craft the release notes for the MetalLB project was the biggest reason for procrastinating a release. And the more I was waiting, the more things I had to release! This, until I began seeking alternatives.

## Enforcing release notes to the PR authors

The first iteration of this mechanism was to add a Github Action job that would fail if the PR author [wasn't touching the release notes file](https://github.com/metallb/metallb/pull/2191). Although this was simple, it served its purpouse **with an exception**: this requirement caused conflicts between pull requests, basically blocking us to use the really powerful github merge queues mechanism that allows to merge (compatible) PRs in parallel.

## Trying to mimic Kubernetes

After giving up with the idea of reinventing the wheel and writing my tool for generating the release notes, I started looking for existing solutions, and I found the [kubernetes release notes generator](https://github.com/kubernetes/release/tree/master/cmd/release-notes). The behaviour is pretty simple: it takes a range of commits, a github token, and the tool will use the github APIs to harvest the content of the pull requests looking for informations about the various changes.

```bash
$ export GITHUB_TOKEN=a_github_api_token
$ release-notes \
  --start-sha 02dc3d713dd7f945a8b6f7ef3e008f3d29c2d549 \
  --end-sha   23649560c060ad6cd82da8da42302f8f7e38cf1e
```

And it generates an output like

```raw

### New Features

- Add a field to the FRRConfiguration CRD to disable MP BGP for the given peer (#128, @AlinaSecret)

### Bug fixes

- Fix the case where merging an FRRConfiguration with no hold / keepalive / connect Time set with one where the time is set to default fails. (#120, @fedepaol)
```

(this is actually taken from the release notes of [frr-k8s](https://raw.githubusercontent.com/metallb/frr-k8s/main/RELEASE_NOTES.md), a MetalLB spin off project).

## What does the tool need

In order to generate the release notes out of a github repository, the tool relies on two things:

- A classification of the various PRs, under the form of labels `kind/something`
- A release-note section with the comment for this PR, similar to what [I added in MetalLB](https://raw.githubusercontent.com/metallb/metallb/main/.github/pull_request_template.md)

### The PR template

The first step to enable the tool is to provide a PR template that contains the instructions to fill the description of the change for the release notes:

```raw
**Release note**:
<!--  Write your release note:
1. Enter your extended release note in the below block. If the PR requires additional action from users switching to the new release, include the string "action required".
2. Follow the instructions for writing a release note from k8s: https://git.k8s.io/community/contributors/guide/release-notes.md
3. If no release note is required, just write "NONE".
-->

``````release-note

```

And then the hard part is convince the contributors not to delete the wall of text but actually read and fill it :P .

### The labels

I wanted to enforce the presence of the labels, and to have them automatically added.
Kubernetes and the other projects of the k8s.io organization can rely on Prow (the kubernetes CI/CD system) doing that, based on the other section of the pull request template that asks the contributor to pick
a kind:

```markdown
**Is this a BUG FIX or a FEATURE ?**:

> Uncomment only one, leave it on its own line:
>
> /kind bug
> /kind cleanup
> /kind feature
> /kind design
> /kind flake
> /kind failing
> /kind documentation
> /kind regression

```

So within kubernetes, a contributor uncomments the right kind (or kinds) and the labels are added to the PR automagically.

Unfortunately, without Prow this does not have any effect on the labels, so I had to be a bit creative and I came up with a solution based on [github actions](https://github.com/metallb/metallb/blob/main/.github/workflows/classify.yaml).

I will comment here each step of the workflow:


For each creation update of the Pull Request:

```yaml
name: Add Labels

on:
  pull_request_target:
    types: [opened, edited]
```

It looks for the uncommented kind:

```
      - uses: actions-ecosystem/action-regex-match@v2
        id: regex-match
        with:
          text: ${{ github.event.pull_request.body }}
          regex: '(?<!> )\/kind (\w+)'
```

Validates the kind to be one of the allowed ones:

```
      - name: Check
        run: |
          if [[ ! "${{ steps.regex-match.outputs.group1 }}" =~ ^(api-change|bug|cleanup|deprecation|design|documentation|failing|feature|flake|regression)$ ]]; then
            echo "kind must belong to
                  - api-change \
                  - bug \
                  - cleanup \
                  - deprecation \
                  - design \
                  - documentation \
                  - failing \
                  - feature \
                  - flake \
                  - regression \
            please add /kind [type] to the body of the PR"
            exit 1
          fi
```

and add the label:

```
      - uses: actions-ecosystem/action-add-labels@v1
        with:
          labels: kind/${{ steps.regex-match.outputs.group1 }}
```

Finally, it ensures the user did not delete the release notes section:

```
      - name: Check release notes were not deleted
        uses: actions/github-script@v7
        with:
          script: |
            const pr = await github.rest.pulls.get({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number
            });
            if (!pr.data.body.includes("```release-note")) {
              throw new Error("Please don't cancel the ```release-note ``` section as it is used to build the release notes");
            }
```

## Invoking the tool

Now that the Pull Requests contain all the informations required by the release notes generator, we can invoke the tool to generate the release notes:

```bash
GOFLAGS=-mod=mod go run k8s.io/release/cmd/release-notes@v0.16.5 \
    --branch $branch \
    --required-author "" \
    --org metallb \
    --dependencies=false \
    --repo metallb \
    --start-sha $from \
    --end-sha $to \
    --output $release_notes
```

At the moment, the process of copying the generated release notes to the actual markdown files containing them is done manually, but I don't exclude some automation will be handle that part as well.

## Conclusion

Automation is the key for maintaining a project healthy. Manual operations don't scale well, and I try to keep the bar high for any type of
manual activity done in the projects I help with. Although sporadic, the activity of generating the release notes manually was really painful, and
having a way to generate them automatically using (almost) existing tools will allow other maintainers to be able to generate them without feeling
that pain.

Here, I showed how by mixing the kubernetes generator tool and a bit of github actions I am now able to generate the release notes without much effort.

