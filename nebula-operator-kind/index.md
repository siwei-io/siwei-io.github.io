# Nebula Operator Kind，一键单机玩转 Nebula K8S 集群



> Nebula-Kind，无需依赖，一键安装尝鲜基于 Nebula Operator 的 K8S Nebula Graph Cluster。
> 
> 注： KIND 是一个 K8S 的 SIG，代表 K8S in Docker。

<!--more-->
## Nebula-Operator-Kind 是什么

Nebula Graph 作为云原生的分布式开源图数据库，有开源的 [K8S Operator](https://github.com/vesoft-inc/nebula-operator) 供大家在 K8S 上方便的通过 CRD 去维护、部署 Nebula Graph 集群。

对于手头没有方便的 K8S 环境的同学，如果想尝鲜、学习 Nebula Graph 的 K8S Operator 的话，可能需要耗费一些精力才能搭起来一整套的控制平面的依赖。

作为一个懒人，我做利用 K8S in Docker([KIND](https://kind.sigs.k8s.io/))，和之前做 [Nebula-Up](https://github.com/wey-gu/nebula-up) 的 shell 脚本架子，快速的搞了一个一键安装工具：Nebula-Operator-Kind

它能直接帮我们：
- 安装 Docker
- 安装 K8S(KIND)
- 安装 PV Provider
- 安装 Nebula-Operator 以及依赖
- 安装 Nebula-Console
- 配置 nodePort 用以一键直连 Nebula 集群
- 安装 kubectl 用来体验 Nebula-Operator 的 CRD 配置

## 如何使用
安装：
```bash
curl -sL nebula-kind.siwei.io/install.sh | bash
```
成功之后：
![install_success](./install_success.webp)

用 `~/.nebula-kind/bin/console` 一行连接集群：
```bash
~/.nebula-kind/bin/console -u user -p password --address=127.0.0.1 --port=30000
```

## 详细信息

Repo 的地址是：https://github.com/wey-gu/nebula-operator-kind ，里边有更多的信息，欢迎大家试用、反馈、PR 哈！



> 题图版权：[Maik Hankemann](https://unsplash.com/photos/a4Gz2DD4dX0) 

