### uv / cmake を入れる
```
curl -LsSf https://astral.sh/uv/install.sh | sh
uv tool update-shell
source ~/.zshrc
source ~/.zshenv

brew install cmake
```

### llama.cpp Metal Build
```
mkdir -m 755 -p ~/shyme
cd ~/shyme
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

cmake -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DGGML_METAL=ON \
  -DGGML_METAL_EMBED_LIBRARY=ON

cmake --build build --config Release -j $(sysctl -n hw.ncpu)
```

---

**以下、過去に試したが、不安定で使い物にならなかったもの**

### uv を入れる
```
curl -LsSf https://astral.sh/uv/install.sh | sh
uv tool update-shell
source ~/.zshrc
source ~/.zshenv
```

### MLX をビルドするために MetalToolchain を入れる
```
xcodebuild -downloadComponent MetalToolchain
```

### mlx-lmのインストール（本家mlxを使用。PrismMLフォークはうまく動かなかった。）
```
uv python install 3.12

uv tool install vllm-mlx --python 3.12
or
uv tool install mlx-lm --python 3.12
```

### Ternary Bonsai 27B を 4 bit KV キャッシュ有効の OpenAI 互換サーバーで起動
```
vllm-mlx serve "t-kawata/Ternary-Bonsai-27B-mlx-2bit" \
    --port 8081 \
    --kv-cache-quantization \
    --kv-cache-quantization-bits 4 \
    --kv-cache-quantization-group-size 64 \
    --max-tokens 262144 \
    --max-request-tokens 262144 \
    --default-temperature 0.7 \
    --default-top-p 0.95 \
    --default-top-k 20
or
mlx_lm.server --model "t-kawata/Ternary-Bonsai-27B-mlx-2bit" --port 8081
```

### Bifrost ディレクトリ
```
mkdir -p ~/shyme/bifrost
```

### Bifrost 設定
```
cat <<EOF > ~/shyme/bifrost/config.json
{
  "\$schema": "https://www.getbifrost.ai/schema",
  "client": {
    "enable_logging": false,
    "enforce_auth_on_inference": false,
    "drop_excess_requests": false
  },
  "providers": {
    "deepseek-anthropic": {
      "keys": [
        {
          "name": "deepseek-key",
          "value": "env.DEEPSEEK_KEY",
          "models": ["deepseek-v4-flash", "deepseek-v4-pro"],
          "weight": 1.0
        }
      ],
      "network_config": {
        "base_url": "https://api.deepseek.com/anthropic",
        "default_request_timeout_in_seconds": 120
      },
      "custom_provider_config": {
        "base_provider_type": "anthropic"
      }
    },
    "lmpx": {
      "keys": [
        {
          "name": "lmpx-key",
          "value": "dummy-key",
          "models": ["ternary-bonsai-27b"],
          "weight": 1.0,
          "aliases": {
            "ternary-bonsai-27b": "t-kawata/Ternary-Bonsai-27B-mlx-2bit"
          }
        }
      ],
      "network_config": {
        "base_url": "http://127.0.0.1:8081",
        "default_request_timeout_in_seconds": 600
      },
      "custom_provider_config": {
        "base_provider_type": "openai",
        "allowed_requests": {
          "chat_completion": true,
          "chat_completion_stream": true
        },
        "request_path_overrides": {
          "chat_completion": "/v1/chat/completions",
          "chat_completion_stream": "/v1/chat/completions"
        }
      }
    }
  },
  "config_store": { "enabled": false }
}
EOF
```

### Bifrost 起動
```
DEEPSEEK_KEY=sk-c???????????????????? npx -y @maximhq/bifrost --transport-version v1.6.4 -app-dir ~/shyme/bifrost
```

### 動作確認
```
curl http://localhost:8080/anthropic/v1/messages \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "ternary-bonsai-27b",
    "max_tokens": 256,
    "messages": [
      {"role": "user", "content": "こんにちは、動作確認です"}
    ],
    "stream": false
  }'

curl http://localhost:8080/anthropic/v1/messages \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "deepseek-anthropic/deepseek-v4-flash",
    "max_tokens": 256,
    "messages": [
      {"role": "user", "content": "こんにちは、動作確認です"}
    ],
    "stream": false
  }'
```
