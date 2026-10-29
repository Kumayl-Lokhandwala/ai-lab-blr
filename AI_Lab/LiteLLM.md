Let's make this a single service running on your 32 GB MacBook, with LiteLLM exposed over HTTP and RouteLLM selecting the model based on prompt complexity.

I'll use this architecture:

- Port `8090` — your existing API-key gateway.
- Port `8000` — new LiteLLM + RouteLLM service.
- Port `11434` — Ollama on each MacBook, assuming that's your inference runtime.

The new service will expose an OpenAI-compatible `/v1/chat/completions` endpoint. RouteLLM will decide which model tier to use, and LiteLLM will manage requests to the corresponding inference deployment.

One important detail: LiteLLM's built-in load balancing doesn't automatically invoke an arbitrary RouteLLM classifier. We'll explicitly integrate the two rather than assume that configuring both is sufficient.

## 1. Create the service

Use two components:

- LiteLLM Proxy: `4000` — inference deployments and load balancing.
- RouteLLM Server: `6060` — complexity-based routing and the public OpenAI-compatible API.

RouteLLM supports a server mode and uses LiteLLM for model calls.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



### Project structure

```
ai-lab/
├── litellm_config.yaml
├── routellm_config.yaml
├── .env
└── start.sh
```

### `litellm_config.yaml`

Replace the example IPs and model names with your actual deployments. This example assumes Ollama.

```

model_list:
  # Weak / fast tier
  - model_name: ai-lab-weak
    litellm_params:
      model: ollama/llama3.2:3b
      api_base: http://192.168.1.101:11434
      api_key: ollama

  # Strong tier
  - model_name: ai-lab-strong
    litellm_params:
      model: ollama/llama3.1:8b
      api_base: http://192.168.1.102:11434
      api_key: ollama

router_settings:
  routing_strategy: simple-shuffle
  num_retries: 2
  allowed_fails: 2
  cooldown_time: 30

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY

```

This config defines one deployment per tier. If you have multiple interchangeable replicas of either tier, add another entry with the same `model_name` and a different `api_base` to enable load balancing within that tier.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.litellm.ai\&sz=32)

liteLLM



### `routellm_config.yaml`

RouteLLM's built-in `mf` router uses a learned preference model to choose between a weak and strong model. It is not a generic heuristic complexity classifier, so its threshold should be calibrated against your workloads.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



```

mf:
  # Start with RouteLLM's default router configuration.
  # Model pair and threshold are supplied at server startup.

```

### `.env`

```
LITELLM_MASTER_KEY=sk-your-long-random-secret
OPENAI_API_KEY=sk-your-routellm-client-key
```

### `start.sh`

```
#!/bin/bash
set -e

source .venv/bin/activate
set -a
source .env
set +a

# Start LiteLLM inference gateway
litellm \
  --config litellm_config.yaml \
  --host 127.0.0.1 \
  --port 4000 &
LITELLM_PID=$!

cleanup() {
  kill "$LITELLM_PID" 2>/dev/null || true
}
trap cleanup EXIT INT TERM

# Start RouteLLM API service
python -m routellm.openai_server \
  --routers mf \
  --config routellm_config.yaml \
  --strong-model openai/ai-lab-strong \
  --weak-model openai/ai-lab-weak \
  --port 6060
```

Important integration note: the starter above defines the intended components, but RouteLLM must be configured to send its strong/weak model calls to your local LiteLLM proxy, including the proxy URL and its authentication key. The exact supported flags/configuration depend on the installed RouteLLM version. Do not assume that setting the model names alone will connect it to port `4000`; verify this before using the script as-is.

## 2. Install the dependencies

On your 32 GB MacBook:

```
mkdir ai-lab
cd ai-lab

python3 -m venv .venv
source .venv/bin/activate

pip install 'litellm[proxy]' 'routellm[serve]'
```

Check the available RouteLLM server options:

```
python -m routellm.openai_server --help
```

Then configure the LiteLLM proxy endpoint in RouteLLM according to the installed version's supported options. Keep port `4000` bound to localhost and expose only the RouteLLM-facing service to your trusted LAN.

## 3. Test the API

Once both services are running and RouteLLM is correctly connected to LiteLLM:

```
curl http://localhost:6060/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "router-mf-0.11593",
    "messages": [
      {
        "role": "user",
        "content": "Explain what a REST API is in simple terms."
      }
    ]
  }'
```

The model string is an example threshold, not a calibrated value for your models. RouteLLM documents this `router-mf-THRESHOLD` format.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



Your existing API gateway on port `8090` can then forward authenticated requests to the service on port `6060`.

Before you run this, one thing matters: the model names, LAN IPs, and how RouteLLM reaches LiteLLM need to match your actual environment. If you send me the two model names and tell me whether they're served by Ollama or MLX, I can tailor the integration to those exact endpoints. 