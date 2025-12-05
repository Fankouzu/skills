# Trackio MCP Server: Real-Time Training Monitoring

The Trackio MCP server enables Claude to fetch and analyze training metrics in real-time, providing intelligent interpretation of loss curves, detection of training issues, and actionable recommendations.

## Overview

When training jobs run with Trackio logging, metrics are stored in a database that the Trackio MCP server can query. This allows:

- **Live monitoring** of training progress
- **Loss curve analysis** to detect overfitting, divergence, or plateaus
- **Run comparison** across experiments
- **Intelligent recommendations** based on metric patterns

## MCP Server Setup

### Starting the Trackio MCP Server

The Trackio dashboard can expose an MCP server for querying metrics:

**Option 1: CLI**
```bash
trackio show --mcp-server
```

**Option 2: Python**
```python
import trackio
trackio.show(mcp_server=True)
```

**Option 3: Environment Variable**
```bash
export GRADIO_MCP_SERVER=True
trackio show
```

The server runs at `http://127.0.0.1:7860/gradio_api/mcp/` by default.

### MCP Client Configuration

Add to your MCP configuration file (e.g., `.claude/settings.json` or Claude Desktop config):

```json
{
  "mcpServers": {
    "trackio": {
      "url": "http://127.0.0.1:7860/gradio_api/mcp/"
    }
  }
}
```

## Available MCP Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `get_all_projects` | List all tracked projects | None |
| `get_runs_for_project` | List runs within a project | `project` (string) |
| `get_metrics_for_run` | Get available metrics for a run | `project`, `run` (strings) |
| `get_metric_values` | Fetch metric history with timestamps/steps | `project`, `run`, `metric` (strings) |
| `get_project_summary` | Summary with run count and recent activity | `project` (string) |
| `get_run_summary` | Run details including config and metrics | `project`, `run` (strings) |

## Monitoring Workflow

### 1. Check Available Projects

```
Use get_all_projects to see what training projects exist.
```

### 2. List Runs in a Project

```
Use get_runs_for_project with project="my-training" to see all runs.
```

### 3. Get Run Summary

```
Use get_run_summary with project="my-training", run="baseline-run"
to see config, metrics, and status.
```

### 4. Fetch Loss Curves

```
Use get_metric_values with project="my-training", run="baseline-run", metric="train/loss"
to get the full loss history for analysis.
```

## Loss Curve Interpretation Guide

When analyzing loss curves from `get_metric_values`, look for these patterns:

### Healthy Training
- **Pattern**: Smooth, consistent decrease in loss
- **Train loss**: Steadily decreasing
- **Eval loss**: Decreasing, tracking train loss
- **Action**: Continue training, monitor for overfitting

### Overfitting
- **Pattern**: Train loss ↓, Eval loss ↑ (diverging)
- **Detection**: Eval loss starts increasing while train loss continues decreasing
- **Actions**:
  - Stop training at lowest eval loss (early stopping)
  - Increase regularization (dropout, weight decay)
  - Reduce model capacity or use smaller LoRA rank
  - Add more training data

### Learning Rate Too High
- **Pattern**: Loss spikes, erratic behavior, or divergence to NaN
- **Detection**: Sudden jumps in loss, unstable training
- **Actions**:
  - Reduce learning rate by 2-10x
  - Use learning rate warmup
  - Enable gradient clipping

### Learning Rate Too Low
- **Pattern**: Loss decreases very slowly or plateaus early
- **Detection**: Minimal progress after many steps
- **Actions**:
  - Increase learning rate by 2-5x
  - Use learning rate scheduler with warmup

### Loss Plateau
- **Pattern**: Loss stops decreasing, stays flat
- **Detection**: No improvement for many steps
- **Possible causes**:
  - Learning rate too low → increase LR
  - Model capacity reached → use larger model
  - Data quality issues → check dataset
  - Need more epochs → continue training

### Gradient Explosion
- **Pattern**: Loss suddenly becomes NaN or infinity
- **Detection**: `train/loss` shows NaN values
- **Actions**:
  - Reduce learning rate significantly
  - Enable gradient clipping: `max_grad_norm=1.0`
  - Check for data issues (NaN in inputs)
  - Use mixed precision carefully

## Metric Reference

Common metrics logged by TRL trainers:

| Metric | Description | Healthy Range |
|--------|-------------|---------------|
| `train/loss` | Training loss | Decreasing over time |
| `eval/loss` | Evaluation loss | Tracks train loss, slightly higher |
| `train/learning_rate` | Current LR | Per schedule |
| `train/epoch` | Current epoch | 0 to num_epochs |
| `train/global_step` | Total steps | Increasing |
| `train/grad_norm` | Gradient norm | < 10 typically |

### DPO-Specific Metrics
| Metric | Description |
|--------|-------------|
| `train/rewards/chosen` | Reward for chosen responses |
| `train/rewards/rejected` | Reward for rejected responses |
| `train/rewards/margins` | Difference (chosen - rejected) |
| `train/rewards/accuracies` | % where chosen > rejected |

**Healthy DPO training**: `rewards/margins` increasing, `rewards/accuracies` > 0.5 and improving.

### GRPO-Specific Metrics
| Metric | Description |
|--------|-------------|
| `train/reward` | Average reward from reward model |
| `train/kl` | KL divergence from reference |

## Example Analysis Session

**User**: "Check on my training run"

**Claude workflow**:

1. **Get projects**: `get_all_projects()`
2. **Find runs**: `get_runs_for_project(project="sft-qwen")`
3. **Get summary**: `get_run_summary(project="sft-qwen", run="baseline-v1")`
4. **Fetch loss**: `get_metric_values(project="sft-qwen", run="baseline-v1", metric="train/loss")`
5. **Fetch eval loss**: `get_metric_values(project="sft-qwen", run="baseline-v1", metric="eval/loss")`
6. **Analyze**:
   - Compare train vs eval loss trends
   - Check for divergence (overfitting)
   - Look for spikes or plateaus
   - Assess overall progress

**Example response**:
```
Training Progress for sft-qwen/baseline-v1:

📊 Current Status (step 1500/3000):
- Train loss: 0.82 (started at 2.3, down 64%)
- Eval loss: 0.91 (started at 2.4, down 62%)
- Learning rate: 1.8e-5 (in decay phase)

✅ Training looks healthy:
- Both losses decreasing smoothly
- Eval loss tracking train loss (no overfitting yet)
- Loss reduction rate is good

📈 Projection:
- At current rate, expect final train loss ~0.5-0.6
- Watch eval loss - if it starts increasing, consider early stopping

💡 Recommendation: Continue training. Check again at step 2000.
```

## Comparing Multiple Runs

Use `get_runs_for_project` to list runs, then fetch metrics for each to compare:

```
For project "hyperparam-sweep":
- Run "lr-1e-4": Final loss 0.65, eval 0.72
- Run "lr-2e-5": Final loss 0.58, eval 0.61 ✓ Best
- Run "lr-1e-5": Final loss 0.71, eval 0.69

Recommendation: lr-2e-5 achieved lowest eval loss with good generalization.
```

## Troubleshooting

### MCP Server Not Responding
- Ensure Trackio dashboard is running with `--mcp-server` flag
- Check server is accessible at configured URL
- Verify no firewall blocking port 7860

### No Metrics Available
- Training may not have started logging yet (wait 30-60 seconds)
- Check `report_to="trackio"` is set in training config
- Verify `trackio.init()` was called with correct project/run names

### Metrics Not Updating
- Trackio syncs every ~5 minutes to HF Dataset
- For real-time local metrics, query the local SQLite DB
- Check training job is still running

## Integration with Training Workflow

1. **Before training**: Set up Trackio in training script (see `trackio_guide.md`)
2. **During training**: Use MCP tools to monitor progress
3. **On issues**: Analyze metrics, provide recommendations
4. **After training**: Compare runs, select best checkpoint

This enables a feedback loop where training can be monitored and adjusted based on real metric data rather than waiting for jobs to complete.
