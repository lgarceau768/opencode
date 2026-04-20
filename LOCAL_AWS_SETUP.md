# Local Build & AWS Bedrock Setup

Instructions for cloning, building, and running this fork of opencode with AWS Bedrock.

## Prerequisites

- [Bun](https://bun.sh) v1.3+
- [AWS CLI](https://aws.amazon.com/cli/) v2
- Git + SSH key configured for GitHub

## Clone & Build

```bash
git clone git@github.com:lgarceau768/opencode.git
cd opencode

# Install dependencies (--ignore-scripts avoids native build failures)
bun install --ignore-scripts

# Build for your current platform only
bun run --bun --cwd packages/opencode build --single --skip-embed-web-ui
```

The binary will be at:
- macOS ARM64: `packages/opencode/dist/opencode-darwin-arm64/bin/opencode`
- macOS x64:   `packages/opencode/dist/opencode-darwin-x64/bin/opencode`
- Linux ARM64:  `packages/opencode/dist/opencode-linux-arm64/bin/opencode`
- Linux x64:    `packages/opencode/dist/opencode-linux-x64/bin/opencode`

Verify the build:

```bash
./packages/opencode/dist/opencode-darwin-arm64/bin/opencode --version
```

## AWS Bedrock Setup

### 1. Configure AWS SSO profile

Add to `~/.aws/config`:

```ini
[profile AdministratorAccess]
sso_start_url = <your-sso-start-url>
sso_region = <your-sso-region>
sso_account_id = <your-account-id>
sso_role_name = AdministratorAccess
region = <your-preferred-region>
```

### 2. Configure opencode for Bedrock

Create `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "amazon-bedrock/us.anthropic.claude-sonnet-4-6",
  "enabled_providers": ["amazon-bedrock"],
  "plugin": [],
  "provider": {
    "amazon-bedrock": {
      "options": {
        "region": "<your-preferred-region>"
      },
      "models": {
        "writer.palmyra-x4": {
          "id": "us.writer.palmyra-x4-v1:0",
          "name": "Writer Palmyra X4",
          "tool_call": false,
          "limit": { "context": 128000, "output": 8192 }
        },
        "writer.palmyra-x5": {
          "id": "us.writer.palmyra-x5-v1:0",
          "name": "Writer Palmyra X5",
          "tool_call": false,
          "limit": { "context": 1000000, "output": 8192 }
        },
        "deepseek.r1": {
          "id": "us.deepseek.r1-v1:0",
          "name": "DeepSeek R1",
          "tool_call": false,
          "limit": { "context": 64000, "output": 32768 }
        },
        "mistral.pixtral-large-2502": {
          "id": "us.mistral.pixtral-large-2502-v1:0",
          "name": "Mistral Pixtral Large",
          "tool_call": false,
          "limit": { "context": 128000, "output": 8192 }
        },
        "meta.llama4-maverick-17b-instruct": {
          "id": "us.meta.llama4-maverick-17b-instruct-v1:0",
          "name": "Meta Llama 4 Maverick 17B",
          "tool_call": false,
          "limit": { "context": 1000000, "output": 8192 }
        },
        "meta.llama4-scout-17b-instruct": {
          "id": "us.meta.llama4-scout-17b-instruct-v1:0",
          "name": "Meta Llama 4 Scout 17B",
          "tool_call": false,
          "limit": { "context": 10000000, "output": 8192 }
        },
        "amazon.nova-2-lite": {
          "id": "us.amazon.nova-2-lite-v1:0",
          "name": "Amazon Nova 2 Lite",
          "limit": { "context": 300000, "output": 5120 }
        }
      }
    }
  }
}
```

> **Note on `tool_call: false`:** Models marked with `tool_call: false` do not support
> tool use in streaming mode on Bedrock. This config prevents opencode from sending
> tool definitions to those models. This fork includes a fix (cherry-picked from
> [anomalyco/opencode#20040](https://github.com/anomalyco/opencode/pull/20040)) that
> makes `tool_call: false` actually respected at runtime.

### 3. Shell alias

Add to `~/.zshrc` or `~/.bashrc`:

```bash
opencode-work() {
  local profile="AdministratorAccess"
  echo "Logging in to AWS SSO ($profile)..."
  aws sso login --profile "$profile" || return 1
  eval "$(aws configure export-credentials --profile "$profile" --format env)"
  /path/to/opencode/packages/opencode/dist/opencode-darwin-arm64/bin/opencode "$@"
}
```

Replace `/path/to/opencode` with where you cloned the repo.

### 4. Usage

```bash
# Login and launch
opencode-work

# Re-running after SSO session expires — just run again, it will re-authenticate
opencode-work
```

## What's Different in This Fork

- **`tool_call: false` fix** — upstream opencode parses this config field but the
  binary does not enforce it, causing failures on Bedrock models that don't support
  streaming + tool use together. This fork includes the fix from
  [anomalyco/opencode#20040](https://github.com/anomalyco/opencode/pull/20040)
  until it lands in the official release.

Tracked upstream in: [sst/opencode#19966](https://github.com/sst/opencode/issues/19966)
