



# 1. StrongSwan 简介





## 1.1. 网络安全防护



StrongSwan 是一套完整的 IPSec 解决方案，可为服务器和客户端提供加密与认证功能。它可用于保护与远程网络的通信，让远程连接体验与本地连接一致。

![topology](./assets/topology.png)

- Gateway：

  网关通常是防火墙，也可以是网络中的任意主机。网关通常还能为小型网络提供 DHCP 和 DNS 服务。在图示中，主机 moon 和 sun 分别是内部主机 alice、venus 和 bob 的网关。

- Remote Access / Roadwarrior Clients：

  漫游客户端通常指笔记本电脑及其他移动设备，它们通过网关远程连接到家庭网络。在图示中，carol 和 dave 代表漫游客户端，它们希望访问两个网关背后的任意一个网络。

- Remote Hosts / Host-to-Host：

  远程主机可以是远程 Web 服务器或备份系统。图示中，主机 winnetou 与网关 moon、sun 中的任意一个的连接，就是该场景的示例。两台主机之间的连接通常可由任意一方发起。

- Remote Sites / Site-to-Site：

  位于不同地点的两个或多个子网中的主机，应能实现互访。再次参考图示，网关 moon 和 sun 背后的两个子网（10.1.0.0/16 和 10.2.0.0/16）可通过连接实现互访，例如主机 alice 和 bob 能安全地通信。



## 1.2. IKE 与 IPSec 基础

StrongSwan 本质上是一个密钥守护进程，它使用互联网密钥交换协议版本 2（IKEv2）在两台对等设备间建立安全关联（SAs）并协商安全策略（SPs）。对于旧版应用，仍支持 IKEv1，但由于稳定性和部分安全问题（该协议已正式弃用），我们强烈不建议使用 IKEv1。

IKE 能对两台对等设备进行强效认证，并生成唯一的、具备强密码学特性的会话密钥。此类 IKE 会话在我们的文档中通常记为 IKE_SA。除认证和密钥材料外，IKE 还能用于交换配置信息（如虚拟 IP 地址），并协商 IPsec SAs（通常称为子安全关联，即 Child SAs 或 CHILD_SAs）。IPsec SAs 定义了需保护的网络流量，以及对这些流量的加密和认证方式。







## 1.3. 认证基础

为确保建立 IKE_SA 的对等设备身份真实，需对其进行认证。strongSwan 提供多种认证方式：



### 1.3.1. 公钥认证

- 证书可自签名（需在所有对等设备上安装），也可由通用证书颁发机构（CA）签名。
- 后者能大幅简化部署与配置：网关只需安装 CA 证书，即可对所有提供该 CA 签名有效证书的对等设备进行认证。
- 可使用证书吊销列表（CRLs）或在线证书状态协议（OCSP）验证证书有效性。
- 私钥可通过 pkcs11 插件存储在智能卡中，确保安全。
- 为防止中间人攻击，对等设备声明的身份需通过证书确认（可通过 subjectDn 或 subjectAltName 扩展字段验证）。



### 1.3.2. 预共享密钥认证

预共	使用高强度密钥才能保证安全。

- 若多个用户知晓同一 PSK（IKEv1 XAuth 与 PSK 组合使用时常见此情况），则任何知晓该密钥的用户都可能伪装成网关。
- 因此，该方式不建议用于大规模部署。



### 1.3.3. 可扩展认证协议







## 1.4. 配置文件







# 2. StrongSwan安装



## 2.1. 安装文档



### 2.1.1. 发行版软件包





### 2.1.2. 自行编译 strongswan



> 注意：在从源代码编译并安装 strongSwan 之前，请确保未安装任何与 strongSwan 相关的发行版软件包。



#### 2.1.2.1. 内核要求

strongSwan 可在大多数发行版的内核上运行。若你自行编译内核，需包含所需的内核模块。



#### 2.1.2.2. 编译 strongswan

strongSwan 的编译采用 GNU 自动工具集（Autotools）。目前可配置的 `./configure options` 数量持续增加。

若需了解部分选项对应的已启用插件，可参考插件列表（list of plugins）。



> 注意：
>
> 许多配置选项默认已启用。因此，请运行 `./configure --help` 查看你所使用的 strongSwan 版本实际支持哪些选项。
>
> 部分插件依赖第三方库。若要编译这些插件，需先安装对应库的头文件。请确保系统中已安装这些文件，例如在基于 Debian 的系统上，可通过安装相应的 `-dev` 包实现。否则，配置脚本会提示无法找到该库或其头文件。
>
> 当修改配置选项（尤其是启用 / 禁用插件）后，重新编译前请务必运行 `make clean`，以确保所有修改都能生效。



编译步骤与其他基于自动工具集的项目一致：

- 下载 strongswan

```
wget https://download.strongswan.org/strongswan-x.x.x.tar.bz2 #注：x.x.x 需替换为具体版本号
```



- 解压缩并进入目录

```
tar xjf strongswan-x.x.x.tar.bz2
cd strongswan-x.x.x
```



- 使用可用选项配置

```
./configure --prefix=/usr --sysconfdir=/etc --<your-options>
```

默认安装路径（prefix）为 `/usr/local`，因此配置文件会从 `/usr/local/etc` 读取。建议至少添加 `--sysconfdir=/etc` 选项，将配置文件放在更通用的目录下。



- 编译源代码并以 root 权限安装二进制文件

```
make
sudo make install
```



#### 2.1.2.3. 从 Git 仓库编译

若要从 Git 仓库编译 strongSwan，需额外安装工具并执行特定步骤。

详情请参考 [`HACKING`](https://github.com/strongswan/strongswan/blob/master/HACKING) 文档。



#### 2.1.2.4. 从其他平台编译

- [strongSwan on FreeBSD](https://docs.strongswan.org/docs/latest/os/freebsd.html)
- [strongSwan on macOS](https://docs.strongswan.org/docs/latest/os/macos.html)
- [strongSwan on Android](https://docs.strongswan.org/docs/latest/os/android.html)
- [strongSwan on Windows](https://docs.strongswan.org/docs/latest/os/windows.html)



#### 2.1.2.5. 单体构建

可将插件集成到其关联的库中，形成所谓的 “单体构建”（通过 `--enable-monolithic` 选项启用）。

采用这种方式，无需为每个插件单独分发共享对象文件，只需分发主要库和可执行文件即可。

运行时加载哪些插件，仍可通过 [此处](https://docs.strongswan.org/docs/latest/plugins/pluginLoad.html) 所述的选项进行控制。



- 静态构建

从 5.5.3 版本开始，可通过以下配置实现 “静态构建”—— 该构建仅依赖第三方库，即我们自己的库和插件均静态链接到可执行文件中：

```
--disable-shared --enable-static --enable-monolithic
```

由于我们构建系统的限制，若需包含第三方静态库，需手动修改 Makefile（例如可参考模糊测试目标的 Makefile：`fuzz/Makefile.am`）。



### 2.1.3. 配置管理

若你使用配置管理（CM）工具管理软件，有以下选项可帮助配置 strongSwan：

- Chef：可用的 “食谱”（cookbooks）包括：	
  - https://supermarket.chef.io/cookbooks/strongswanaws
  - https://supermarket.chef.io/cookbooks/strongswan-base



- Puppet：可用的模块（modules）包括：
  - https://forge.puppet.com//Nextdoor/strongswan
  - https://forge.puppet.com//jpds/strongswan





# 3. 配置



## 3.1. 快速入门配置

用户、主机和网关的证书由虚构的 strongSwan 证书颁发机构（CA）颁发。

在我们的示例场景中，所有 VPN 端点都必须存在 CA 证书 strongswanCert.pem，才能对对等方进行身份验证。

对于特定的 VPN 应用，你可以使用任何第三方 CA 颁发的证书，也可以通过 strongSwan 的 pki 工具自行生成所需的私钥和证书，其使用方法在证书快速入门部分有说明。



### 3.1.1. Site-to-Site 案例

在此场景中，两个安全网关（moon 和 sun）将通过二者之间建立的 VPN 隧道，把两个子网（moon-net 和 sun-net）连接起来，网络拓扑如下：

```bash
10.1.0.0/16 -- | 192.168.0.1 | === | 192.168.0.2 | -- 10.2.0.0/16
  moon-net          moon                 sun           sun-net
```



- moon 网关的配置

```bash
/etc/swanctl/x509ca/strongswanCert.pem
/etc/swanctl/x509/moonCert.pem
/etc/swanctl/private/moonKey.pem

/etc/swanctl/swanctl.conf:

  connections {
    net-net {
      remote_addrs = 192.168.0.2
      local {
        auth = pubkey
        certs = moonCert.pem
      }
      remote {
        auth = pubkey
        id = "C=CH, O=strongSwan, CN=sun.strongswan.org"
      }
      children {
        net-net {
          local_ts  = 10.1.0.0/16
          remote_ts = 10.2.0.0/16
          start_action = trap
        }
      }
    }
  }
```



- sun 网关的配置

```bash
/etc/swanctl/x509ca/strongswanCert.pem
/etc/swanctl/x509/sunCert.pem
/etc/swanctl/private/sunKey.pem

/etc/swanctl/swanctl.conf:

  connections {
    net-net {
      remote_addrs = 192.168.0.1
      local {
        auth = pubkey
        certs = sunCert.pem
      }
      remote {
        auth = pubkey
        id = "C=CH, O=strongSwan, CN=moon.strongswan.org"
      }
      children {
        net-net {
          local_ts  = 10.2.0.0/16
          remote_ts = 10.1.0.0/16
          start_action = trap
        }
      }
    }
  }
```



本场景中使用的本地和对端身份标识，是终端实体证书中包含的主题可分辨名称（DN）。通过以下命令可将证书和私钥加载到 charon 守护进程中：

```bash
swanctl --load-creds
```



通过以下命令加载 swanctl.conf 中定义的连接配置：

```bash
swanctl --load-conns
```



当配置为`start_action = trap`时，只要有首个明文 IP 数据包需要通过隧道传输，IPsec 连接就会自动建立。



### 3.1.2. Host-to-Host 场景

此场景为两台独立主机之间的配置，主机后方没有子网。尽管 IPsec 传输模式已满足主机到主机连接的需求，但本示例仍使用默认的 IPsec 隧道模式，网络拓扑如下：

```bash
| 192.168.0.1 | === | 192.168.0.2 |
     moon                sun
```



- moon 主机配置

```bash
 /etc/swanctl/x509ca/strongswanCert.pem
/etc/swanctl/x509/moonCert.pem
/etc/swanctl/private/moonKey.pem

/etc/swanctl/swanctl.conf:

  connections {
    host-host {
      remote_addrs = 192.168.0.2
      local {
        auth=pubkey
        certs = moonCert.pem
      }
      remote {
        auth = pubkey
        id = "C=CH, O=strongSwan, CN=sun.strongswan.org"
      }
      children {
        host-host {
          start_action = trap
        }
      }
    }
  }
```



- sun 主机配置

```bash
/etc/swanctl/x509ca/strongswanCert.pem
/etc/swanctl/x509/sunCert.pem
/etc/swanctl/private/sunKey.pem

/etc/swanctl/swanctl.conf:

  connections {
    host-host {
      remote_addrs = 192.168.0.1
      local {
        auth = pubkey
        certs = sunCert.pem
      }
      remote {
        auth = pubkey
        id = "C=CH, O=strongSwan, CN=moon.strongswan.org"
      }
      children {
        host-host {
          start_action = trap
        }
      }
    }
  }
```



### 3.1.3. 移动用户场景

这是一种非常常见的场景：strongSwan 网关为任意数量的远程 VPN 客户端提供服务，这些客户端通常使用动态 IP 地址，网络拓扑如下：

```bash
10.1.0.0/16 -- | 192.168.0.1 | === | x.x.x.x |
  moon-net          moon              carol
```



- moon 网关的配置

```bash
/etc/swanctl/x509ca/strongswanCert.pem
/etc/swanctl/x509/moonCert.pem
/etc/swanctl/private/moonKey.pem

/etc/swanctl/swanctl.conf:

  connections {
    rw {
      local {
        auth = pubkey
        certs = moonCert.pem
        id = moon.strongswan.org
      }
      remote {
        auth = pubkey
      }
      children {
        rw {
          local_ts  = 10.1.0.0/16
        }
      }
    }
  }
```



- 移动用户 carol 的配置

```bash
/etc/swanctl/x509ca/strongswanCert.pem
/etc/swanctl/x509/carolCert.pem
/etc/swanctl/private/carolKey.pem

/etc/swanctl/swanctl.conf:

  connections {
    home {
      remote_addrs = moon.strongswan.org
      local {
        auth = pubkey
          certs = carolCert.pem
          id = carol@strongswan.org
        }
      remote {
        auth = pubkey
        id = moon.strongswan.org
      }
      children {
        home {
          remote_ts  = 10.1.0.0/16
          start_action = start
        }
      }
    }
  }
```



在`remote_addrs`中，我们选择了主机名`moon.strongswan.org`，运行时会通过 DNS 将其解析为对应的目标 IP 地址。

本场景中，移动用户 carol 的身份标识为邮箱`carol@strongswan.org`，该标识必须作为 “主题备用名称（subjectAltName）” 包含在 carol 的客户端证书（carolCert.pem）中。



## 3.2. 快速入门证书

本节并非关于如何使用 strongSwan pki 工具的完整教程，仅列出若你想生成自用证书及证书吊销列表（CRLs）以搭配 strongSwan 使用时，需关注的几个要点。



### 3.2.1. 生成 CA 证书

通过 `pki --gen` 命令可生成密钥，示例如下：

```bash
pki --gen --type ed25519 --outform pem > strongswanKey.pem
```

该命令会生成一个加密强度为 128 位的爱德华兹椭圆曲线密钥（EdDSA）。其他可选密钥类型包括 ecdsa、传统 rsa 及 ed448。并非所有实现都支持 EdDSA，因此为保证最佳兼容性，建议使用 RSA 或 ECDSA 密钥（Windows 对后者还有额外限制，详见《Windows 证书要求》）。



> 警告：
>
> 切勿将证书机构（CA）的私钥存储在可直接、持续访问互联网的主机（如 VPN 网关）上。一旦此主签名密钥被盗，你的公钥基础设施（PKI）将完全陷入不安全状态。



使用 `pki --self` 命令，可将对应的公钥打包成有效期为 10 年（3652 天）的自签名 CA 证书，命令如下：

```plaintext
pki --self --ca --lifetime 3652 --in strongswanKey.pem \
           --dn "C=CH, O=strongSwan, CN=strongSwan Root CA" \
           --outform pem > strongswanCert.pem
```



通过 `pki --print` 命令可查看证书详情，示例如下：

```plaintext
pki --print --in strongswanCert.pem

subject:  "C=CH, O=strongSwan, CN=strongSwan Root CA"
issuer:   "C=CH, O=strongSwan, CN=strongSwan Root CA"
validity:  not before May 18 08:32:06 2017, ok
           not after  May 18 08:32:06 2027, ok (expires in 3651 days)
serial:    57:e0:6b:3a:9a:eb:c6:e0
flags:     CA CRLSign self-signed
subjkeyId: 2b:95:14:5b:c3:22:87:de:d1:42:91:88:63:b3:d5:c1:92:7a:0f:5d
pubkey:    ED25519 256 bits
keyid:     a7:e1:6a:3f:e7:6f:08:9d:89:ec:23:92:a9:a1:14:3c:78:a8:7a:f7
subjkey:   2b:95:14:5b:c3:22:87:de:d1:42:91:88:63:b3:d5:c1:92:7a:0f:5d
```



若希望 CA 私钥和 X.509 证书以二进制 DER 格式存储，省略 `--outform pem` 参数即可。



`/etc/swanctl/x509ca` 目录用于存储所有必需的 CA 证书，支持二进制 DER 或 Base64 PEM 格式。无论文件后缀如何，strongSwan 会自动识别正确格式。



### 3.2.2. 生成终端实体证书



#### 3.2.2.1. 生成终端实体私钥

同样使用 `pki --gen` 命令，为主机（示例主机名：moon）生成 Ed25519 私钥，命令如下：

```bash
pki --gen --type ed25519 --outform pem > moonKey.pem
```



也可通过以下命令生成传统的 3072 位 RSA 密钥，并以二进制 DER 格式存储：

```
pki --gen --type rsa --size 3072 > moonKey.der
```



根据密钥类型不同，主机或用户的私钥会存储在 `/etc/swanctl` 目录下对应的子目录中，可选子目录包括 ecdsa、rsa、pkcs8 或 private。



作为一个可选项，现代英特尔平台普遍配备 TPM 2.0 可信平台模块，可将其用作虚拟智能卡，安全存储 RSA 或 ECDSA 私钥。详情可参考《TPM 2.0 教程》。



#### 3.2.2.2. 生成证书请求

使用 `pki --req` 命令生成 PKCS#10 格式的证书请求（需由 CA 签名），命令如下：

```
pki --req --type priv --in moonKey.pem \
          --dn "C=CH, O=strongswan, CN=moon.strongswan.org" \
          --san moon.strongswan.org --outform pem > moonReq.pem
```



通过多次使用 `--san` 参数，可在请求中添加任意数量的 “主体备用名称（subjectAlternativeName）”，支持以下格式：

```
--san sun.strongswan.org     # 完全限定主机名
--san carol@strongswan.org   # RFC822 格式用户邮箱
--san 192.168.0.1            # IPv4 地址
--san fec0::1                # IPv6 地址
```



强烈建议在 VPN 网关的证书中，将网关主机名作为 “主体备用名称（subjectAltName）” 包含在内。不同客户端对网关证书可能有额外要求，例如 Windows、iOS 及 macOS 客户端的特定证书要求。



#### 3.2.2.3. 由 CA 签发终端实体证书

基于证书请求，使用 `pki --issue` 命令由 CA 签发终端实体证书，命令如下：

```
pki --issue --cacert strongswanCert.pem --cakey strongswanKey.pem \
            --type pkcs10 --in moonReq.pem --serial 01 --lifetime 1826 \
            --outform pem > moonCert.pem
```



若省略 `--serial` 参数（需后跟十六进制值），系统将自动生成随机序列号。

部分第三方 VPN 客户端要求 VPN 网关证书包含 “TLS 服务器认证扩展密钥用法（EKU）” 标志，可通过添加 `--flag serverAuth` 参数实现。



若需使用后续章节所述的 “动态 CRL 获取” 功能，可通过 `--crl` 参数在终端实体证书中添加一个或多个 CRL 分发点，示例：

```
--crl  http://crl.strongswan.org/strongswan.crl
--crl "ldap://ldap.strongswan.org/cn=strongSwan Root CA, o=strongSwan,c=CH?certificateRevocationList"
```



通过以下命令查看已签发的主机证书详情：

```
pki --print --in moonCert.pem

subject:  "C=CH, O=strongSwan, CN=moon.strongswan.org"
issuer:   "C=CH, O=strongSwan, CN=strongSwan Root CA"
validity:  not before May 19 10:28:19 2017, ok
           not after  May 19 10:28:19 2022, ok (expires in 1825 days)
serial:    01
altNames:  moon.strongswan.org
flags:     serverAuth
CRL URIs:  http://crl.strongswan.org/strongswan.crl
authkeyId: 2b:95:14:5b:c3:22:87:de:d1:42:91:88:63:b3:d5:c1:92:7a:0f:5d
subjkeyId: 60:9d:de:30:a6:ca:b9:8e:87:bb:33:23:61:19:18:b8:c4:7e:23:8f
pubkey:    ED25519 256 bits
keyid:     39:1b:b3:c2:34:72:1a:01:08:40:ce:97:75:b8:be:ce:24:30:26:29
subjkey:   60:9d:de:30:a6:ca:b9:8e:87:bb:33:23:61:19:18:b8:c4:7e:23:8f
```

终端实体证书通常存储在 `/etc/swanctl/x509` 目录中。



应为网络中的每个对等节点（包括所有 VPN 客户端和 VPN 网关）生成独立的私钥，以及由 CA 签名的匹配 X.509 终端实体证书，并将对等节点的私钥和 X.509 证书存储在对应主机的 `/etc/swanctl` 目录中。



### 3.2.3. PKCS#12 容器

Windows、Android、macOS 或 iOS 系统的 VPN 客户端通常需要三类文件：自身私钥、主机 / 用户证书、CA 证书。

最便捷的加载方式是将这些文件打包到 PKCS#12 容器中，命令如下（需使用 OpenSSL 工具）：

```
openssl pkcs12 -export -inkey carolKey.pem \
               -in carolCert.pem -name "carol" \
               -certfile strongswanCert.pem -caname "strongSwan Root CA" \
               -out carolCert.p12
```



当前 strongSwan pki 工具无法创建 PKCS#12 容器，需使用 OpenSSL 工具。



> 重要：
>
> Apple 和 Android 客户端仅支持采用 3DES 加密和传统密钥派生函数（KDF）的 PKCS#12 容器。若使用 OpenSSL 3.0 及以上版本，需添加 `-legacy` 参数。















## 3.3. 配置文件



### 3.3.1. 常规选项

- strongswan.conf 文件
- strongswan.d 目录



### 3.3.2. 连接与凭证



#### 3.3.2.1. 适用于 vici 的现代控制接口

以下配置文件和目录由 swanctl 命令行工具通过 vici 控制接口使用。

- swanctl.conf 文件（swanctl.conf file）
- swanctl 目录（swanctl directory）



#### 3.3.2.2. 适用于已弃用的基于 stroke 的控制接口

以下配置文件和目录由 ipsec 命令行工具及启动进程（starter process）通过 stroke 控制接口使用。

- ipsec.conf 文件（ipsec.conf file）
- ipsec.secrets 文件（ipsec.secrets file）
- ipsec.d 目录（ipsec.d directory）









# 6. How-tos



## 6.1. 转发与拆分隧道



### 6.1.1. 概述

在远程访问场景中，客户端通常会将所有流量发送到网关。下文将说明如何转发这些流量，并将其正确路由回移动用户（roadwarriors）。

在某些场景下，可能更适合仅将特定流量通过网关发送。例如，避免网关承担网页流量转发压力，或更糟的文件共享流量转发压力。因此，本文还将说明如何为不同客户端启用所谓的 “拆分隧道”（split-tunneling）。



### 6.1.2. 转发客户端流量

若要将流量转发到网关后的主机（或在不使用拆分隧道时转发到互联网主机），需在 Linux 网关上启用以下选项：

```bash
sysctl net.ipv4.ip_forward=1
sysctl net.ipv6.conf.all.forwarding=1
```



可将上述命令添加到`/etc/sysctl.conf`（或`/etc/sysctl.d/`目录下的文件）中，以实现永久启用。

若网关上的防火墙限制较严格，可配置：

```bash
connections.<conn>.children.<child>.updown = /usr/libexec/ipsec/_updown iptables
```



这会让默认的上下线脚本（updown script）自动添加允许流量转发的规则。

在远程访问配置中，VPN 客户端会从已配置的地址池获取一个虚拟 IP 地址。在网关侧，关键在于：所有网关需转发流量至的主机，必须知道要通过 VPN 网关将数据包路由回远程访问客户端。

请注意，在云平台上部署时可能需要额外考虑（例如源 / 目标地址检查 / 过滤）。



#### 6.1.2.1. LAN 内主机

对于网关后的 LAN（局域网）内主机，可能存在以下几种情况：



- 虚拟 IP 来自网关后的子网

在此场景下，要么使用 dhcp 插件，要么由网关从 “网关后整个 LAN 的子网” 中分配虚拟 IP（该子网需与其他 LAN 主机通过 DHCP 获取的 IP 地址段区分开）。若如此，必须使用 farp 插件，以便 LAN 内主机知晓需将响应数据包发送至 VPN 网关。对于 IPv6，可通过邻居发现协议（NDP）代理实现类似功能（例如通过自定义上下线脚本或 vici 事件监听器）。



NDP 代理的简单上下线脚本示例：

```c
#!/bin/bash

LAN_DEV=$(ip -6 route get ${PLUTO_PEER_SOURCEIP6_1} | sed -ne 's/^.*dev \(\S\+\) .*/\1/p')
case $PLUTO_VERB in
  up-client-v6)
    ip -6 neigh add proxy ${PLUTO_PEER_SOURCEIP6_1} dev ${LAN_DEV}
    ;;
  down-client-v6)
    ip -6 neigh del proxy ${PLUTO_PEER_SOURCEIP6_1} dev ${LAN_DEV}
    ;;
esac
```



- 虚拟 IP 来自独立子网 / site to site 场景

若 VPN 网关是目标 LAN 的默认网关，则无需额外操作。若不是，则需在网关后的所有主机上添加一条路由（手动添加，或通过 DHCP 选项 121 添加），告知这些主机：“分配给移动用户的虚拟 IP 子网” 或 “站点到站点场景中的远程子网” 可通过 VPN 网关访问。

或者，在实际的默认网关上配置一条静态路由，将 “虚拟子网和远程子网” 的流量重定向到 VPN 网关。也可将虚拟 IP 转换为网关的（内网）IP 地址（即 NAT 转换），这样远程客户端的请求对 LAN 内主机而言，会表现为来自网关的请求（关于 NAT 配置的说明见下一节）。

若 VPN 网关不是 LAN 的默认网关，当主机将 “发往远程主机 / 子网的流量” 发送到 VPN 网关时，可能会收到 ICMP 重定向消息，提示其将流量发送到 LAN 的默认网关（这通常无法正常工作，还可能导致流量未加密就传出）。为避免此问题，需通过以下设置禁用此类 ICMP 消息发送：

```bash
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0
```



若在网络接口启动前未设置上述第二个选项，还需为单个接口设置该选项：

```bash
net.ipv4.conf.<iface>.send_redirects=0
```

注：`<iface>`需替换为实际的网络接口名



#### 6.1.2.2. 互联网主机











# 7. 守护进程





## 7.1. charon

charon 守护进程是为 strongSwan 项目从零构建的，用于实现 IKEv2 协议。其大部分代码位于 libcharon 库中，这使得 IKE 守护进程核心可被其他程序使用，例如 charon-systemd、charon-svc、charon-cmd、NetworkManager 插件 charon-nm，或是 Android 应用。



### 7.1.1. 架构

```plaintext
      +---------------------------------+       +----------------------------+
      |          Credentials            |       |          Backends          |
      +---------------------------------+       +----------------------------+

       +------------+    +-----------+        +------+            +----------+
       |  receiver  |    |           |        |      |  +------+  | CHILD_SA |
       +----+-------+    | Scheduler |        | IKE- |  | IKE- |--+----------+
            |            |           |        | SA   |--| SA   |  | CHILD_SA |
    +-------+--+         +-----------+        |      |  +------+  +----------+
 <->|  socket  |               |              | Man- |
    +-------+--+         +-----------+        | ager |  +------+  +----------+
            |            |           |        |      |  | IKE- |--| CHILD_SA |
       +----+-------+    | Processor |--------|      |--| SA   |  +----------+
       |   sender   |    |           |        |      |  +------+
       +------------+    +-----------+        +------+

      +---------------------------------+       +----------------------------+
      |               Bus               |       |      Kernel Interface      |
      +---------------------------------+       +----------------------------+
             |                    |                           |
      +-------------+     +-------------+                     V
      | File-Logger |     |  Sys-Logger |                  //////
      +-------------+     +-------------+
```



- Processor：

  线程机制借助一个线程池（称为 “处理器”）实现，该线程池包含固定数量的预创建线程。守护进程中的所有线程均源自该处理器。若要将工作委派给线程，需将任务加入处理器队列，以实现异步执行。

- Scheduler：

  调度器负责执行定时事件。可将任务加入调度器队列，使其在指定时间执行（例如重新密钥协商）。调度器自身不执行任务，而是将任务加入处理器队列。

- IKE_SA Manager：

  IKE 安全关联管理器负责管理所有 IKE 安全关联（IKE_SA）。它还处理同步问题：每个 IKE 安全关联必须严格借出，并在使用后归还。该管理器能确保同一时间只有一个线程可借出单个 IKE 安全关联，这使得我们可将（复杂的）IKE 安全关联例程设计为非线程安全的。

- IKE_SA：

  IKE 安全关联包含每个 IKE 安全关联的状态与逻辑，并负责处理消息。

- CHILD_SA：

  CHILD 安全关联包含 IPsec 安全关联的状态并对其进行管理。一个 IKE 安全关联可对应多个 CHILD 安全关联。与内核的通信通过内核接口在此处进行。

- Kernel Interface：

  内核接口负责安装 IPsec 安全关联、策略、路由和虚拟地址。它还提供枚举接口的方法，并可向守护进程通知底层的状态变化。

- Bus：

  总线接收来自不同线程的信号，并将其转发给相关监听器。调试信号、重要的状态变化或错误消息均通过总线传输。

- Controller：

  控制器为插件提供简单的 API，用于控制守护进程（例如发起 IKE 安全关联、关闭 IKE 安全关联等）。

- Backends：

  后端是提供配置的可插拔模块。它们必须实现一个 API，供守护进程核心用于获取配置。

- Credentials：

  通过已注册的插件提供信任链验证与凭证服务。



### 7.1.2. 插件

守护进程在启动时加载插件，这些插件需实现 plugin_t 接口（路径：src/libstrongswan/plugins/plugin.h）。每个插件会向守护进程注册，以接入自身功能。

目前可用的 libcharon 插件列表正在不断增加。

```plaintext
  +-------------------------------------+
  | charon                  +---+ +-----+------+
  |                         |   | |   vici     |
  |                         |   | +-----+------+
  | +-------------+         |   | +-----+------+
  | | bus         |  ---->  | p | |   stroke   |
  | +-------------+         | l | +-----+------+
  | +-------------+  <----  | u | +-----+------+
  | | controller  |         | g | |    sql     |
  | +-------------+  ---->  | i | +-----+------+
  | +-------------+         | n | +-----+------+
  | | credentials |  <----  |   | |  eap_aka   |
  | +-------------+         | l | +-----+------+
  | +-------------+  ---->  | o | +-----+------+
  | | backends    |         | a | |  eap_sim   |
  | +-------------+  <----  | d | +-----+------+
  | +-------------+         | e | +-----+------+
  | | eap         |  ---->  | r | |  eap_md5   |
  | +-------------+         |   | +-----+------+
  |                         |   | +-----+------+
  |                         |   | |eap_identity|
  |                         +---+ +-----+------+
  +-------------------------------------+
```



## 7.2. charon-systemd

charon-systemd 守护进程实现的 IKE 守护进程与 charon 非常相似，但专为与 systemd 配合使用而设计。它使用 systemd 库进行原生集成，并附带一个简单的 systemd 服务文件。该守护进程由 systemd 直接管理，通过 swanctl 配置后端进行配置。



### 7.2.1. 构建选项

要构建该守护进程，需在`./configure`选项中添加：

```shell
--enable-systemd --enable-swanctl
```



若要禁用传统的 ipsec 后端，需额外添加：

```bash
--disable-charon --disable-stroke
```

以使用现代工具构建一个轻量且简洁的 IKE 守护进程。



systemd 单元文件目录会通过 pkg-config 自动检测，也可通过`--with-systemdsystemunitdir=` 选项在`./configure`中手动设置。



### 7.2.2. 运行方式

charon-systemd 作为原生 systemd 守护进程安装，其服务单元名为`strongswan`。该服务单元需通过以下命令启用一次：

```bash
sudo systemctl enable strongswan
```



之后可通过以下命令手动启动守护进程：

```bash
sudo systemctl start strongswan
```



随时可通过以下命令停止：

```bash
sudo systemctl stop strongswan
```



通常在重启后，systemd 会自动启动 strongswan 服务，并通过 swanctl 加载 IPsec 配置（包括连接、池和凭证）。若不确定 charon-systemd 守护进程是否在运行，可通过以下命令检查：

```bash
systemctl status strongswan

strongswan.service - strongSwan IPsec IKEv1/IKEv2 daemon using swanctl
     Loaded: loaded (/lib/systemd/system/strongswan.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-01-26 16:53:41 CET; 5 days ago
    Process: 1354 ExecStartPost=/usr/sbin/swanctl --load-all --noprompt (code=exited, status=0/SUCCESS)
   Main PID: 1308 (charon-systemd)
     Status: "charon-systemd running, strongSwan 6.0dr13, Linux 5.13.0-27-generic, x86_64"
      Tasks: 17 (limit: 18891)
     Memory: 113.1M
     CGroup: /system.slice/strongswan.service
             └─1308 /usr/sbin/charon-systemd
```



### 7.2.3. 配置

`strongswan.conf`中许多特定于 charon 的配置选项也适用于 charon-systemd。实际上，charon 部分中设置的选项会被 charon-systemd 自动继承。

以下是 charon-systemd 特有的选项：

| Key     | Default | Description                                                  |
| :------ | :------ | :----------------------------------------------------------- |
| journal |         | 配置原生 systemd 日志（journal）记录器的部分，与日志记录中描述的 syslog 记录器非常相似 |



#### 7.2.3.1. 日志记录

默认情况下，charon-systemd 后端会记录到 systemd 日志（journal），可通过`journalctl`查看。日志级别的配置与 charon 的日志配置非常相似，但需使用`journal`部分：

```bash
charon-systemd {
  journal {
    default = 1
    ike = 2
    knl = 3
    # ...
  }
}
```



当然，也可在`strongswan.conf`的`charon-systemd`部分中定义传统的 syslog 和 filelog 记录器。详情请参考日志记录配置。若要禁用日志记录器并使其静默，可设置：

```bash
default = -1
```



基于日志（journal）的记录器会在自定义日志字段中提供一些额外元数据：

| Field            | Description                 |
| :--------------- | :-------------------------- |
| LEVEL            | 数值型 strongSwan 日志级别  |
| GROUP            | 日志子系统字符串            |
| THREAD           | 发出日志条目的线程编号      |
| IKE_SA_UNIQUE_ID | IKE_SA 唯一标识符（若存在） |
| IKE_SA_NAME      | IKE_SA 配置名称（若存在）   |



MESSAGE 字段包含日志消息，MESSAGE_ID 使用特定于每种日志消息类型的唯一标识符。日志级别还会映射到 PRIORITY 字段中存储的值（0 对应 LOG_NOTICE，1 对应 LOG_INFO，更高级别对应 LOG_DEBUG，详见 syslog (3)）



## 7.3. charon-svc

charon-svc 是适用于 Windows 平台的 strongSwan IKE 服务。它基于 libcharon 构建，相当于 Unix 系统上的 charon 在 Windows 平台的对应版本。



### 7.3.1. 使用方法

charon-svc 是一款混合应用程序，既可作为命令行应用程序运行，也可作为系统服务运行。

从控制台调用时，应用程序会在前台运行，按 ^C 即可终止。这主要用于测试和调试场景，例如在调试器中运行 charon。另一种更适合生产环境的方式是将应用程序安装为 Windows 服务，可通过任何合适的方法实现，例如使用 sc 工具：

```bash
sc create “strongSwan IKE service” binPath= C:\path\to\charon-svc.exe
```



创建服务后，可通过 sc 工具或 “服务管理控制台” 管理单元对其进行控制。



### 7.3.2. 配置

Windows 使用基于 vici 协议的 swanctl 配置后端。目前，应用程序本身没有任何 strongswan.conf 选项，但 libcharon、libstrongswan 及相关插件的所有选项均适用。与 charon 不同的是，配置项位于 charon-svc 根节点下。charon 节点中设置的选项会被 charon-svc 自动继承。



日志配置遵循以下说明：在 Windows 系统上，除非显式基于 syslog 客户端库构建，否则默认不会将日志记录到 syslog。一个简单的日志配置示例如下：

```plaintext
charon-svc {
  filelog {
    log.txt {
      flush_line = yes
    }
    stdout {
    }
  }
}
```



所有配置的文件路径都是相对于应用程序二进制文件的，因为 charon-svc 启动后会更改其工作目录。



### 7.3.3. 自动配置加载

Windows 版本通常使用 swanctl 作为配置后端。当 charon-svc 作为服务运行时，没有初始化脚本负责在服务启动后加载配置和凭证。因此，libcharon 具备调用启动和停止脚本来执行这些（及其他）任务的能力。要加载 swanctl 配置，可使用以下 strongswan.conf 配置段：

```bash
charon-svc {
  start-scripts {
    swanctl-creds = swanctl --load-creds --noprompt
    swanctl-conns = swanctl --load-conns
  }
}
```



凭证应在连接之前加载，因为连接可能会引用凭证。



## 7.4. charon-cmd



### 7.4.1. 命令格式

```bash
charon-cmd --host <hostname> --identity <id> [options]
```



### 7.4.2. 描述

charon-cmd 是一款命令行程序，用于通过版本 1 和 2 的互联网密钥交换协议（IKE）建立 IPsec VPN 连接。它支持多种不同的移动用户场景。与 IKE 的 charon 守护进程一样，charon-cmd 必须以 root 身份（更具体地说，是具有 CAP_NET_ADMIN 权限的用户）运行。

在以下选项中，至少需要提供 --host 和 --identity。根据所选的认证配置文件，还需通过相应选项提供凭证。

strongswan.conf 中许多特定于 charon 的配置选项也适用于 charon-cmd。例如，要配置自定义的标准输出（stdout）日志，可使用以下配置片段：

```bash
charon-cmd {
  filelog {
    stdout {
      default = 1
      ike = 2
      cfg = 2
    }
  }
}
```



### 7.4.3. 选项

| Option                | Description                                                  |
| :-------------------- | :----------------------------------------------------------- |
| --help                | 显示使用信息和可用选项的简要摘要                             |
| --version             | 显示 strongSwan 版本                                         |
| --debug <level>       | 设置默认日志级别（默认为 1）。<level>是 - 1 到 4 之间的数字。有关更精细配置日志输出的选项，请参考日志记录配置 |
| --host <hostname>     | 要连接的 DNS 名称或 IP 地址                                  |
| --identity<id>        | 客户端在 IKE 交换中使用的身份标识                            |
| --remote-identity<id> | 预期的服务器身份标识，默认为主机名                           |
| --cert <path>         | 受信任的证书，用于认证或信任链验证。若要提供多个证书，可多次使用 --cert 选项 |
| --rsa <path>          | 用于认证的 RSA 私钥（若需要密码，会按需请求）。对于其他密钥类型，请使用 --priv |
| --priv <path>         | 用于认证的私钥（若需要密码，会按需请求）                     |
| --p12 <path>          | 包含私钥和证书的 PKCS#12 文件，用于认证和信任链验证（若需要密码，会按需请求） |
| --agent[=<socket>]    | 使用 SSH 代理进行认证。若未指定套接字，将从 SSH_AUTH_SOCK 环境变量中读取 |
| --local-ts <subnet>   | 为本地端额外提议的流量选择器，请求的虚拟 IP 地址总会被提议   |
| --remote-ts <subnet>  | 为远端提议的流量选择器，默认为 0.0.0.0/0                     |
| --profile <name>      | 要使用的认证配置文件。支持的配置文件列表见下文 “认证配置文件” 部分。默认情况下，若提供了私钥则为 ikev2-pub，否则为 ikev2-eap |



### 7.4.4. 认证配置文件



#### 7.4.4.1. IKEv2 配置文件

| Name                | Description                                                  |
| :------------------ | :----------------------------------------------------------- |
| `**ikev2-pub**`     | IKEv2 with public key client and server authentication       |
| `**ikev2-eap**`     | IKEv2 with EAP client authentication and public key server authentication |
| `**ikev2-pub-eap**` | IKEv2 with public key and EAP client authentication ([RFC 4739](https://datatracker.ietf.org/doc/html/rfc4739)) and public key server authentication |



#### 7.4.4.2. IKEv1 主模式配置文件

| Name                  | Description                                                  |
| :-------------------- | :----------------------------------------------------------- |
| `**ikev1-pub**`       | IKEv1 with public key client and server authentication       |
| `**ikev1-xauth**`     | IKEv1 with public key client and server authentication, followed by client XAuth authentication |
| `**ikev1-xauth-psk**` | IKEv1 with pre-shared key (PSK) client and server authentication, followed by client XAuth authentication |
| `**ikev1-hybrid**`    | IKEv1 with public key server authentication only, followed by client XAuth authentication |



#### 7.4.4.3. IKEv1 主动配置文件

| Name                     | Description                                                  |
| :----------------------- | :----------------------------------------------------------- |
| `**ikev1-pub-am**`       | IKEv1，使用公钥进行客户端和服务器认证                        |
| `**ikev1-xauth-am**`     | IKEv1，使用公钥进行客户端和服务器认证，随后进行客户端 XAuth 认证 |
| `**ikev1-xauth-psk-am**` | IKEv1，使用预共享密钥（PSK）进行客户端和服务器认证，随后进行客户端 XAuth 认证。**不安全！！！** |
| `**ikev1-hybrid-am**`    | IKEv1，仅使用公钥进行服务器认证，随后进行客户端 XAuth 认证   |





















