# Heketi 开发介绍

Heketi提供了RESTful管理接口，可用于管理GlusterFS卷的生命周期。 Heketi的目标是提供一种在多个存储群集中创建，列出和删除GlusterFS卷的简单方法。 Heketi将智能地管理群集中整个磁盘的分配，创建和删除。 在满足任何请求之前，Heketi首先需要了解集群的拓扑（topologies ）也就是需要配置topologies.json文件 。 此json文件将数据资源组织为以下内容：群集、节点、设备的归属、以及块的归属。

# 开发
要与Heketi服务进行通信，您将需要使用客户端库或直接与REST端点进行通信。 Heketi目前支持以下客户端库：Go，Python。

# Go 客户端库

## 以下是一个使用Go客户端库的小例子仅供参考
    package main

    import (
        "fmt"
        client"github.com/heketi/heketi/client/api/go-client"
        // api"github.com/heketi/heketi/pkg/glusterfs/api"
    )

    func main() {

        type Options struct{
            Url string
            User string
            Key string
        }
        options := Options{Url:"http://192.168.3.126:8080",User:"admin",Key:"My Secret"}

        // Create a client
        clientObj := client.NewClient(options.Url, options.User, options.Key)
        
        // Get a cluster id
        clusterListRes,_ := clientObj.ClusterList()
        if len((*clusterListRes).Clusters) == 0{
            fmt.Println("don't have a cluster")
            return
        }
        clusterList := (*clusterListRes).Clusters
        clusterInfo,_ := clientObj.ClusterInfo(clusterList[0])

        //Create node
        // nodeReq := &api.NodeAddRequest{}
        // n := 0
        // nodeReq.ClusterId = clusterInfo.Id
        // nodeReq.Hostnames.Manage = []string{"cent" + fmt.Sprintf("%v", 1)}
        // nodeReq.Hostnames.Storage = []string{"storage" + fmt.Sprintf("%v", 1)}
        // nodeReq.Zone = n + 1

        // // Add node
        // node, err := clientObj.NodeAdd(nodeReq)
        // fmt.Println(node,err)

        for _,node:= range clusterInfo.Nodes {
            nodeInfo ,_:= clientObj.NodeInfo(node)
            fmt.Println(*nodeInfo)
        }  
    }

想要了解更多有段Heketi客户端API代码细节的可以参考如下链接：
- **Source code:** https://github.com/heketi/heketi/tree/master/client/api/go-client
- **Doc:**[https://godoc.org/github.com/heketi/heketi/client/api/go-client](https://godoc.org/github.com/heketi/heketi/client/api/go-client)

# 快速搭建开发环境

在开发Heketi 客户端代码时，我们需要一个模拟Heketi Server的环境来做开发。在这种模式下，Heketi无需与任何存储节点进行通信，而是模拟通信，同时仍支持所有REST调用并保持状态，这种最简单的方式就是运行一个Heketi server的docker容器。 具体命令如下：

    docker run -d -p 8080:8080 heketi/heketi
    curl http://localhost:8080/hello
    Hello from Heketi

# 认证模式

Heketi使用基于IETF提议的JSON Web令牌（JWT）标准的无状态身份验证模型。 如规范所指定，JWT令牌具有一组声明，可以将这些声明添加到令牌中以确定其正确性。 Heketi要求使用以下标准声明：

- **iss：** Issuer. Heketi支持两种issuers的方式 
    - admin: 对所有的API都有权限
    - user: 仅仅对卷相关API有权限
- **iat:** Issued-at-time
- **exp:** Time when the token should expire

并且用户可以遵循Atlassian所述的模型进行定制：
- **qsh:** URL Tampering prevention.

Heketi支持使用HMAC SHA-256算法加密的令牌签名，该算法由规范指定为HS256。