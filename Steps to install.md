# One‑shot setup: OpenCode + Gemma 4 26B on macOS

Copy the entire bash block below and paste it into your terminal. It will:
- Install prerequisites
- Build llama.cpp with Gemma 4 fixes
- Start the model server (in background)
- Install and patch OpenCode
- Create the config file
- Ready to run `opencode`

```bash
#!/bin/bash
set -e

# 1. Prerequisites
brew install git gh cmake bun

# 2. Build llama.cpp with Gemma 4 fixes
git clone --depth 50 https://github.com/ggml-org/llama.cpp.git /tmp/llama-cpp-build
cd /tmp/llama-cpp-build
git fetch origin pull/21343/head:pr-21343
git cherry-pick pr-21343 --no-commit
cmake -B build -DGGML_METAL=ON -DLLAMA_CURL=ON
cmake --build build --config Release -j$(sysctl -n hw.ncpu) -- llama-server

# 3. Start llama-server (32k context, port 8089, background)
./build/bin/llama-server \
  -hf ggml-org/gemma-4-26B-A4B-it-GGUF:Q4_K_M \
  --port 8089 -ngl 99 -c 32768 --jinja &
sleep 5
curl http://127.0.0.1:8089/health   # should show {"status":"ok"}

# 4. Install stock OpenCode
curl -fsSL https://opencode.ai/install -o /tmp/opencode-install.sh
bash /tmp/opencode-install.sh
source ~/.zshrc
brew uninstall opencode 2>/dev/null || true

# 5. Build OpenCode with PR #16531 (tool‑call compat)
git clone https://github.com/anomalyco/opencode.git /tmp/opencode-build
cd /tmp/opencode-build
gh pr checkout 16531
bun install
cd packages/opencode
bun run build -- --single --skip-install
cp ~/.opencode/bin/opencode ~/.opencode/bin/opencode.backup
cp dist/opencode-darwin-arm64/bin/opencode ~/.opencode/bin/opencode
chmod +x ~/.opencode/bin/opencode

# 6. Create opencode.json in current directory
cat > opencode.json << 'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "llama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "llama.cpp (local)",
      "options": {
        "baseURL": "http://127.0.0.1:8089/v1",
        "toolParser": [
          { "type": "raw-function-call" },
          { "type": "json" }
        ]
      },
      "models": {
        "gemma4-26b": {
          "name": "Gemma 4 26B",
          "tool_call": true,
          "limit": {
            "context": 32768,
            "output": 8192
          }
        }
      }
    }
  },
  "model": "llama/gemma4-26b"
}
EOF

echo "✅ Setup complete. Now run: opencode"
