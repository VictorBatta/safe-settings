# GitHub Safe-Settings

[![Node.js CI](https://github.com/github/safe-settings/actions/workflows/node-ci.yml/badge.svg)](https://github.com/github/safe-settings/actions/workflows/node-ci.yml)
[![Dependabot][dependabot-badge]][dependabot-link]

`Safe-settings`– an app to manage policy-as-code and apply repository settings to repositories across an organization.

1. In `safe-settings` all the settings are stored centrally in an `admin` repo within the organization. This is important. Unlike [Settings Probot](https://github.com/probot/settings), the settings files cannot be in individual repositories.
1. There are 3 levels at which the settings could be managed:
   1. Org-level settings are defined in `.github/settings.yml` 
   1. `Suborg` level settings. A `suborg` is an arbitrary collection of repos belonging to projects, business units, or teams. The `suborg`settings reside in a yaml file for each `suborg` in the `.github/suborgs`folder.
   1. `Repo` level settings. They reside in a repo specific yaml in `.github/repos`folder
1. It is recommended to break the settings into org-level, suborg-level, and repo-level units. This will allow different teams to be define and manage policies for their specific projects or business units.With `CODEOWNERS`, this will allow different people to be responsible for approving changes in different projects.

**Note:** The settings file must have a `.yml`extension only. `.yaml` extension is ignored, for now.

## How it works

### Events
The App listens to the following webhook events:

- **push**: If the settings are created or modified, that is, if  push happens in the `default` branch of the `admin` repo and the file added or changed is `.github/settings.yml` or `.github/repos/*.yml`or `.github/suborgs/*.yml`, then the settings would be applied either globally to all the repos, or specific repos. For each repo, the settings that is actually applied depend on the default settings for the org, overlayed with settings for the suborg that the repo belongs to, overlayed with the settings for that specific repo.
  
- **repository.created**: If a repository is created in the org, the settings for the repo - the default settings for the org, overlayed with settings for the suborg that the repo belongs to, overlayed with the settings for that specific repo - is applied. 

- **branch_protection_rule**: If a branch protection rule is modified or deleted, `safe-settings` will `sync` the settings to prevent any unauthorized changes.

- **repository.edited**: If the `default` branch is changed if the the settings that would be applied for the repo

- **pull_request.opened**, **pull_request.reopened**, **check_suite.requested**: If the settings are changed, but it is not in the `default` branch, and there is an existing PR, the code will validate the settings changes by running safe-settings in `nop` mode and update the PR with the `dry-run` status. 

**Exceptions:** Certain repos may be exempted from policy enforcement by specifying them in a runtime config file called `deployment-settings.yml`. If no file is specified, then the following repositories -  `'admin', '.github', 'safe-settings'` are exempted by default.

### Schedule

The App can be configured to apply the settings on a schedule. This could a way to address configuration drift since webhooks have not always guaranteed to be delivered.

 To set periodically converge the settings to the configuration, set the `CRON` environment variable. This is based on [node-cron](https://www.npmjs.com/package/node-cron) and details on the possible values can be found [here](#Env variables).

### Pull Request Workflow
`Safe-settings` explicitly looks in the `admin` repo in the organization for the settings files. The `admin` repo could be a restricted repository with `branch protections` and `codeowners`  

In that set up, when a changes happen to the settings files and there is a PR for merging the changes back to the `default` branch in the `admin` repo, `safe-settings` will run `checks`  – which will run in **nop** mode and produce a report of the changes that would happen, including the API calls and the payload. 

The checks will fail if `org-level`branch protections are overridden at the repo or suborg level with a lesser number of required approvers.

### The Settings file

The settings file can be used to set the policies at the `Org`, `suborg` or `repo` level. 

Using the settings, the following things could be configured:

- `Repository settings` - home page, url, visibility, has_issues, has_projects, wikis, etc.
- `default branch`naming and renaming 
- `Repository Topics`
- `Teams and permissions`
- `Collaborators and permissions`
- `Issue labels`
- `Branch protections`. If the name of the branch is `default` in the settings, it is applied to the `default` branch of the repo.
- `repository name validation` using regex pattern

It is possible to provide an `include` or `exclude` settings to restrict the `collaborators`, `teams`, `labels` to a list of repos or exclude a set of repos for a collaborator.

Here is an example settings file:


```yaml
# These settings are synced to GitHub by https://github.com/github/safe-settings

repository: 
  # This is the settings that need to be applied to all repositories in the org 
  # See https://docs.github.com/en/rest/reference/repos#create-an-organization-repository for all available settings for a repository  
  # A short description of the repository that will show up on GitHub
  description: description of the repo
  
  # A URL with more information about the repository
  homepage: https://example.github.io/
    
  # Keep this as true for most cases
  # A lot of the policies below cannot be implemented on bare repos
  # Pass true to create an initial commit with empty README.
  auto_init: true
    
  # A comma-separated list of topics to set on the repository
  topics: github, probot, new-topic, another-topic, topic-12
  
  # Either `true` to make the repository private, or `false` to make it public. 
  # If this value is changed and if Org members cannot change the visibility of repos
  # it would result in an error when updating a repo
  private: true
  
  # Can be public or private. If your organization is associated with an enterprise account using 
  # GitHub Enterprise Cloud or GitHub Enterprise Server 2.20+, visibility can also be internal. 
  visibility: private
  
  # Either `true` to enable issues for this repository, `false` to disable them.
  has_issues: true
  
  # Either `true` to enable projects for this repository, or `false` to disable them.
  # If projects are disabled for the organization, passing `true` will cause an API error.
  has_projects: true
  
  # Either `true` to enable the wiki for this repository, `false` to disable it.
  has_wiki: true
  
  # The default branch for this repository.
  default_branch: main-enterprise
  
  # Desired language or platform [.gitignore template](https://github.com/github/gitignore) 
  # to apply. Use the name of the template without the extension. 
  # For example, "Haskell".
  gitignore_template: node
  
  # Choose an [open source license template](https://choosealicense.com/) 
  # that best suits your needs, and then use the 
  # [license keyword](https://help.github.com/articles/licensing-a-repository/#searching-github-by-license-type) 
  # as the `license_template` string. For example, "mit" or "mpl-2.0".
  license_template: mit
  
  # Either `true` to allow squash-merging pull requests, or `false` to prevent
  # squash-merging.
  allow_squash_merge: true
  
  # Either `true` to allow merging pull requests with a merge commit, or `false`
  # to prevent merging pull requests with merge commits.
  allow_merge_commit: true
  
  # Either `true` to allow rebase-merging pull requests, or `false` to prevent
  # rebase-merging.
  allow_rebase_merge: true
  
  # Either `true` to allow auto-merge on pull requests, 
  # or `false` to disallow auto-merge.
  # Default: `false`
  allow_auto_merge: true
  
  # Either `true` to allow automatically deleting head branches 
  # when pull requests are merged, or `false` to prevent automatic deletion.
  # Default: `false`
  delete_branch_on_merge: true  
      
# The following attributes are applied to any repo within the org
# So if a repo is not listed above is created or edited
# The app will apply the following settings to it
labels:
  # Labels: define labels for Issues and Pull Requests
  - name: bug
    color: CC0000
    description: An issue with the system

  - name: feature
    # If including a `#`, make sure to wrap it with quotes!
    color: '#336699'
    description: New functionality.

  - name: first-timers-only
    # include the old name to rename an existing label
    oldname: Help Wanted
    color: '#326699'

  - name: new-label
    # include the old name to rename an existing label
    oldname: Help Wanted
    color: '#326699'

collaborators:
# Collaborators: give specific users access to any repository.
# See https://docs.github.com/en/rest/reference/collaborators#add-a-repository-collaborator for available options
- username: regpaco
  permission: push
# The permission to grant the collaborator. Can be one of:
# * `pull` - can pull, but not push to or administer this repository.
# * `push` - can pull and push, but not administer this repository.
# * `admin` - can pull, push and administer this repository.
- username: beetlejuice
  permission: pull
# You can exclude a list of repos for this collaborator and all repos except these repos would have this collaborator
  exclude:
  - actions-demo

- username: thor
  permission: push
# You can include a list of repos for this collaborator and only those repos would have this collaborator
  include:
  - actions-demo
  - another-repo

# See https://docs.github.com/en/rest/reference/teams#create-a-team for available options
teams:
  - name: core
    # The permission to grant the team. Can be one of:
    # * `pull` - can pull, but not push to or administer this repository.
    # * `push` - can pull and push, but not administer this repository.
    # * `admin` - can pull, push and administer this repository.
    permission: admin
  - name: docss
    permission: push
  - name: docs
    permission: pull

branches:
  # If the name of the branch value is specified as `default`, then the app will create a branch protection rule to apply against the default branch in the repo
  - name: default
    # https://docs.github.com/en/rest/reference/branches#update-branch-protection
    # Branch Protection settings. Set to null to disable
    protection:
      # Required. Require at least one approving review on a pull request, before merging. Set to null to disable.
      required_pull_request_reviews:
        # The number of approvals required. (1-6)
        required_approving_review_count: 1
        # Dismiss approved reviews automatically when a new commit is pushed.
        dismiss_stale_reviews: true
        # Blocks merge until code owners have reviewed.
        require_code_owner_reviews: true
        # Specify which users and teams can dismiss pull request reviews. Pass an empty dismissal_restrictions object to disable. User and team dismissal_restrictions are only available for organization-owned repositories. Omit this parameter for personal repositories.
        dismissal_restrictions:
          users: []
          teams: []
      # Required. Require status checks to pass before merging. Set to null to disable
      required_status_checks:
        # Required. Require branches to be up to date before merging.
        strict: true
        # Required. The list of status checks to require in order to merge into this branch
        contexts: []
      # Required. Enforce all configured restrictions for administrators. Set to true to enforce required status checks for repository administrators. Set to null to disable.
      enforce_admins: true
      # Required. Restrict who can push to this branch. Team and user restrictions are only available for organization-owned repositories. Set to null to disable.
      restrictions:
        apps: []
        users: []
        teams: []
        
validator:
  #pattern: '[a-zA-Z0-9_-]+_[a-zA-Z0-9_-]+.*' 
  pattern: '[a-zA-Z0-9_-]+'
```



### Additional values

In addition to these values above, the settings file can have some addtional values

1.  `force_create`: This is set in the repo-level settings to force create the repo if the repo does not exist. 
2. `template`: This is set in the repo-level settings, and is used with the `force_create`flag to use a specific repo template when creating the repo
3. `suborgrepos`: This is set in the suborg-level settings to define an array of repos. This field can also take a `glob` pattern to allow wild-card expression to specify repos in a suborg. For e.g. `test*`would include `test`, `test1`, `testing`, etc.
4. The `suborgteams` section contains a list of teams, and all the repos belonging to the teams would be part of the `suborg` 



### Env variables

You can pass environment variables; easiest way to do it is in a `.env`file.

1. __CRON__ you can pass a cron input to run `safe-settings` at a regular schedule. This is based on [node-cron](https://www.npmjs.com/package/node-cron). For eg.
```
# ┌────────────── second (optional)
# │ ┌──────────── minute
# │ │ ┌────────── hour
# │ │ │ ┌──────── day of month
# │ │ │ │ ┌────── month
# │ │ │ │ │ ┌──── day of week
# │ │ │ │ │ │
# │ │ │ │ │ │
# * * * * * *
CRON=* * * * * # Run every minute
```
1. Logging level could be set using **LOG_LEVEL**. For e.g.
```
LOG_LEVEL=trace
```

### Runtime Settings 

1. Besides the above settings files, the application can be bootstrapped with `runtime` settings.
2. The `runtime` settings are configured in `deployment-settings.yml` that is in the directory from where the GitHub app is running.
3. Currently the only setting that is possible are `restrictedRepos: [... ]` which allows you to configure a list of repos within your `org` that are excluded from the settings. If the `deployment-settings.yml` is not present, the following repos are added by default to the `restricted`repos list: `'admin', '.github', 'safe-settings'`


### Notes

1. Label color can also start with `#`, e.g. `color: '#F341B2'`. Make sure to wrap it with quotes!
1. Each top-level element under branch protection must be filled (eg: `required_pull_request_reviews`, `required_status_checks`, `enforce_admins` and `restrictions`). If you don't want to use one of them you must set it to `null` (see comments in the example above). Otherwise, none of the settings will be applied.
2. The precedence order is repository > suborg > org (.github/repos/*.yml > .github/suborgs/*.yml > .github/settings.yml



## How to use

1. __[Install the app](docs/deploy.md)__. 

2. Create an `admin` repo within your organization (the repository must be called `admin`). 

3. Add the settings for the `org`, `suborgs`, and `repos` . List of sample files could be found [here](docs/sample-settings).

   

## Deployment

See [docs/deploy.md](docs/deploy.md) if you would like to run your own instance of this plugin.

## License

`safe-settings` is licensed under the [ISC license](https://github.com/github/safe-settings/blob/master/LICENSE)

`safe-settings` uses 3rd party libraries, each with their own license. These are found [here](https://github.com/github/safe-settings/blob/master/NOTICE.md).


[dependabot-link]: https://dependabot.com/

[dependabot-badge]: https://badgen.net/dependabot/probot/settings/?icon=dependabot

[github-actions-ci-link]: https://github.com/probot/settings/actions?query=workflow%3A%22Node.js+CI%22+branch%3Amaster

[github-actions-ci-badge]: https://github.com/probot/settings/workflows/Node.js%20CI/badge.svg
