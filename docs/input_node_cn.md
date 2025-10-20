Input Node (输入节点) 完整部署教程
第一步：环境准备
1.1 系统要求
# 推荐配置
操作系统: Ubuntu 20.04/22.04 LTS
CPU: 4核心以上
内存: 8GB RAM
硬盘: 50GB SSD
网络: 公网 IP 地址（必须！）
1.2 安装 Go 环境
第二步：下载和编译项目
git clone https://github.com/AIL2Lab/AIL2-ComputeNet.git
cd AIL2-ComputeNet
2.2 编译 Input Node
# 下载依赖
go mod download
go mod tidy
# 编译
cd host
go build -o host-node main.go
# 生成 Input 节点配置（适用于有公网IP的机器）
./host-node -init input
控制台会打印
Encode private key
Encode public key
Transform Peer ID
打开input.json文件编辑，
在配置文件中找到 "Bootstrap" 字段，修改为：
"Bootstrap": [
    "/ip4/122.99.183.54/tcp/6001/p2p/16Uiu2HAmRTpigc7jAbsLndB2xDEBMAXLb887SBEFhfdJeEJNtqRM",
    "/ip4/8.219.75.114/tcp/6001/p2p/16Uiu2HAmS4CErxrmPryJbbEX2HFQbLK8r8xCA5rmzdSU59rHc9AF"
  ]
保存并退出，然后使用./host-node -config input.json启动节点