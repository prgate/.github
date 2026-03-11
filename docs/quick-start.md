# Quick Start

Get PRGate reviewing your PRs in under 5 minutes.

## 1. Create your policy file

Add `org-policy.yaml` to your repo root. This tells PRGate what to look for:

```yaml
policy_version: 1.0.0
severity_threshold: medium

org_rules:
  critical:
    - rule_id: sql_injection
      name: SQL Injection Risk
      enabled: true
    - rule_id: hardcoded_secrets
      name: Hardcoded Secrets
      enabled: true
  high:
    - rule_id: insufficient_validation
      name: Insufficient Input Validation
      enabled: true
  medium: []
  low: []
  info: []
```

Start with 2-3 rules. Add more as you tune signal vs. noise.

## 2. Create your repo config

Add `.prgate.yaml` to your repo root:

```yaml
config_version: 1.0.0
enabled: true
ai:
  provider: anthropic        # or: openai, deepseek, ollama, groq, etc.
  model: claude-sonnet-4-20250514
min_confidence: 0.8
max_findings: 10
ignore_paths:
  - "**/*.test.py"
  - "docs/**"
  - ".venv/**"
```

Any provider supported by [litellm](https://docs.litellm.ai/docs/providers) works. Some cheap options:

| Provider | Model | Cost |
|----------|-------|------|
| DeepSeek | `deepseek-chat` | ~$0.001/PR |
| Groq | `groq/llama-3.1-70b-versatile` | Free tier available |
| Ollama | `ollama/codellama` | Free (self-hosted) |
| Anthropic | `claude-sonnet-4-20250514` | ~$0.01/PR |
| OpenAI | `gpt-4o` | ~$0.01/PR |

## 3. Set up the GitHub Action

Create `.github/workflows/prgate.yml`:

```yaml
name: PRGate Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  prgate-review:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: prgate-org/prgate@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          llm-api-key: ${{ secrets.LLM_API_KEY }}
```

## 4. Add your LLM API key

Go to **Settings > Secrets and variables > Actions** in your GitHub repo. Add:

| Secret name | Value |
|-------------|-------|
| `LLM_API_KEY` | Your provider's API key |

`GITHUB_TOKEN` is provided automatically by GitHub Actions.

## 5. Open a PR

Push a branch, open a PR. PRGate will:

1. Read the PR diff
2. Filter out ignored paths
3. Send the diff + your policy rules to the LLM
4. Validate findings against schema
5. Post comments on the PR

## What the output looks like

PRGate posts a summary comment on your PR:

```markdown
## PRGate Code Review

Found **3** finding(s):

### CRITICAL

- **User input passed directly to SQL query** (src/db/users.py:42)
  - Confidence: 95%
  - Evidence: `cursor.execute(f"SELECT * FROM users WHERE id={user_id}")`
  - Action: Use parameterized queries: `cursor.execute("SELECT * FROM users WHERE id=?", (user_id,))`
  - Rule: `sql_injection`

### HIGH

- **Missing input validation on API endpoint** (src/api/routes.py:18)
  - Confidence: 88%
  - Action: Validate request body with Pydantic model before processing
  - Rule: `insufficient_validation`

### MEDIUM

- **Bare except clause swallows all errors** (src/utils/retry.py:31)
  - Confidence: 82%
  - Evidence: `except: pass`
  - Action: Catch specific exceptions: `except (ConnectionError, Timeout)`
  - Rule: `error_handling`
```

Findings with line numbers also appear as inline comments on the diff.

## Customizing for your org

### Override rules per repo

In `.prgate.yaml`, you can override org policy:

```yaml
# Disable a noisy rule for this repo
rule_overrides:
  code_style:
    enabled: false
```

### Use a local LLM (Ollama)

```yaml
ai:
  provider: ollama
  model: codellama
  api_base: http://localhost:11434
```

No API key needed. Your code never leaves your network.

### Reduce noise

- Raise `min_confidence` to `0.9` to only see high-confidence findings
- Lower `max_findings` to `5` to cap output
- Add `ignore_paths` for generated code, vendored deps, tests

## Troubleshooting

**PRGate posts no comments:**
- Check the Actions tab for errors in the workflow run
- Verify `LLM_API_KEY` is set in repo secrets
- Ensure `pull-requests: write` permission is set

**Too many findings / noise:**
- Raise `min_confidence` (default: none, try `0.85`)
- Lower `max_findings` (default: 20, try `5`)
- Narrow your policy rules — fewer rules = better signal

**LLM call failed:**
- Verify your API key is valid and has credits
- Check the model name matches your provider's format
- For Ollama: ensure the model is pulled and the server is running

## Next steps

- [Full configuration reference](https://github.com/prgate/prgate#readme#configuration)
- [Policy file format](https://github.com/prgate/prgate#readme#org-wide-policy-org-policyyaml)
- [Finding schema](https://github.com/prgate/prgate#readme#finding-schema)
