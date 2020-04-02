
# Heketi 调用封装

### 包依赖
    import (
        "fmt"
        "errors"
        client"github.com/heketi/heketi/client/api/go-client"
        api"github.com/heketi/heketi/pkg/glusterfs/api"
    )

### Heketi属性
    type Heketi struct{
        Url string
        User string
        Key string
        ClientObj *client.Client
    }
### 客户端实例化接口
    func(self *Heketi)NewClient(){
        // Create a client object
        clientObj := client.NewClient(self.Url, self.User, self.Key)
        self.ClientObj = clientObj
    }

### 创建一个集群接口
    func(self *Heketi)CreateCluster(block bool,file bool)(string,error){
        // create a cluster
        clusterCreateReq := &api.ClusterCreateRequest{}
        clusterCreateReq.ClusterFlags.Block = true
        clusterCreateReq.ClusterFlags.File = true
        clusterInfoRep,err := self.ClientObj.ClusterCreate(clusterCreateReq)
        return clusterInfoRep.Id,err
    }

### 集群ID列表返回
    func(self *Heketi)ClusterList()([]string,error){
        // Get  cluster list
        clusterListRes,err := self.ClientObj.ClusterList()
        if err != nil{
            return nil,err
        }
        if len((*clusterListRes).Clusters) == 0{
            fmt.Println("don't have a cluster")
            return []string{},errors.New("don't have a cluster")
        }
        clusterList := (*clusterListRes).Clusters
        return clusterList,nil
    }

### 集群信息获取
    func(self *Heketi)ClusterInfo(clusterId string)([]string,[]string,[]string,error){
        clusterInfoRes,err := self.ClientObj.ClusterInfo(clusterId)
        return clusterInfoRes.Nodes,clusterInfoRes.Volumes ,clusterInfoRes.BlockVolumes ,err
    }

### 集群中添加一个节点
    func(self *Heketi)AddNode(clusterId string,zone int,hostnameManage []string,hostnamesStorage []string,Tags map[string]string)(*api.NodeInfoResponse,error){
        // Create node
        nodeReq := &api.NodeAddRequest{}
        nodeReq.ClusterId = clusterId
        nodeReq.Hostnames.Manage = []string{"cent" + fmt.Sprintf("%v", zone)}
        nodeReq.Hostnames.Storage = []string{"storage" + fmt.Sprintf("%v", zone)}
        nodeReq.Zone = zone + 1
        nodeReq.Tags = Tags
        // Add node
        node, err := self.ClientObj.NodeAdd(nodeReq)
        return node ,err
    }
### 给节点添加一个存储设备
    func(self *Heketi)AddDevice(nodeId string, deviceName string)(bool,error){
            // add device
            deviceReq := &api.DeviceAddRequest{}
            deviceReq.Name = deviceName
            deviceReq.NodeId = nodeId
            err := self.ClientObj.DeviceAdd(deviceReq)
            if err != nil{
                return false,err
            }
            return true,nil
    }
### 在设备中创建一个卷
    func(self *Heketi)VolumeCreate(size int,name string,replica int,clustersIdList []string,block bool)(string,error){
        volumeCreateReq := &api.VolumeCreateRequest{} 
        volumeCreateReq.Size = size
        volumeCreateReq.Clusters = clustersIdList
        volumeCreateReq.Name = name
        volumeCreateReq.Block  = block
        volumeCreateReq.Durability.Replicate.Replica = replica
        volumeInfoRes := &api.VolumeInfoResponse{}
        volumeInfoRes,err := self.ClientObj.VolumeCreate(volumeCreateReq)
        return volumeInfoRes.VolumeInfo.Id,err
    }

### 卷删除
    func(self *Heketi)VolumeDelete(id string)(error){
        return self.ClientObj.VolumeDelete(id)
    }
### 卷信息获取
    func(self *Heketi)VolumeInfo(id string){
        info,_ := self.ClientObj.VolumeInfo(id)
        fmt.Println(info)
    }

### 获取集群信息细节
    func(self *Heketi)GetClusterInfoDetail(clusterId string)(map[string]interface{}){

        /*
            {
                "ClusterId":"",
                "ClusterInfo":[
                    {
                        "NodeName":"",
                        "NodeId":"",
                        "NodeState":"",
                        "DevideInfo":[
                        {
                            "DeviceId":"",
                            "DeviceName":"",
                            "DeviceState":"",
                            "DeviceTotal":0,
                            "DeviceUsed":0,
                            "DeviceFree":0,
                        }
                    ]
                    }
                ]
            }
        */
        clusterInfo,_ := self.ClientObj.ClusterInfo(clusterId)
        clusterInfoDetailMap := map[string]interface{}{}
        var nodeList  []interface{}
        clusterInfoDetailMap["ClusterId"] = clusterId
        //select  cluster's node
        for _,node:= range clusterInfo.Nodes {
            nodeInfoRes ,_:= self.ClientObj.NodeInfo(node) 
            var newdDviceInfoList  []interface{}
            newNodeInfo := map[string]interface{}{}
            newNodeInfo["NodeState"] = (*nodeInfoRes).State
            newNodeInfo["NodeId"] = (*nodeInfoRes).NodeInfo.Id
            newNodeInfo["NodeName"] = (*nodeInfoRes).NodeInfo.NodeAddRequest.Hostnames.Manage
            devicesInfoResList := (*nodeInfoRes).DevicesInfo   
            for _,deviceInfoRes := range devicesInfoResList{
                deviceInfo := map[string]interface{}{}
                deviceInfo["DeviceId"] = deviceInfoRes.DeviceInfo.Id
                deviceInfo["DeviceName"] = deviceInfoRes.DeviceInfo.Device.Name
                deviceInfo["DeviceUsed"] = deviceInfoRes.DeviceInfo.Storage.Used
                deviceInfo["DeviceFree"] = deviceInfoRes.DeviceInfo.Storage.Free
                deviceInfo["DeviceTotal"] = deviceInfoRes.DeviceInfo.Storage.Total 
                deviceInfo["DeviceState"] = deviceInfoRes.State
                newdDviceInfoList = append(newdDviceInfoList,deviceInfo)
                newNodeInfo["DevideInfo"] = newdDviceInfoList											
            }
            nodeList = append(nodeList,newNodeInfo)
        }
        clusterInfoDetailMap["ClusterInfo"] = nodeList
        return clusterInfoDetailMap
    }