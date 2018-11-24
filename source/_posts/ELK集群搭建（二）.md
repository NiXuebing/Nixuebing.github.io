title: ELK集群搭建（二）
author: ACE NI
tags:
  - ELK
categories:
  - 高并发架构
date: 2018-04-14 16:10:00
---
# Elasticsearch 破解

本来是想先整logstash订阅kafka集群清洗日志到elasticsearch，但是突然发现kibana显示x-pack一个月过期。
百度一下果然有破解方法，这一下强迫症就犯了，不破解不舒服斯基。
蛋疼的是网上资料基本都是5版本的，最高也就6.0版本，和我现在用的6.2.3的版本还是有点差距，果然一路趟坑，花了一天时间终于爬完了破解之旅。

<!--more-->

## 反编译

反编译肯定先找到源码，网上的资料基本都是 `/usr/local/elasticsearch/plugins/x-pack/x-pack-6.0.0.jar`， 一看，6.2.3版果然没有`x-pack-6.2.3.jar`，不过在x-pack-core下有个`x-pack-core-6.2.3.jar`，解压一看正好有`org/elasticsearch/license/LicenseVerifier.class`，这就可以玩下去了。。。

```bash
cd /data/local
# 新建测试目录
mkdir test

# 剪切到测试目录
cp /usr/local/server/elasticsearch-6.2.3/plugins/x-pack/x-pack-core/x-pack-core-6.2.3.jar test/

# 切换到测试目录
cd test

# 解压jar包
jar -xvf x-pack-core-6.2.3.jar

# 移除jar包
rm x-pack-core-6.2.3.jar

```

找到`org/elasticsearch/license/LicenseVerifier.class`，使用Luyten反编译（用的Mac版，不知道Windows是不是一样）

```java
package org.elasticsearch.license;

import java.nio.*;
import java.util.*;
import java.security.*;
import org.elasticsearch.common.xcontent.*;
import org.apache.lucene.util.*;
import org.elasticsearch.common.io.*;
import java.io.*;

public class LicenseVerifier
{
    public static boolean verifyLicense(final License license, final byte[] encryptedPublicKeyData) {
        byte[] signedContent = null;
        byte[] signatureHash = null;
        try {
            final byte[] signatureBytes = Base64.getDecoder().decode(license.signature());
            final ByteBuffer byteBuffer = ByteBuffer.wrap(signatureBytes);
            final int version = byteBuffer.getInt();
            final int magicLen = byteBuffer.getInt();
            final byte[] magic = new byte[magicLen];
            byteBuffer.get(magic);
            final int hashLen = byteBuffer.getInt();
            signatureHash = new byte[hashLen];
            byteBuffer.get(signatureHash);
            final int signedContentLen = byteBuffer.getInt();
            signedContent = new byte[signedContentLen];
            byteBuffer.get(signedContent);
            final XContentBuilder contentBuilder = XContentFactory.contentBuilder(XContentType.JSON);
            license.toXContent(contentBuilder, (ToXContent.Params)new ToXContent.MapParams((Map)Collections.singletonMap("license_spec_view", "true")));
            final Signature rsa = Signature.getInstance("SHA512withRSA");
            rsa.initVerify(CryptUtils.readEncryptedPublicKey(encryptedPublicKeyData));
            final BytesRefIterator iterator = contentBuilder.bytes().iterator();
            BytesRef ref;
            while ((ref = iterator.next()) != null) {
                rsa.update(ref.bytes, ref.offset, ref.length);
            }
            return rsa.verify(signedContent) && Arrays.equals(Base64.getEncoder().encode(encryptedPublicKeyData), signatureHash);
        }
        catch (IOException ex) {}
        catch (NoSuchAlgorithmException ex2) {}
        catch (SignatureException ex3) {}
        catch (InvalidKeyException e) {
            throw new IllegalStateException(e);
        }
        finally {
            Arrays.fill(encryptedPublicKeyData, (byte)0);
            if (signedContent != null) {
                Arrays.fill(signedContent, (byte)0);
            }
            if (signatureHash != null) {
                Arrays.fill(signatureHash, (byte)0);
            }
        }
    }

    public static boolean verifyLicense(final License license) {
        byte[] publicKeyBytes;
        try (final InputStream is = LicenseVerifier.class.getResourceAsStream("/public.key")) {
            final ByteArrayOutputStream out = new ByteArrayOutputStream();
            Streams.copy(is, (OutputStream)out);
            publicKeyBytes = out.toByteArray();
        }
        catch (IOException ex) {
            throw new IllegalStateException(ex);
        }
        return verifyLicense(license, publicKeyBytes);
    }
}

```

拷贝内容到新建的文件`LicenseVerifier.java`，两个方法返回值改为true

```java
package org.elasticsearch.license;

import java.nio.*;
import java.util.*;
import java.security.*;
import org.elasticsearch.common.xcontent.*;
import org.apache.lucene.util.*;
import org.elasticsearch.common.io.*;
import java.io.*;

public class LicenseVerifier
{
    public static boolean verifyLicense(final License license, final byte[] encryptedPublicKeyData) {
        return true;
    }

    public static boolean verifyLicense(final License license) {
        return true;
    }
}
```

对`LicenseVerifier.java`进行重新编译，需要依赖三个包
```bash
javac -cp "/usr/local/server/elasticsearch-6.2.3/lib/elasticsearch-6.2.3.jar:
/usr/local/server/elasticsearch-6.2.3/lib/lucene-core-7.2.1.jar:
/usr/local/server/elasticsearch-6.2.3/plugins/x-pack/x-pack-core/x-pack-core-6.2.3.jar" LicenseVerifier.java
```

生成`LicenseVerifier.class` 替换原来的class文件，重新打包
```bash
jar -cvf x-pack-core-6.2.3.jar ./*
mv x-pack-core-6.2.3.jar /usr/local/server/elasticsearch-6.2.3/plugins/x-pack/x-pack-core/
```

至此破解完成，多台ES则把x-pack-core-6.2.3.jar都替换掉，先别急着启动，一启一个坑


## 注册

[在线注册](https://register.elastic.co/)，成功会收到邮件, 有下载`license.json`的链接。
```
{"license":{
    "uid":"xxx",
    "type":"platinum",
    "issue_date_in_millis":1515024000000,
    "expiry_date_in_millis":1596646399999,
    "max_nodes":100,
    "issued_to":"aaa",
    "issuer":"Web Form",
    "signature":"111",
    "start_date_in_millis":1515024000000
    }
}
```

`type` 修改为 `platinum`(白金版)
`expiry_date_in_millis` 修改为 `2524579200999` (2050年)


## 修改elasticsearch.yml

x-pack白金版必须启动SSL，就必须要配置CA证书，然后就是一堆坑，`p12`格式的证书一直跑不通，只能用`key+crt`
```
bin/x-pack/certgen

# 根据提示 填写输出的证书名，节点名，ip，DNS等，多个节点则最后一步选Y，进行下一个节点证书的生成，出错了没关系，删了生成的zip包重新再来就好了

```

生成好证书后就解压放到合理位置就好了，elasticsearch.yml里指定路径，我的是放在`config/certs`下， 配置如下：
```yml
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: dev-elasticsearch
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: node-2
#
# Add custom attributes to the node:
#
node.attr.rack: r2
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
path.data: /data/share/server_data/mq_02/elasticsearch
#
# Path to log files:
#
path.logs: /data/share/server_log/mq_02/elasticsearch
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
#bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: 172.18.1.153
#
# Set a custom port for HTTP:
#
http.port: 9200
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when new node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
discovery.zen.ping.unicast.hosts: ["172.18.1.152", "172.18.1.153", "172.18.1.154"]
#
# Prevent the "split brain" by configuring the majority of nodes (total number of master-eligible nodes / 2 + 1):
#
#discovery.zen.minimum_master_nodes: 2
#
# For more information, consult the zen discovery module documentation.
#
# ---------------------------------- Gateway -----------------------------------
#
# Block initial recovery after a full cluster restart until N nodes are started:
#
#gateway.recover_after_nodes: 3
#
# For more information, consult the gateway module documentation.
#
# ---------------------------------- Various -----------------------------------
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true

xpack.ssl.key: certs/${node.name}/${node.name}.key
xpack.ssl.certificate: certs/${node.name}/${node.name}.crt
xpack.ssl.certificate_authorities: certs/ca/ca.crt
```

## 破解

首先清理`path.data`下的`nodes`数据，从头开始，启动第一个节点
```bash
bin/elasticsearch

#应该是提示需要配置密码，另起窗口进行密码配置
bin/x-pack/set-passwords interactive

#配置完密码可以先查看当前权限
curl -XGET -u elastic:XXX 'http://172.18.1.152:9200/_xpack/license'

#发送licence.json修改权限，在license.json同级目录运行
curl -XPUT -u elastic:XXX 'http://172.18.1.152:9200/_xpack/license?acknowledge=true' -H "Content-Type: application/json" -d @license.json


#再次查看权限应该已经修改为platinum
curl -XGET -u elastic:XXX 'http://172.18.1.152:9200/_xpack/license'


```

集群启动配好证书和`elasticsearch.yml`后启动节点就行，会自动进行选举和复制，无需再次破解。
