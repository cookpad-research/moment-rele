# moment-rele

> R.E.L.E すべての更新は記録される。

Shared GitHub Actions workflows and composite actions for the Moment organization.

## Available Actions

### slack-notify

Send formatted Slack notifications for CI/CD events with consistent styling.

#### Features

- Color-coded status (green for success, red for failure, yellow for cancelled)
- Emoji indicators for quick visual status
- Automatic type prefixes (Build, Deploy, Release, etc.)
- Optional version and environment fields
- Optional changelog for release notifications
- Link to GitHub Actions run details

#### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `status` | Yes | - | Workflow status: `success`, `failure`, `cancelled` |
| `type` | Yes | - | Notification type: `build`, `deploy-staging`, `deploy-production`, `release`, `test`, `ci`, `review` |
| `title` | Yes | - | Notification title (e.g., "Learner App", "moment-web") |
| `slack-webhook-url` | Yes | - | Slack incoming webhook URL |
| `version` | No | `''` | App version to display |
| `environment` | No | `''` | Environment name (staging/production) |
| `changelog` | No | `''` | Changelog content for release notifications |
| `mentions` | No | `''` | Slack user group IDs (e.g., `S0AC8FPCFMW`) or user IDs (e.g., `U12345`) to mention. Comma-separated for multiple. |
| `mention-context` | No | `''` | Optional message to show with mentions (e.g., `"Please review:"`, `"FYI:"`) |

#### Usage

```yaml
jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout moment-rele
        uses: actions/checkout@v4
        with:
          repository: cookpad-research/moment-rele
          ref: main
          sparse-checkout: .github/actions
          path: .moment-rele

      - name: Notify Slack
        uses: ./.moment-rele/.github/actions/slack-notify
        with:
          status: success
          type: deploy-production
          title: moment-web
          version: '1.2.3'
          environment: production
          slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

#### Example Output

```
:white_check_mark: moment-web - Production Deploy Success

Branch/Tag: v1.2.3
Triggered by: username
Version: 1.2.3
Environment: production

View Details
```

#### Mention Support

You can mention Slack user groups or individual users in notifications to ensure specific teams or people are notified:

**Simple user group mention:**
```yaml
- name: Notify Slack
  uses: ./.moment-rele/.github/actions/slack-notify
  with:
    status: success
    type: deploy-production
    title: My App
    version: '1.2.3'
    mentions: 'S0AC8FPCFMW'  # User group ID
    slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**Multiple mentions with context:**
```yaml
- name: Notify Slack with mentions
  uses: ./.moment-rele/.github/actions/slack-notify
  with:
    status: success
    type: release
    title: Learner App
    version: '1.24.0'
    mentions: 'S0AC8FPCFMW, S123XYZ, U456ABC'  # Multiple IDs
    mention-context: 'Please review and push to the App Store.'
    slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**Conditional mentions (production only):**
```yaml
- name: Notify Slack
  uses: ./.moment-rele/.github/actions/slack-notify
  with:
    status: ${{ steps.status.outputs.status }}
    type: deploy-production
    title: My App
    mentions: ${{ needs.setup.outputs.environment == 'production' && steps.status.outputs.status == 'success' && 'S0AC8FPCFMW' || '' }}
    mention-context: 'Deployment complete.'
    slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**Supported ID formats:**
- User group IDs starting with 'S' (e.g., `S0AC8FPCFMW`) → Auto-formatted as `<!subteam^ID>`
- User IDs starting with 'U' or 'W' (e.g., `U12345`) → Auto-formatted as `<@ID>`
- Pre-formatted strings (e.g., `<!subteam^S0AC8FPCFMW>`) → Used as-is
- Special mentions (e.g., `<!here>`, `<!channel>`) → Used as-is

---

### parse-changelog

Extract and format changelog from GitHub release body or auto-generate it.

#### Features

- Parse changelog from `github.event.release.body`
- Auto-generate changelog using [mikepenz/release-changelog-builder-action](https://github.com/mikepenz/release-changelog-builder-action)
- Format markdown for Slack compatibility
- Truncate to configurable length (respects Slack limits)

#### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `mode` | No | `release-body` | Mode: `release-body` or `auto-generate` |
| `release-body` | No | `''` | Release body content to parse |
| `max-length` | No | `2000` | Maximum character length for changelog |
| `github-token` | No | `''` | GitHub token (required for `auto-generate` mode) |

#### Outputs

| Output | Description |
|--------|-------------|
| `changelog` | Formatted changelog text |
| `has-changelog` | Boolean (`true`/`false`) indicating if changelog was found |

#### Usage

```yaml
jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout moment-rele
        uses: actions/checkout@v4
        with:
          repository: cookpad-research/moment-rele
          ref: main
          sparse-checkout: .github/actions
          path: .moment-rele

      # Parse changelog from release body
      - name: Parse changelog
        id: changelog
        uses: ./.moment-rele/.github/actions/parse-changelog
        with:
          mode: release-body
          release-body: ${{ github.event.release.body }}
          max-length: 1500

      # Use in notification
      - name: Notify Slack
        uses: ./.moment-rele/.github/actions/slack-notify
        with:
          status: success
          type: release
          title: My App
          version: ${{ github.event.release.tag_name }}
          changelog: ${{ steps.changelog.outputs.changelog }}
          slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

#### Auto-generate Mode

```yaml
- name: Generate changelog
  id: changelog
  uses: ./.moment-rele/.github/actions/parse-changelog
  with:
    mode: auto-generate
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

---

## Complete Example: Release Notification with Changelog

```yaml
name: Deploy Production

on:
  release:
    types: [published]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: echo "Deploying..."

  notify:
    needs: [deploy]
    # IMPORTANT: If you rename the deploy job, update both `needs:` AND the
    # `needs.<job>.result` references below — they must match the job id exactly.
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Determine status
        id: status
        run: |
          if [[ "${{ needs.deploy.result }}" == "success" ]]; then
            echo "status=success" >> $GITHUB_OUTPUT
          elif [[ "${{ needs.deploy.result }}" == "cancelled" ]]; then
            echo "status=cancelled" >> $GITHUB_OUTPUT
          else
            echo "status=failure" >> $GITHUB_OUTPUT
          fi

      - name: Checkout moment-rele
        uses: actions/checkout@v4
        with:
          repository: cookpad-research/moment-rele
          ref: main
          sparse-checkout: .github/actions
          path: .moment-rele

      - name: Parse changelog
        id: changelog
        uses: ./.moment-rele/.github/actions/parse-changelog
        with:
          release-body: ${{ github.event.release.body }}

      - name: Notify Slack
        uses: ./.moment-rele/.github/actions/slack-notify
        with:
          status: ${{ steps.status.outputs.status }}
          type: release
          title: My App
          version: ${{ github.event.release.tag_name }}
          environment: production
          changelog: ${{ steps.changelog.outputs.changelog }}
          slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Required Secrets

Each repository using these actions needs the following secret:

| Secret | Description |
|--------|-------------|
| `SLACK_WEBHOOK_URL` | Slack incoming webhook URL for your notification channel |

### Setting up Slack Webhook

1. Go to your Slack workspace's App Directory
2. Search for "Incoming Webhooks" or create a new Slack App
3. Add a new webhook and select the target channel
4. Copy the webhook URL and add it as a repository secret

---

## Versioning

This repository uses semantic versioning. Pin to specific versions for stability:

```yaml
# Recommended: Pin to main for latest
ref: main

# Alternative: Pin to specific commit for maximum stability
ref: abc1234
```

---

## Repository Structure

```
moment-rele/
├── .github/
│   └── actions/
│       ├── slack-notify/
│       │   └── action.yml      # Slack notification action
│       └── parse-changelog/
│           └── action.yml      # Changelog parsing action
└── README.md                   # This file
```

---

## Contributing

1. Create a feature branch
2. Make your changes
3. Test the actions in a workflow
4. Submit a pull request

---

## License

Internal use only - Cookpad Research
