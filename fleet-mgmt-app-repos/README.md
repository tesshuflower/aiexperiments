# Claude code procedures for ACM business repository repos including volsync building

## Working with kube contexts

Currently expecting 1 kubeconfig file per cluster (rather than working with
multiple contexts).

- kubeconfig files can be copied to the fleet-mgmt-app-repos/.kube dir
  (or you can ask Claude to copy them for you)


## Slack notifications:

If you want slack notifications (you can tell claude to run a task or
monitor something and then inform you when it's complete), you can do
the following:

### Create a slack channel to post to:
1. Go to slack > channels and create a new channel
   (for these examples I will call it `tflow-dev-notifications`

### Create a slack app that claude will use:

1. Go to https://api.slack.com/apps
1. Click "Create New App"
1. Choose "From scratch"
1. Give it a name like "Dev Notifications"
1. Select your workspace
1. Go to "Incoming Webhooks" in the left sidebar
1. Toggle "Activate Incoming Webhooks" to On
1. Click "Add New Webhook to Workspace"
   (may need to click button "Request to add new Webhook" first)
1. Select your #tflow-dev-notifications channel
1. Click "Allow"

### Now set the CLAUDE_SLACK_WEBHOOK_URL env var

```bash
export CLAUDE_SLACK_WEBHOOK_URL="your-webhook-url"
```

or, to set this globally:

```bash
# Add to ~/.bashrc or ~/.zshrc
echo 'export CLAUDE_SLACK_WEBHOOK_URL="your-webhook-url"' >> ~/.bashrc
```

Now restart claude code and it should be able to find the env var
