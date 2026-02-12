# Bedrock Integration Worklog

**Date:** 2026-02-12
**Objective:** Add AWS Bedrock as a provider for Anthropic Claude models in ClawRouter

---

## Summary

Successfully integrated AWS Bedrock with 3 Claude models using cross-region inference profiles. No upstream LiteLLM sync required - all Bedrock infrastructure already present in codebase.

---

## Models Added

| Model Name | Bedrock Inference Profile | Tier |
|------------|---------------------------|------|
| `bedrock/claude-opus-4-6` | `us.anthropic.claude-opus-4-6-v1` | TOP |
| `bedrock/claude-sonnet-4-5` | `us.anthropic.claude-sonnet-4-5-20250929-v1:0` | MID |
| `bedrock/claude-sonnet-4` | `us.anthropic.claude-sonnet-4-20250514-v1:0` | MID |

---

## Files Modified

### 1. `litellm/proxy/proxy_config.yaml`
**Changes:** Added 3 Bedrock model entries with US cross-region inference profiles

```yaml
- model_name: bedrock/claude-opus-4-6
  litellm_params:
    model: bedrock/us.anthropic.claude-opus-4-6-v1
    aws_region_name: os.environ/AWS_REGION_NAME

- model_name: bedrock/claude-sonnet-4-5
  litellm_params:
    model: bedrock/us.anthropic.claude-sonnet-4-5-20250929-v1:0
    aws_region_name: os.environ/AWS_REGION_NAME

- model_name: bedrock/claude-sonnet-4
  litellm_params:
    model: bedrock/us.anthropic.claude-sonnet-4-20250514-v1:0
    aws_region_name: os.environ/AWS_REGION_NAME
```

### 2. `models.yaml`
**Changes:** Added `bedrock` provider to both `provider_models` and `tier_candidates`

```yaml
# Provider configuration
bedrock:
  key_env: AWS_ACCESS_KEY_ID
  extra_env:
    - AWS_SECRET_ACCESS_KEY
    - AWS_REGION_NAME
  models:
    - name: bedrock/claude-opus-4-6
      id: bedrock/us.anthropic.claude-opus-4-6-v1
    - name: bedrock/claude-sonnet-4-5
      id: bedrock/us.anthropic.claude-sonnet-4-5-20250929-v1:0
    - name: bedrock/claude-sonnet-4
      id: bedrock/us.anthropic.claude-sonnet-4-20250514-v1:0

# Tier candidate (top tier)
top:
  - provider: bedrock
    model: bedrock/us.anthropic.claude-opus-4-6-v1
    display: "Claude-Opus-4.6 (Bedrock)"
    cost: "$15.00/M"
```

### 3. `.env.example`
**Changes:** Added AWS credential placeholders

```bash
# AWS Bedrock
AWS_ACCESS_KEY_ID = ""
AWS_SECRET_ACCESS_KEY = ""
AWS_REGION_NAME = "us-west-1"
```

---

## Key Technical Insights

### Bedrock Support Already Exists
- **No fork sync needed**: LiteLLM codebase already includes full Bedrock support including Claude Opus 4.6
- **Comprehensive infrastructure**: `litellm/llms/bedrock/` contains complete implementation for chat, embeddings, image gen, etc.
- **Provider coverage**: Supports Anthropic, Meta, Amazon, Mistral, AI21, Cohere, DeepSeek, Qwen, and more via Bedrock

### Cross-Region Inference Profiles
- **Critical requirement**: Direct model IDs fail with error: `"Invocation of model ID anthropic.claude-sonnet-4-5-20250929-v1:0 with on-demand throughput isn't supported"`
- **Solution**: Use inference profile IDs (e.g., `us.anthropic.claude-opus-4-6-v1`)
- **Profile types available**:
  - **Global**: `global.anthropic.claude-sonnet-4-5-20250929-v1:0` (routes to any AWS region globally)
  - **Regional (US)**: `us.anthropic.claude-sonnet-4-5-20250929-v1:0` (routes within US regions only)
  - **Other regions**: EU, AP also supported

### AWS Authentication
- **Required env vars**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME`
- **Optional vars**: `AWS_SESSION_TOKEN` (for temporary credentials)
- **Region used**: `us-west-1` (configurable via `AWS_REGION_NAME`)
- **Supported auth methods**: Access keys, IAM roles, STS assume role, web identity tokens, AWS profiles

---

## Server Configuration

### Port Changes
- **Original plan**: Port 4141 (per CLAUDE.md)
- **Actual deployment**: Port 4242 (4141 was occupied by Node.js process)
- **Legacy ports killed**: 4000, 8000 (deprecated instances)

### Startup Command
```bash
source .venv/bin/activate
litellm --config litellm/proxy/proxy_config.yaml --port 4242
```

### Health Check
```bash
curl http://localhost:4242/health
```

---

## Verification Results

### Health Status
All 3 Bedrock models: ✅ **HEALTHY**

```json
{
  "healthy_endpoints": [
    "us.anthropic.claude-opus-4-6-v1",
    "us.anthropic.claude-sonnet-4-5-20250929-v1:0",
    "us.anthropic.claude-sonnet-4-20250514-v1:0"
  ],
  "unhealthy_endpoints": []
}
```

### Live Test
```bash
curl http://localhost:4242/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{"model":"bedrock/claude-opus-4-6","messages":[{"role":"user","content":"What is 2+2?"}]}'

# Response: "Four."
```

---

## Troubleshooting Log

### Issue 1: Model ID Format
- **Problem**: Used bare model IDs initially (e.g., `anthropic.claude-sonnet-4-5-20250929-v1:0`)
- **Error**: `"Invocation of model ID ... with on-demand throughput isn't supported"`
- **Solution**: Switched to US cross-region inference profiles with `us.` prefix

### Issue 2: Incomplete Profile IDs
- **Problem**: Used shortened IDs (e.g., `us.anthropic.claude-sonnet-4-5-v1`)
- **Error**: `"LLM Provider NOT provided"`
- **Solution**: Used full version strings (e.g., `us.anthropic.claude-sonnet-4-5-20250929-v1:0`)

### Issue 3: Port Conflicts
- **Problem**: Port 4141 already in use
- **Solution**: Deployed to port 4242 instead

---

## Future Considerations

### For Replication at Scale

1. **Credential Management**:
   - Use AWS Secrets Manager or SSM Parameter Store for production
   - Consider IAM roles for EC2/ECS deployments (eliminates credential rotation)
   - Implement least-privilege policies (Bedrock invoke-only permissions)

2. **Region Selection**:
   - **US profiles**: Best for US-based workloads (lower latency)
   - **Global profiles**: Consider for international deployments (~10% cost savings)
   - **Regional compliance**: EU/AP profiles for data residency requirements

3. **Model Availability**:
   - Verify inference profile availability in target source regions
   - Claude Opus 4.6: Supports `us-west-1` (used here) + `us-east-1`, `us-east-2`, `us-west-2`, `ca-central-1`, `ca-west-1`
   - Claude Sonnet 4/4.5: Supports `us-east-1`, `us-east-2`, `us-west-2` for US profiles

4. **Cost Optimization**:
   - Global profiles offer ~10% savings vs geographic profiles
   - Cross-region data transfer costs may apply
   - Monitor with AWS Cost Explorer tagged by model/profile

5. **Auto-Router Integration**:
   - Bedrock Opus 4.6 already added as top-tier candidate in `models.yaml`
   - Will be suggested by `setup.sh` when AWS credentials detected
   - Test tier routing with `[high]` tag override

6. **OpenClaw Plugin**:
   - Need to add AWS credential mapping in `openclaw-plugin/src/keys.ts`
   - Map OpenClaw auth profiles → `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME`

---

## References

- [AWS Bedrock Inference Profiles Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html)
- [AWS Blog: Global Cross-Region Inference for Claude Sonnet 4.5](https://aws.amazon.com/blogs/machine-learning/unlock-global-ai-inference-scalability-using-new-global-cross-region-inference-on-amazon-bedrock-with-anthropics-claude-sonnet-4-5/)
- [AWS Blog: Cross-Region Inference Announcement](https://aws.amazon.com/about-aws/whats-new/2025/09/amazon-bedrock-global-cross-region-inference-anthropic-claude-sonnet-4/)

---

## Engineer Notes

- **No code changes required** - pure configuration
- **Existing Bedrock support is mature** - includes retry logic, streaming, tool calling, vision
- **Test coverage exists** - `tests/test_litellm/router_strategy/test_auto_router.py` validates routing
- **Profile IDs are version-specific** - expect updates as new model versions release
- **Cost transparency**: Bedrock pricing visible in AWS Cost Explorer with model-level granularity
