# hostnic-cni

[![codebeat badge](https://codebeat.co/badges/33b711c7-0d90-4023-8bb1-db32ec32e4b7)](https://codebeat.co/projects/github-com-yunify-hostnic-cni-master) [![Build Status](https://travis-ci.org/yunify/hostnic-cni.svg?branch=master)](https://travis-ci.org/yunify/hostnic-cni) [![Go Report](https://goreportcard.com/badge/github.com/yunify/hostnic-cni)](https://goreportcard.com/report/github.com/yunify/hostnic-cni) [![License](https://img.shields.io/github/license/openshift/source-to-image.svg)](https://www.apache.org/licenses/LICENSE-2.0.html) [![codecov](https://codecov.io/gh/yunify/hostnic-cni/branch/master/graph/badge.svg)](https://codecov.io/gh/yunify/hostnic-cni)


中文 | [English](README_en.md)

**hostnic-cni** 是一个 [Container Network Interface](https://github.com/containernetworking/cni) 插件。 本插件会直接调用 IaaS 的接口去创建网卡，并将容器的内部的接口连接到网卡上，不同Node上的Pod能够借助IaaS的SDN进行通讯。此插件的优点有：

1. **更强大**： Pod通讯借助于IaaS平台SDN能力，相比于传统的CNI，能够处理更多流量，更大的吞吐量以及更低的延迟。
2. **更灵活**：大部分插件的PodIP对外不可访问，对于一些需要PodIP的服务比如Spring Cloud原生的插件无法使用。Hostnic中Pod可直接被外部访问，能够作为企业的容器服务平台的基础设施。同时PodIP也可以静态配置，方便企业管控。
3. **更快速**：Pod在跨二层Node中也能有和二层通讯的访问速度，跨网段跨Region的集群内部Pod之间的访问速度更快
4. **更安全**：Hostnic也支持网络策略，提供本地的网络策略，同时用户也可以利用IaaS平台的VPC功能做更多的控制。可以利用IaaS平台的网卡监控对网络流量进行监控和限制
5. **更稳定**：集成IaaS平台的LoadBalancer，相比于原生的nodeport的方式更稳定，也不需要用到大端口。

## 插件原理

[插件原理](docs/proposal.md)

## 使用说明


1. `hostnic`需要有在云平台上操作网络的权限，所以首先需要增加 IaaS 的 sdk 配置文件，并将其存储中kube-system中的`qcsecret`中。

    ```bash
    cat >config.yaml <<EOF
    qy_access_key_id: "Your access key id"
    qy_secret_access_key: "Your secret access key"
    # your instance zone
    zone: "pek3a"
    EOF

    ## 创建Secret
    kubectl create secret generic qcsecret --from-file=./config.yaml -n kube-system
    ```
    access_key 以及 secret_access_key 可以登录青云控制台，在 **API 秘钥**菜单下申请。  请参考https://docs.qingcloud.com/product/api/common/overview.html。默认是配置文件指向青云公网api server，如果是私有云，请按照下方示例配置更多的参数：
    ```
    qy_access_key_id: 'ACCESS_KEY_ID'
    qy_secret_access_key: 'SECRET_ACCESS_KEY'

    host: 'api.xxxxx.com'
    port: 443
    protocol: 'https'
    uri: '/iaas'
    connection_retries: 3
    ```
2. 安装yaml文件，等待所有节点的hostnic起来即可
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/yunify/hostnic-cni/master/deploy/hostnic.yaml
    ```

3. (**可选**)启用Network Policy，建议安装
hostnic支持network policy，如果需要，执行下面的命令即可
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/yunify/hostnic-cni/master/deploy/policy.yaml
    ```
## 已知的问题
1. 由于目前iaas不支持多IP网卡，所以每个Node上只能挂载62个Pod(除去主网卡)，对于一般规模的集群已经足够了。
2. 由于一个已知的BUG，在青云上多网卡主机重启会修改默认路由。所以需要在/etc/rc.local中添加一个指向主网卡`eth0`默认路由，比如`ip route replace default via 192.168.1.1 dev eth0`
3. 由于Linux的内核的问题，偶尔会出现一个网卡在重启之后消失的情况，这个时候需要去控制台手动重新挂载这个网卡
4. 由于很多系统中默认开启NetworkManager服务， 当vnic中ip被删除掉之后， 该服务仍然会通过dhcp定期获取ip, 这样也会导致pod不可访问。
所以需要停止NetworkManager服务。 

