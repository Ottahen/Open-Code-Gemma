# Clone latest llama.cpp (includes merged PR #21326)
git clone --depth 50 https://github.com/ggml-org/llama.cpp.git /tmp/llama-cpp-build
cd /tmp/llama-cpp-build

# Cherry-pick the tokenizer fix (PR #21343, not yet merged)
git fetch origin pull/21343/head:pr-21343
git cherry-pick pr-21343 --no-commit

# Build with Metal (Apple GPU) support
cmake -B build -DGGML_METAL=ON -DLLAMA_CURL=ON
cmake --build build --config Release -j$(sysctl -n hw.ncpu) -- llama-server

# Verify
./build/bin/llama-server --version
