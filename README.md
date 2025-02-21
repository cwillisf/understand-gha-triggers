# Understanding GitHub Actions triggers

This repository is an attempt to better understand the details of triggering GitHub Actions events.

## How can I ...?

### Quickly set up GHA without reading all of this?

It depends on how many builds you want to run. Do you have a limited GHA budget for the repo in question?

#### Luxury

This is nice if your builds aren't limited: it'll check every pushed commit, as well as the proposed merge for each
pull request. If you push a commit before the workflows finish, it'll cancel the one checking the proposed merge
(since that's no longer the active proposal) but it'll continue the check on the previously-pushed commit.

```yaml
on:
  push:
  pull_request:
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }} @ ${{ github.head_ref || github.sha }}
```

#### Minimal

You might want this if you'd like to limit builds as much as possible but maintain safety around your main branch(es).
This will only check pushes and pull requests that target your main branch. New workflows will cancel old ones for the
same branch or PR.

```yaml
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }} @ ${{ github.head_ref || github.ref_name }}
```

### Run a workflow on push and pull request events without duplicates?

I want to run a build-and-test workflow in what seems like two categories of events:

- A commit is pushed to any branch
- Something changes affecting the merge result of a pull request:
  - A pull request is opened
  - A commit is pushed to the PR branch
  - The PR target branch is updated

Early on, I tried something like this:

```yaml
on:
  push:
  pull_request:
```

Note that `pull_request:` without any filters is equivalent to `pull_request: [opened, reopened, synchronize]`.

This achieves the goal as stated! However, if you push a commit to a branch corresponding to an open PR, the workflow
runs twice: once for the push event and once for the pull request event. Depending on the situation, this could be
fine, but it might be expensive or otherwise undesirable.

The most common advice I've seen is to put a branch filter on the `push` event, like this:

```yaml
on:
  push:
    branches:
      - main
  pull_request:
```

That may be fine or even preferable in many cases, but I want build-and-test feedback on all branches, not just
`main`.

Consulting the information recorded below, here are the events that should trigger the workflow:

User action | Event(s)
--- | ---
Push to a branch | `push`
Open a pull request | `pull_request_target/opened`, `pull_request/opened`
Push to a pull request branch | `push`, `pull_request_target/synchronize`, `pull_request/synchronize`

One interesting subtlety is that the proposed merge commit -- the commit most interesting to build and test -- is only
available in the `github.sha` field of `pull_request` events.

The `github.sha` in the `push` event is the commit that was
pushed, and the `github.sha` in the `pull_request_target` event is the base branch commit.

It could also be interesting to build and test the branch by itself, without merging. That commit is available in the
`github.sha` field of the `push` event, and in the `github.event.after` field of the `pull_request` and
`pull_request_target` events.

If we want to test both, it's easy: we should run the workflow on both `push` events to test the branch as-is, and on
`pull_request` events to test the merge result. In both cases, we want to build and test the commit from the
`github.sha` field, which is normal GHA behavior.

If we only want to test the merge, that's more difficult but still possible. We already established that we need to
run the workflow on `pull_request` events, since that's the only way to get the merge commit. But what do we do about
`push` events? How do we run on `push` to a branch _unless_ that branch is the head of an open PR? The `push` event
doesn't include any information indicating that the commit or branch is the head of a PR.

Concurrency groups to the rescue! A later workflow run can cancel an earlier one if their concurrency groups match.
This only happens with queued workflows by default, but if `cancel-in-progress` is set to `true`, it will also cancel
workflows that are already running.

There are two convenient ways to do that. One way is to use `github.sha` from the `push` event and
`github.event.after` from the `pull_request` event, but that feels a bit like lying: the `pull_request` event will
"win" but isn't actually running on the commit named in the concurrency group. The other way is to use the branch
name: that's `github.ref_name` (not `github.ref`!) in the `push` event and `github.head_ref` in the `pull_request` event.

All things considered, I have two recommendations for this scenario:

1. You can run the workflow on "either" `push` or `pull_request` events by running on both and using something like
   `${{ github.workflow }} @ ${{ github.head_ref || github.ref_name }}` for your concurrency group. That'll force a
   PR's proposed merge commit to take precedence. Optionally, filter `push`, `pull_request`, or both events to
   specific branches.
2. Consider actually running on both events, since they're not actually testing the same thing: the `push` event tests
   the branch by itself, and the `pull_request` event tests the result of the proposed merge. In that case, you could
   use something like `${{ github.workflow }} @ ${{ github.ref }}` or `${{ github.workflow }} @ ${{ github.head_ref ||
   github.sha }}` as your concurrency group.

## Generated workflows

Most of the GHA workflows in this repository are generated using the `generate-workflows.sh` script. See the script
for more details.

## Data received by workflows

### Common properties

`github` property | Type | Value
--- | --- | ---
`repository` | string | `"<owner>/<repo>"`
`repository_owner` | string | `"<owner>"`
`repository_visibility` | string | `"public"` or `"private"`
`event_path` | string | `"/home/runner/work/_temp/_github_workflow/event.json"`
`runner.os` | string | Example: `"Linux"`
`runner.arch` | string | Example: `"X64"` (note the capital `X`)
`runner.name` | string | Example: `"GitHub Actions 5"`
`runner.environment` | string | Example: `"github-hosted"`
`runner.tool_cache` | string | Example: `"/opt/hostedtoolcache"`
`runner.temp` | string | Example: `"/home/runner/work/_temp"`
`runner.workspace` | string | Example: `"/home/runner/work/<repo>"`

`github.event` property | Type | Value
--- | --- | ---
`repository` | object | Repository info, including info to construct various URLs
`sender` | object | Info about the GitHub user (or app?) who triggered the event

### Push to `main` branch

#### Push to `main`: `push` event

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/heads/main"`
`sha` | string | Git SHA of the commit that was pushed
`head_ref` | string | `""` (empty string)
`base_ref` | string | `""` (empty string)
`event_name` | string | `"push"`
`ref_name` | string | `"main"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`after` | string | Git SHA of the repo after the push = Git SHA of the commit that was pushed
`before` | string | Git SHA of the repo before the push
`commits` | array | Array of commit objects that were pushed
`compare` | string | URL to compare before and after the push
`forced` | boolean | Whether the push was forced
`head_commit` | object | Commit object of the head commit
`pusher` | object | Pusher object (name, email)
`ref` | string | `"refs/heads/main"`

### Create an issue

#### Create an issue: `issues-opened` event

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/heads/main"`
`sha` | string | Git SHA of the default branch
`head_ref` | string | `""` (empty string)
`base_ref` | string | `""` (empty string)
`event_name` | string | `"issues"`
`ref_name` | string | `"main"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"opened"`
`issue` | object | Issue object

`github.event.issue` property | Type | Value
--- | --- | ---
`body` | string | Body text for the issue
`created_at` | string | Timestamp of issue creation
`number` | number | Issue number
`title` | string | Title of the issue

### Push a new branch

Caused by something like: `git push --set-upstream origin example-branch`

This triggers two events: first a `create`, then a `push`

#### Push a new branch: `create` event

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/heads/example-branch"`
`sha` | string | Git SHA of the new branch
`head_ref` | string | `""` (empty string)
`base_ref` | string | `""` (empty string)
`event_name` | string | `"create"`
`ref_name` | string | `"example-branch"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`description` | null | ?
`master_branch` | string | Example: `"main"`
`pusher_type` | string | `"user"`
`ref` | string | `"example-branch"`
`ref_type` | string | `"branch"`

#### Push a new branch: `push` event

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/heads/example-branch"`
`sha` | string | Git SHA of the commit that was pushed
`head_ref` | string | `""` (empty string)
`base_ref` | string | `""` (empty string)
`event_name` | string | `"push"`
`ref_name` | string | `"example-branch"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`after` | string | Git SHA of the branch after the push = Git SHA of the commit that was pushed
`base_ref` | string | `"refs/heads/main"`
`before` | string | `"0000000000000000000000000000000000000000"`
`commits` | array | `[]`
`created` | boolean | `true`
`deleted` | boolean | `false`
`forced` | boolean | `false`
`head_commit` | object | Commit object of the branch head commit
`pusher` | object | Pusher object (name, email)
`ref` | string | `"refs/heads/example-branch"`

### Open a pull request

In this case, the pull request from `example-branch` to `main` was opened with an assignee and a label, but no
reviewers.

This triggered several workflows, in this order:

1. `pull_request_target` with action `assigned`
2. `pull_request_target` with action `labeled`
3. `pull_request_target` with action `opened`
4. `pull_request_target` (without an action filter)
5. `pull_request` with action `opened`
6. `pull_request` (without an action filter)
7. `pull_request` with action `assigned`
8. `pull_request` with action `labeled`

All of these receive a `github.event` containing the same `pull_request` object:

`github.event.pull_request` property | Type | Value
--- | --- | ---
`_links` | object | Links to various PR-related resources
`additions` | number | Number of additions in the PR
`assignee` | object | User object of the primary assignee
`assignees` | array | Array of user objects of all assignees
`base` | object | Base branch object
`body` | string | Body text of the PR
`changed_files` | number | Number of files changed in the PR
`commits` | number | Number of commits in the PR
`created_at` | string | Timestamp of PR creation
`deletions` | number | Number of deletions in the PR
`draft` | boolean | Whether the PR is a draft
`head` | object | Head branch object
`labels` | array | Array of label objects. Each object includes a name, color, etc.
`number` | number | PR number
`requested_reviewers` | array | Array of user objects of requested reviewers

#### Open a pull request: `pull_request_target` events

All the `pull_request_target` workflows receive these `github` properties:

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/heads/main"`
`sha` | string | Git SHA of `main`
`head_ref` | string | `"example-branch"`
`base_ref` | string | `"main"`
`event_name` | string | `"pull_request_target"`
`ref_name` | string | `"main"`
`ref_type` | string | `"branch"`

The event with `types: [assigned]` receives:

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"assigned"`
`assignee` | object | User object of the assignee
`number` | number | PR number
`pull_request` | object | PR object (see above)

The event with `types: [labeled]` receives:

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"labeled"`
`label` | object | Label object
`number` | number | PR number
`pull_request` | object | PR object (see above)

The event with `types: [opened]` receives:

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"opened"`
`number` | number | PR number
`pull_request` | object | PR object (see above)

The event without an action filter receives the `opened` event.

#### Open a pull request: `pull_request` events

All the `pull_request` workflows receive these `github` properties:

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/pull/2/merge"`
`sha` | string | Git SHA of the (proposed) PR merge commit
`head_ref` | string | `"example-branch"`
`base_ref` | string | `"main"`
`event_name` | string | `"pull_request"`
`ref_name` | string | `"2/merge"`
`ref_type` | string | `"branch"`

The event with `types: [opened]` receives:

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"opened"`
`number` | number | PR number
`pull_request` | object | PR object (see above)

The event without an action filter receives the `opened` event.

The event with `types: [assigned]` receives:

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"assigned"`
`assignee` | object | User object of the assignee
`number` | number | PR number
`pull_request` | object | PR object (see above)

The event with `types: [labeled]` receives:

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"labeled"`
`label` | object | Label object
`number` | number | PR number
`pull_request` | object | PR object (see above)

### Change PR target branch

In this case, the PR was changed from `main` to `another-base`.

This triggered two workflows:

1. `pull_request_target` with action `edited`
2. `pull_request` with action `edited`

#### Change PR target branch: `pull_request_target` event

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/heads/another-base"`
`sha` | string | Git SHA of the new base branch
`head_ref` | string | `"example-branch"`
`base_ref` | string | `"another-base"`
`event_name` | string | `"pull_request_target"`
`ref_name` | string | `"another-base"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"edited"`
`changes` | object | Changes object with a `"base"` property
`number` | number | PR number
`pull_request` | object | PR object (see above)

#### Change PR target branch: `pull_request` event

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/pull/2/merge"`
`sha` | string | Git SHA of the new (proposed) PR merge commit
`head_ref` | string | `"example-branch"`
`base_ref` | string | `"another-base"`
`event_name` | string | `"pull_request"`
`ref_name` | string | `"2/merge"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"edited"`
`changes` | object | Changes object with a `"base"` property
`number` | number | PR number
`pull_request` | object | PR object (see above)

### Push to a pull request branch

In this case, I pushed a commit to `example-branch` while a PR was open, proposing to merge `example-branch` into
`another-base`.

This triggered several workflows, in this order:

1. `push`
2. `pull_request_target` with no action filter
3. `pull_request_target` with action `synchronize`
4. `pull_request` with no action filter
5. `pull_request` with action `synchronize`

All of these receive a `github.event` containing the same `push` object:

#### Push to a pull request branch: `push` event

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/heads/example-branch"`
`sha` | string | Git SHA of the commit that was pushed
`head_ref` | string | `""` (empty string)
`base_ref` | string | `""` (empty string)
`event_name` | string | `"push"`
`ref_name` | string | `"example-branch"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`after` | string | Git SHA of the branch after the push = Git SHA of the commit that was pushed
`base_ref` | string | `null`
`before` | string | Git SHA of the branch before the push
`commits` | array | Array of commit objects that were pushed
`compare` | string | URL to compare before and after the push
`forced` | boolean | Whether the push was forced
`head_commit` | object | Commit object of the head commit
`pusher` | object | Pusher object (name, email)
`ref` | string | `"refs/heads/example-branch"`

#### Push to a pull request branch: `pull_request_target` events

Both `pull_request_target` workflows receive generally the same information:

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/heads/another-base"`
`sha` | string | Git SHA of the base branch (`another-base`)
`head_ref` | string | `"example-branch"`
`base_ref` | string | `"another-base"`
`event_name` | string | `"pull_request_target"`
`ref_name` | string | `"another-base"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"synchronize"`
`after` | string | Git SHA of `example-branch` after the push = Git SHA of the commit that was pushed
`before` | string | Git SHA of `example-branch` before the push
`number` | number | PR number
`pull_request` | object | PR object (see above)

#### Push to a pull request branch: `pull_request` events

Both `pull_request` workflows receive generally the same information:

`github` property | Type | Value
--- | --- | ---
`ref` | string | `"refs/pull/2/merge"`
`sha` | string | Git SHA of the (proposed) PR merge commit
`head_ref` | string | `"example-branch"`
`base_ref` | string | `"another-base"`
`event_name` | string | `"pull_request"`
`ref_name` | string | `"2/merge"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"synchronize"`
`after` | string | Git SHA of the branch after the push = Git SHA of the commit that was pushed
`before` | string | Git SHA of the branch before the push
`number` | number | PR number
`pull_request` | object | PR object (see above)

## Merge a pull request through a merge queue

A merge queue can generate checks on commits other than the head and merge commits of a PR. Behind the scenes, GitHub
creates a temporary branch to hold the merge result, then triggers the `merge_group` event on that branch. Of course,
creating that branch also triggers other events, so depending on how the concurrency groups are set up, these events
can cancel each other. If the `merge_group` event is canceled, that's considered a failure, causing the merge to be
evicted from the queue.

Setup: the `merge_group-checks-requested