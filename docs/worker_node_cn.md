Worker Node 完整部署教程
第一阶段：环境准备
推荐操作系统: Ubuntu 
第一阶段：环境准备
1.1 安装 NVIDIA Container Toolkit
1.2 安装 Go 环境
第二阶段：部署 AIL2-ComputeNet Worker
2.1 下载和编译
git clone https://github.com/AIL2Lab/AIL2-ComputeNet.git
cd AIL2-ComputeNet
# 下载依赖
go mod download
go mod tidy
# 编译 Worker Node
cd host
go build -o host-node main.go
# 生成 Worker 节点配置
./host-node -init worker
控制台会打印
Encode private key
Encode public key
Transform Peer ID
修改生成的worker.json文件，将.App.PeersCollect.Enabled修改为true
在配置文件中找到 "Bootstrap" 字段，修改为：
"Bootstrap": [
    "/ip4/122.99.183.54/tcp/6001/p2p/16Uiu2HAmRTpigc7jAbsLndB2xDEBMAXLb887SBEFhfdJeEJNtqRM",
    "/ip4/8.219.75.114/tcp/6001/p2p/16Uiu2HAmS4CErxrmPryJbbEX2HFQbLK8r8xCA5rmzdSU59rHc9AF"
  ]
启动 Worker Node
./host-node -config worker.json
curl -s http://localhost:7002/api/v0/id | jq .
curl -s http://localhost:7002/api/v0/id | jq .
curl -s http://localhost:7002/api/v0/swarm/peers | jq .
curl -s http://localhost:7002/api/v0/pubsub/peers | jq .
如果有返回内容，说明成功了！
接下来直接使用 Ollama
curl -fsSL https://ollama.com/install.sh | sh
验证安装
ollama --version
启动 Ollama 服务
ollama serve > /tmp/ollama.log 2>&1 &
拉取模型
ollama pull qwen2:1.5b
测试模型
ollama run qwen2:1.5b "Hello! What is 2+2?"

创建 OpenAI API 适配器
安装依赖
pip3 install flask requests
创建适配器
cat > ~/ollama-adapter.py << 'EOF'
#!/usr/bin/env python3
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)
OLLAMA_URL = "http://localhost:11434"

@app.route('/v1/models', methods=['GET'])
def list_models():
    return jsonify({
        "object": "list",
        "data": [{"id": "Qwen2-1.5B", "object": "model"}]
    })

@app.route('/v1/chat/completions', methods=['POST'])
def chat_completions():
    data = request.json
    messages = data.get('messages', [])
    
    prompt = ""
    for msg in messages:
        role = msg.get('role', 'user')
        content = msg.get('content', '')
        if role == 'system':
            prompt += f"System: {content}\n"
        elif role == 'user':
            prompt += f"User: {content}\n"
        elif role == 'assistant':
            prompt += f"Assistant: {content}\n"
    prompt += "Assistant: "
    
    try:
        resp = requests.post(
            f"{OLLAMA_URL}/api/generate",
            json={
                "model": "qwen2:1.5b",
                "prompt": prompt,
                "stream": False,
                "options": {
                    "temperature": data.get('temperature', 0.7),
                    "num_predict": data.get('max_tokens', 100)
                }
            },
            timeout=60
        )
        
        if resp.status_code == 200:
            result = resp.json()
            return jsonify({
                "choices": [{
                    "index": 0,
                    "message": {
                        "role": "assistant",
                        "content": result.get('response', '').strip()
                    },
                    "finish_reason": "stop"
                }],
                "usage": {
                    "prompt_tokens": result.get('prompt_eval_count', 0),
                    "completion_tokens": result.get('eval_count', 0),
                    "total_tokens": result.get('prompt_eval_count', 0) + result.get('eval_count', 0)
                }
            })
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    print("=" * 60)
    print("Ollama OpenAI API Adapter")
    print("Listening on: http://0.0.0.0:8000")
    print("Model: qwen2:1.5b")
    print("=" * 60)
    app.run(host='0.0.0.0', port=8000)

EOF

chmod +x ~/ollama-adapter.py
启动适配器
nohup python3 ~/ollama-adapter.py > /tmp/adapter.log 2>&1 &

注册模型到 Worker Node
1. 获取 Worker Peer ID
WORKER_PEER_ID=$(curl -s http://localhost:7002/api/v0/id | jq -r '.peer_id')
echo "Worker Peer ID: $WORKER_PEER_ID"
# 2. 注册模型
curl -X POST http://localhost:7002/api/v0/ai/project/register \
  -H "Content-Type: application/json" \
  -d '{
    "project": "DecentralGPT",
    "models": [{
      "model": "Qwen2-1.5B",
      "api": "http://localhost:8000/v1/chat/completions",
      "type": 0,
      "cid": "ollama-qwen2-1.5b"
    }]
  }'

# 应该返回: {"code":0,"message":"ok"}
验证注册
curl -s http://localhost:7002/api/v0/ai/project/peer \
  -H "Content-Type: application/json" \
  -d '{"node_id": "'$WORKER_PEER_ID'"}' | jq .

从 Input Node 测试通信
现在切换到 Input Node 机器
curl -X POST http://localhost:7002/api/v0/chat/completion/proxy \
  -H "Content-Type: application/json" \
  -d '{
    "project": "DecentralGPT",
    "model": "Qwen2-1.5B",
    "messages": [
      {"role": "user", "content": "你好！请用一句话介绍人工智能。"}
    ],
    "stream": false
  }'
