# deploy-stack-action

Executes a `stack` deploy via self-hosted GitHub Actions runner.

Allows us to reuse this logic across all applications we deploy via [Stack](https://github.com/warpcast/stack).

## Inputs
- `git-repo` **(required)** GitHub repository to clone (e.g. `my-org/my-repo-name`)
- `git-ref` **(required)** Commit ref to check out (typically a hash)
- `docker-image` **(required)** Full URL to the Docker image to pull on deployed instances
- `project` **(required)** the project name as specified in the `deploy.yml` configuration
- `release-id` _(optional)_ name of the release (default timestamp + commit hash of the form `2024-12-31T23-59-59-123Z-deadbeef`)

## How to use

If you haven't already, register a dedicated self-hosted GitHub Actions runner.

```
PROJECT=my-project
REPO=my-project # Sometimes you'll have a separate private repo for deploying. Specify that if so.
RUNNER_VERSION=2.324.0

mkdir actions-runner
cd actions-runner
curl -o actions-runner-linux-arm64-${RUNNER_VERSION}.tar.gz -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-arm64-${RUNNER_VERSION}.tar.gz
tar xzf ./actions-runner-linux-arm64-${RUNNER_VERSION}.tar.gz
```

Go to your repository's Runners settings and get a temporary token.
For example: `https://github.com/warpcast/my-project/settings/actions/runners/new?arch=arm64&os=linux`

Configure and install the runner as a service:
```
./config.sh --url https://github.com/warpcast/${REPO} --token ${TOKEN} --name ${PROJECT}-deployer --work ${PROJECT}-deployer --labels ${PROJECT}-deployer --unattended

sudo ./svc.sh install
sudo ./svc.sh start
```

Invoke in your workflow using:

```yaml
jobs:
  my-job:
    runs-on: my-project-deployer # Must match a GitHub Actions self-hosted runner that was previously registered with this label
    steps:
      - uses: warpcast/deploy-action
        with:
          git-repo: ${{ github.repository }}
          git-ref: ${{ github.sha }}
          docker-image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-organization/my-repository:${{ github.sha }} # ECR repo
          project: 'my-project'
          pods: 'pod-a pod-b pod-c' # Optional
```

There's no need to clone the repo before invoking this action as it will handle the cloning itself.
This action assumes that only trusted code is being executed by the time it is run.
Thus it is **NOT** recommended to use this action as part of a pull request, but rather as part of a release workflow that runs post-merge.
