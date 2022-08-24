# GitHubDocs

Documentation for the Screenful GitHub integration

## How to: authorise GitHub so that Screenful has only read-only access

### Pre-requisites: 

- some experience with [terraform infrastructure as code][1] applied to github.

### How-to

1. Create a new bot/service-account github user via github signup.
    - note: the bot/service-account should have only `pull` access to repositories. If you have an existing bot you can re-use, you can. Be aware of privilege escalation for that user. 
3. Set up <https://registry.terraform.io/providers/integrations/github/latest/docs> terraform/github.
4. Configure terraform `github_membership` to invite the bot to the org:

    ```terraform
    resource "github_membership" "bot" {
      username = "robot-github-username-customize-this"
      role     = "member"
    }
    ```
    
3. Configure terraform `github_team` to make a new `bots-readonly` team (teams permission-assignment is lower maintenance than managing individual users):

    ```terraform
    resource "github_team" "bots-readonly" {
      name       = "bots-readonly"
    }
    ```

4. Configure terraform `github_team_membership` to place the user in the team:

    ```terraform
    resource "github_team_membership" "bot-is-member-of-bots-readonly" {
      team_id  = github_team.bots-readonly.id
      username = github_membership.bot.username
      role     = "member"
    }
    ```

5. Configure `n` `github_repository` resources to grant access to via team/repo linkage (in next step):

    ```terraform
    resource "github_repository" "some-repo" {
      name = "..."
      # ...
    }
    ```

6. Configure terraform `github_team_repository` to grant read-only access to the team to the repositories you want to wire up:

    ```terraform
    resource "github_team_repository" "bots-readonly-can-pull-from-some-repo" {
      team_id    = github_team.bots-readonly.id
      repository = github_repository.some-repo.name
      permission = "pull"
    }
    # ... n times, 1 per repo
    ```

6. `terraform apply`, and `yes` if the changes look good to you.
7. as the bot, sign up to screenful.com
8. as a github org-admin, accept the invite - <https://github.com/organizations/ORG_NAME/settings/oauth_application_policy>
9. as the bot in screenful, add github data source(s)

### Caveats

1. The user (like all users in your org) will inherit the org's `Member privileges > Base permissions`. 
    - Assumption: these will not be more than `Read`. 
    - Check via <https://github.com/organizations/ORG_NAME/settings/member_privileges>
    - If these change in future, the user has privileges escalated silently.

[1]: https://learn.hashicorp.com/tutorials/terraform/github-user-teams
