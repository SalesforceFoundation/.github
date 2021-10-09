---
author: James Estevez
title: Customizing PR workflows with github-script
---

In this document we'll walk through an example workflow step by step to
illustrate how your team can use the `github-script` library to
customize your pull request workflows.

# Codeowners pull request workflow

Let's start with a descriptive name. These map to the sidebar in the
actions pane so it pays to be descriptive. We recommend something like
"Team Name Review Workflows".[1] Here we have a variation on that theme
with:

``` yaml
name: Shared SFDO Codeowners Review Workflows
```

## What should trigger the workflow?

[`on`](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#on)
is a key part of the workflow definition. The [full
list](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows)
is available in the documentation, but here we're concerned with two
types of events on pull requests:

1.  When a PR has a specific label applied, and
2.  when a review has been requested.

``` yaml
on:
  pull_request:
    types:
      - labeled
      - review_requested
```

Other `pull_request` event types of note:

| Event names                              | trigger                |
|------------------------------------------|------------------------|
| `opened`, `reopened`, `closed`           | PR state changes       |
| `ready_for_review`, `converted_to_draft` | Draft PR state changes |
| `synchronize`                            | Updates and merges     |

We also add `workflow_call`. This marks the workflow as available for
reuse across the GitHub organization:

``` yaml
workflow_call:
```

There are other applications besides our CODEOWNERS use case, such as
scrum teams working across multiple repositories, or using workflows
from base packages downstream in extension packages. You can read more
about reusable workflows
[here](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows).

## Environment Variables

You can set environment variables at the workflow, job, or step levels.
It is possible to override higher level definitions, for example by
defining a variable at the job level to override a workflow variable
while that job is executing.

In this case, we're using them as constants to reuse variables across
jobs, and to make future updates easier. The main text block is the body
of a comment we want to add to any PR that triggers codeowners review:

``` markdown
Release Engineering asks that teams use the following process for routine reviews:

1. After creating a non-draft pull request that includes automation updates, a release engineer will be auto-assigned to the PR.
2. When dev review is complete and the PR is ready for the release engineer to review, add a "ready for RE review" label to the PR to let us know when the PR is ready for us to review.
3. If you've added the "ready for RE review" label but haven't received a review within a 36 hours, @-mention the assigned RE in a comment on the PR.
4. If you don't receive a response from the assigned RE by the end of the next business day (or your request is urgent), post a message to #sfdo-releng-support that includes a link to this PR and one of us will review as soon as we're able.
```

We can use YAML's `|` operator to demarcate a text block:

``` yaml
env:
  CODEOWNERS_TEAM: 'release-engineering-reviewers'
  READY_FOR_REVIEW_COMMENT: |
    This PR has been labeled as ready for Release Engineering review by
  CODEOWNERS_COMMENT: |
    Release Engineering asks that teams use the following process for routine reviews:

    1. After creating a non-draft pull request that includes automation updates, a release engineer will be auto-assigned to the PR.
    2. When dev review is complete and the PR is ready for the release engineer to review, add a "ready for RE review" label to the PR to let us know when the PR is ready for us to review.
    3. If you've added the "ready for RE review" label but haven't received a review within a 36 hours, @-mention the assigned RE in a comment on the PR.
    4. If you don't receive a response from the assigned RE by the end of the next business day (or your request is urgent), post a message to #sfdo-releng-support that includes a link to this PR and one of us will review as soon as we're able.
```

## Job and Steps

This section forms the core of our workflow's business logic. Recall
that we have two scenarios we need to handle, so we'll split them into
two conditionally executed steps. Begin with the job definition:

``` yaml
jobs:
  codeowners_workflow_comments:
    runs-on: ubuntu-latest
    steps:
```

Again, we try to pick a meaningful name. We also specifiy the container
in which our workflow will execute, `ubuntu-latest`. Use this unless you
have a specific use case for another runner.

Select a `name` and `id`. The `id` is needed if you want to refer to the
output of the step in another step.

``` yaml
- name: CODEOWNERS Review Requested
  id: codeowners-comment
```

### Conditional step execution

We defined when the *workflow* should run using `on`, but when should
*this* step run?

The [`github`
context](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context)
contains some information about the workflow run, and `github.event`
contains the webhook payload ([pull request JSON
schema](https://github.com/octokit/webhooks/blob/2f77c8a109369c318b577b356df596eecb7d4667/payload-schemas/api.github.com/pull_request/review_requested.schema.json)).
We use these values in an
[expression](https://docs.github.com/en/actions/learn-github-actions/expressions)
to only run the step when two conditions are met.

For the `codeowners-comment` step, we use the webhook payload to only
execute the step when a review is requested from the team we've defined
in the `CODEOWNERS_TEAM` environment variable:

``` yaml
if: github.event.action == 'review_requested' && !github.event.pull_request.draft &&
    github.event.requested_team.slug == env.CODEOWNERS_TEAM
```

We'll want our second step, `codeowners-labeled-ready`, to use slightly
different logic:

``` yaml
if: github.event.action == 'labeled' && !github.event.pull_request.draft &&
    contains(github.event.pull_request.labels.*.name, 'ready for RE review')
```

Note that we're using the [`contains`
function](https://docs.github.com/en/actions/learn-github-actions/expressions#contains)
and the [`*` object
filter](https://docs.github.com/en/actions/learn-github-actions/expressions#object-filters)
syntax.

## github-script

The [`github-script`
action](https://github.com/SalesforceFoundation/github-script#actionsgithub-script)
allows us to easily write scripts against the GitHub REST and GraphQL
APIs in Javascript.[2]

### Javascript

The first of the two scripts uses
[createComment](https://octokit.github.io/rest.js/v18#issues-create-comment)
to insert the markdown comment we've set as an environment variable
above.

We begin by storing the value of our `codeowners_comment` environment
variable in a constant called `comment_body`, demonstrating how we can
access environment variables via `process.env`:

``` javascript
const comment_body = process.env.CODEOWNERS_COMMENT;
const comment_greeting = "Hi 👋";
```

We also want to prevent the posting of
[duplicate](https://github.com/SalesforceFoundation/Grants-Management/pull/897#issuecomment-937141993)
[comments](https://github.com/SalesforceFoundation/Grants-Management/pull/897#issuecomment-938195335).
There's more than one way to do this, so we'll pick the complicated way
(The simpler approach uses octokit's
[`listComments`](https://octokit.github.io/rest.js/v18#issues-list-comments).)
as a demonstration of how to use a GraphQL query:

``` javascript
const query = `query ($owner:String!, $name:String!,$number:Int!,) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $number) {
      comments(last: 100) {
        nodes {
          body
          author {
            login
          }
        }
      }
    }
  }
}`;
```

We can access the PR number, GitHub organization slug, and the name of
the repository via `context`. We can also use the webhook payload (same
as `github.event` above!) directly via `context.payload`. We use these
variables and run the query:

``` javascript
const variables = {
  owner: context.repo.owner,
  name: context.repo.repo,
  number: context.payload.pull_request.number,
};
const result = await github.graphql(query, variables);
```

Next, iterate over the query result and check whether any the
github-actions bot has authored any comments that begin with our
`comment_greeting`:

``` javascript
const no_existing_comment = result.repository.pullRequest.comments.nodes.every(
  (comment) => {
    comment.author.login == "github-actions" &&
      !comment.body.startsWith(comment_body);
  }
);
```

Finally, if no existing comments are found, use a template literal to
combine a mention of the PR author with the body of our comment:

``` javascript
if (no_existing_comment) {
  github.issues.createComment({
    issue_number: context.issue.number,
    owner: context.repo.owner,
    repo: context.repo.repo,
    body: `${comment_greeting} @${context.payload.pull_request.user.login}! ${comment_body}`,
  });
}
```

Our review comment script adds a bit more logic:

``` javascript
const comment_body = process.env.READY_FOR_REVIEW_COMMENT
let mention_text = '';

let requested_reviewers = context.payload.pull_request.requested_reviewers
  .map((reviewer) => reviewer.login)

if (requested_reviewers.length >= 1) {
  let reviewer_mentions = requested_reviewers
    .map((login) => `@${login}`)
    .join(', ');
    mention_text = `Reviews have been requested from: ${reviewer_mentions}.`
}

github.issues.createComment({
  issue_number: context.issue.number,
  owner: context.repo.owner,
  repo: context.repo.repo,
  body: `${comment_body} @${context.payload.sender.login}. ${mention_text}`,
});
```

The main difference is that we're iterating over the list of the
currently assigned reviewers to extract their usernames so that they can
be mentioned in the body of the comment.

We're really only scratching the surface of what the action allows with
these examples.

### Invoking our script

So, now that we've got our scripts, how do we use them? We have two
options: inline or script. Since they're short enough, we safely define
our scripts inline. As we did for the [environment
variables](id:e2938afc-a18f-4bd1-a2e2-35a208d9fe73), we use the same `|`
operator to define a literal block:

``` yaml
uses: SalesforceFoundation/github-script@v4
with:
  script: |
    const comment_body = process.env.CODEOWNERS_COMMENT;
    const comment_greeting = "Hi 👋";
    const query = `query ($owner:String!, $name:String!,$number:Int!,) {
      repository(owner: $owner, name: $name) {
        pullRequest(number: $number) {
          comments(last: 100) {
            nodes {
              body
              author {
                login
              }
            }
          }
        }
      }
    }`;
    const variables = {
      owner: context.repo.owner,
      name: context.repo.repo,
      number: context.payload.pull_request.number,
    };
    const result = await github.graphql(query, variables);
    const no_existing_comment = result.repository.pullRequest.comments.nodes.every(
      (comment) => {
        comment.author.login == "github-actions" &&
          !comment.body.startsWith(comment_body);
      }
    );
    if (no_existing_comment) {
      github.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `${comment_greeting} @${context.payload.pull_request.user.login}! ${comment_body}`,
      });
    }
```

(For details on how to invoke a separate file see the
[README](https://github.com/SalesforceFoundation/github-script#run-a-separate-file).)

We can now define our second step, `codeowners-labeled-ready`:

``` yaml
- name: Labeled ready for CODEOWNERS Review
  id: codeowners-labeled-ready
  if: github.event.action == 'labeled' && !github.event.pull_request.draft &&
      contains(github.event.pull_request.labels.*.name, 'ready for RE review')
  uses: SalesforceFoundation/github-script@v4
  with:
    script: |
      const comment_body = process.env.READY_FOR_REVIEW_COMMENT
      let mention_text = '';

      let requested_reviewers = context.payload.pull_request.requested_reviewers
        .map((reviewer) => reviewer.login)

      if (requested_reviewers.length >= 1) {
        let reviewer_mentions = requested_reviewers
          .map((login) => `@${login}`)
          .join(', ');
          mention_text = `Reviews have been requested from: ${reviewer_mentions}.`
      }

      github.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `${comment_body} @${context.payload.sender.login}. ${mention_text}`,
      });
```

That's it! 🎉

# Invoking a shared workflow

Now that we defined a workflow, reusing it is relatively simple. In
total:

``` yaml
name: Call a reusable workflow

on:
  pull_request:
    types:
      - labeled
      - review_requested

jobs:
  shared-codeowners:
    uses: SalesforceFoundation/.github/.github/workflows/codeowners.yml@main
```

See the
[docs](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows)
for details.

# Making a workflow available as a template

The last bit is mostly of interest to release engineers and automation
champions that want to make a workflow available in every
`SalesforceFoundation` repository. This is accomplished by creating a
[workflow
template](https://docs.github.com/en/actions/learn-github-actions/creating-workflow-templates)
by adding our new [shared
workflow](id:064df6e9-c5fd-4633-ba26-7e9120e96ca9) and the following
JSON file:

``` json
{
    "name": "SFDO Codeowners Review Workflows",
    "description": "Housekeeping scripts for automated pull request workflows.",
    "iconName": "sfdo",
    "filePatterns": [
        "CODEOWNERS$"
    ]
}
```

Pretty simple.

# Where to get help

1.  Review:
    1.  the official
        [documentation](https://docs.github.com/en/actions/learn-github-actions)
    2.  Support Community [Github Actions
        topic](https://github.community/c/code-to-cloud/github-actions/)
2.  Review existing workflows like NPSP's [compliance
    workflow](https://github.com/SalesforceFoundation/NPSP/blob/1a784fef84ac88053578a3ec7afaa8fc5cf9acf9/.github/workflows/compliance.yml)
3.  Ask SFDO Release Engineering:
    1.  If it's simple: #sfdo-releng-support
    2.  If it's complex: Open a WI from the [RelEng Service
        Catalog](https://confluence.internal.salesforce.com/display/SFDOIDP/Catalog+of+Services#CatalogofServices-GitHubActionsEnablement)

[1] RelEng note: Prefix shared workflow names to make them easier to
identify reusable workflows.

[2] Why? In short, because our policy excludes third party actions.
