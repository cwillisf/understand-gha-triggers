# Understanding GitHub Actions triggers

This repository is an attempt to better understand the details of triggering GitHub Actions events.

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
`sha` | string | Git SHA of the base branch
`head_ref` | string | `"example-branch"`
`base_ref` | string | `"another-base"`
`event_name` | string | `"pull_request_target"`
`ref_name` | string | `"another-base"`
`ref_type` | string | `"branch"`

`github.event` property | Type | Value
--- | --- | ---
`action` | string | `"synchronize"`
`after` | string | Git SHA of the branch after the push = Git SHA of the commit that was pushed
`before` | string | Git SHA of the branch before the push
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
