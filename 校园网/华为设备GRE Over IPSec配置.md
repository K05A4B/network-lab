# 华为设备GRE Over IPSec配置

1. 配置GRE隧道

```shell
interface Tunnel0/0/<1-1023> # 配置GRE隧道
  description 描述
  tunnel-protocol gre # 指定隧道协议为gre
  source <GRE的源ip地址>
  destination <GRE的目的ip地址>
```

2. 配置acl用于筛选需要送入ipsec的流量

```bash
acl number <acl编号>
  description <描述>
  rule 0 permit ip source <源ip地址> 0 destination <目的地址> 0
```

3. 配置IPSec提议

```bash
ipsec proposal <name>
  encapsulation-mode transport // 传输模式
  esp authentication-algorithm sha2-512 // 采用sha2-512进行认证
  esp encryption-algorithm aes-256
```

4. 配置IKE提议

```bash
ike proposal 1
  encryption-algorithm aes-cbc-128
  dh groupl4 // 使用2048 bits Diffie-Hellman group
  authentication-algorithm sha1
```

5. 配置ike对等体

```bash
ike peer <IPSec提议名> v1
  ike-proposal <IKE提议编号>
  pre-shared-key cipher <预共享密钥>
  remote-address <对端地址>
```

6. 配置ipsec策略

```bash
ipsec policy <IPSec提议名> <策略编号> isakmp
 security acl <ACL编号>  //配置需要加密传送的ACL                                
 ike-peer <对等体名字>  //选择使用的IKE对等体                                     
 proposal <ipsec提议名>  //选择IPSec提议
```

7. 在对应的接口配置安全策略

```bash
interface <端口编号>
  ipsec policy <策略名>
```

