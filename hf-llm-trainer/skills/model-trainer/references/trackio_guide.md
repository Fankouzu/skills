# Trackio Integration for TRL Training

**Trackio** is an experiment tracking library that provides real-time metrics visualization for remote training on Hugging Face Jobs infrastructure.

⚠️ **IMPORTANT**: For Jobs training (remote cloud GPUs):
- Training happens on ephemeral cloud runners (not your local machine)
- Trackio syncs metrics to a Hugging Face Space for real-time monitoring
- Without a Space, metrics are lost when the job completes
- The Space dashboard persists your training metrics permanently

## Setting Up Trackio for Jobs

**Step 1: Add trackio dependency**
```python
# /// script
# dependencies = [
#     "trl>=0.12.0",
#     "trackio",  # Required!
# ]
# ///
```

**Step 2: Create a Trackio Space (one-time setup)**

**Option A: Let Trackio auto-create (Recommended)**
Pass a `space_id` to `trackio.init()` and Trackio will automatically create the Space if it doesn't exist.

**Option B: Create manually**
- Create Space via Hub UI at https://huggingface.co/new-space
- Select Gradio SDK
- OR use command: `huggingface-cli repo create my-trackio-dashboard --type space --space_sdk gradio`

**Step 3: Initialize Trackio with space_id**
```python
import trackio

trackio.init(
    project="my-training",
    space_id="username/trackio",  # CRITICAL for Jobs! Replace 'username' with your HF username
    config={
        "model": "Qwen/Qwen2.5-0.5B",
        "dataset": "trl-lib/Capybara",
        "learning_rate": 2e-5,
    }
)
```

**Step 4: Configure TRL to use Trackio**
```python
SFTConfig(
    report_to="trackio",
    # ... other config
)
```

**Step 5: Finish tracking**
```python
trainer.train()
trackio.finish()  # Ensures final metrics are synced
```

## What Trackio Tracks

Trackio automatically logs:
- ✅ Training loss
- ✅ Learning rate
- ✅ GPU utilization
- ✅ Memory usage
- ✅ Training throughput
- ✅ Custom metrics

## How It Works with Jobs

1. **Training runs** → Metrics logged to local SQLite DB
2. **Every 5 minutes** → Trackio syncs DB to HF Dataset (Parquet)
3. **Space dashboard** → Reads from Dataset, displays metrics in real-time
4. **Job completes** → Final sync ensures all metrics persisted

## Default Configuration Pattern

**Use sensible defaults for trackio configuration unless user requests otherwise.**

### Recommended Defaults

```python
import trackio

trackio.init(
    project="qwen-capybara-sft",
    name="baseline-run",             # Descriptive name user will recognize
    space_id="username/trackio",     # Default space: {username}/trackio
    config={
        # Keep config minimal - hyperparameters and model/dataset info only
        "model": "Qwen/Qwen2.5-0.5B",
        "dataset": "trl-lib/Capybara",
        "learning_rate": 2e-5,
        "num_epochs": 3,
    }
)
```

**Key principles:**
- **Space ID**: Use `{username}/trackio` with "trackio" as default space name
- **Run naming**: Unless otherwise specified, name the run in a way the user will recognize
- **Config**: Keep minimal - don't automatically capture job metadata unless requested
- **Grouping**: Optional - only use if user requests organizing related experiments

## Grouping Runs (Optional)

The `group` parameter helps organize related runs together in the dashboard sidebar. This is useful when user is running multiple experiments with different configurations but wants to compare them together:

```python
# Example: Group runs by experiment type
trackio.init(project="my-project", run_name="baseline-run-1", group="baseline")
trackio.init(project="my-project", run_name="augmented-run-1", group="augmented")
trackio.init(project="my-project", run_name="tuned-run-1", group="tuned")
```

Runs with the same group name can be grouped together in the sidebar, making it easier to compare related experiments. You can group by any configuration parameter:

```python
# Hyperparameter sweep - group by learning rate
trackio.init(project="hyperparam-sweep", run_name="lr-0.001-run", group="lr_0.001")
trackio.init(project="hyperparam-sweep", run_name="lr-0.01-run", group="lr_0.01")
```

## Environment Variables for Jobs

You can configure trackio using environment variables instead of passing parameters to `trackio.init()`. This is useful for managing configuration across multiple jobs.



**`HF_TOKEN`**
Required for creating Spaces and writing to datasets (passed via `secrets`):
```python
hf_jobs("uv", {
    "script": "...",
    "secrets": {
        "HF_TOKEN": "$HF_TOKEN"  # Enables Space creation and Hub push
    }
})
```

### Example with Environment Variables

```python
hf_jobs("uv", {
    "script": """
# Training script - trackio config from environment
import trackio
from datetime import datetime

# Auto-generate run name
timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M")
run_name = f"sft_qwen25_{timestamp}"

# Project and space_id can come from environment variables
trackio.init(run_name=run_name, group="SFT")

# ... training code ...
trackio.finish()
""",
    "flavor": "a10g-large",
    "timeout": "2h",
    "secrets": {"HF_TOKEN": "$HF_TOKEN"}
})
```

**When to use environment variables:**
- Managing multiple jobs with same configuration
- Keeping training scripts portable across projects
- Separating configuration from code

**When to use direct parameters:**
- Single job with specific configuration
- When clarity in code is preferred
- When each job has different project/space

## Viewing the Dashboard

After starting training:
1. Navigate to the Space: `https://huggingface.co/spaces/username/trackio`
2. The Gradio dashboard shows all tracked experiments
3. Filter by project, compare runs, view charts with smoothing

## MCP Server for Programmatic Access

Trackio provides an MCP (Model Context Protocol) server that enables Claude to query training metrics programmatically. This allows for intelligent monitoring, loss curve analysis, and automated recommendations.

### Enabling the MCP Server

Start the Trackio dashboard with MCP server enabled:

```bash
# CLI
trackio show --mcp-server

# Or via Python
import trackio
trackio.show(mcp_server=True)

# Or via environment variable
export GRADIO_MCP_SERVER=True && trackio show
```

The server runs at `http://127.0.0.1:7860/gradio_api/mcp/` by default.

### MCP Client Configuration

Add to your Claude MCP configuration:

```json
{
  "mcpServers": {
    "trackio": {
      "url": "http://127.0.0.1:7860/gradio_api/mcp/"
    }
  }
}
```

### Available MCP Tools

| Tool | Description |
|------|-------------|
| `get_all_projects` | List all tracked projects |
| `get_runs_for_project` | List runs within a project |
| `get_metrics_for_run` | Get available metrics for a run |
| `get_metric_values` | Fetch metric history (loss curves) |
| `get_project_summary` | Project overview with run count |
| `get_run_summary` | Run details with config and metrics |

**See:** `references/trackio_mcp_monitoring.md` for complete MCP monitoring guide including loss curve interpretation and troubleshooting patterns.

## Recommendation

- **Trackio**: Best for real-time monitoring during long training runs
- **Trackio MCP**: Best for programmatic monitoring and AI-assisted analysis
- **Weights & Biases**: Best for team collaboration, requires account
