---
title: "Ipv6腾讯云DDNS配置"
date: 2022-01-25T22:42:47+08:00
draft: false
categories: 技术
keywords: ipv6 腾讯云 ddns 群晖 clash 域名 ubuntu docker 主机网络
---

自从家里的设备全部支持ipv6后(如何支持见上文), 怎么说呢，就一发不可收拾了。总想着尽可能的利用起来ipv6。想起来自己还有个域名，就尝试着试试看能不能让自己的域名仅支持ipv6。



试了之后发现还是挺简单的，虽然不能直接在域名解析上添加A记录，但是腾讯云还是很友好的发现的想添加的是一个ipv6的地址，于是就指引我去添加AAAA的记录过去了。添加完之后，就等着解析了。很快家里的网络就可以正常解析我只添加了AAAA记录的新地址blog.jiangbo.space。我的博客流程的打开了。



这里有一点比较坑的是，如果你使用了代理工具，特别是clash，这个是亲测可能会有影响的。我目前使用机场的配置文件中默认是不支持ipv6的，导致我走了很多弯路。如果使用这种不支持ipv6的配置，又使用了clash的代理，那么很不幸，即使你本地网络支持ipv6你也不能正常访问仅提供ipv6支持的网站。clash的配置要支持ipv6倒是也简单，添加ipv6：true即可。还望广大机场能尽快跟进，默认配置中就加上这句话。



域名解析添加以后，开始几天倒是还是正常使用的。我一致以为ipv6那么多，运营商现在肯定是给每个用户一个固定的ipv6地址。后来才发现我多想了，现在依旧是动态的，而且是动态的前缀，这就导致我配置一次AAAA记录是不行的。等到路由器断线重连之后，家里网络的ipv6地址就换了个边。这种情况就需要DDNS登场了。



我用的腾讯云解析的dns，这个可能比较小众，反正是没有找到现成的代码。这个时候就发现程序员的主观能动性了，自己主动撸一个了。写起来倒是比较简单，腾讯云还可以主动生成大部分代码，剩下的就是拼拼凑凑了。放在ubuntun上跑起来也简单，直接定时任务没5分钟刷一次好了。具体代码如下：

```
package main

import (
	"fmt"
	"net"
	"regexp"
	"strings"

	"github.com/tencentcloud/tencentcloud-sdk-go/tencentcloud/common"
	"github.com/tencentcloud/tencentcloud-sdk-go/tencentcloud/common/errors"
	"github.com/tencentcloud/tencentcloud-sdk-go/tencentcloud/common/profile"
	dnspod "github.com/tencentcloud/tencentcloud-sdk-go/tencentcloud/dnspod/v20210323"
)

const dnsPodDomain = "dnspod.tencentcloudapi.com"
// 主域名，不包含www
const domain = ""
// 腾讯云secretId
const secretId = ""
// 腾讯云secretKey
const secretKey = ""

func main() {
	// 目标子域名
	subDomains := []string{"ipv6"}

	// 密钥
	credential := common.NewCredential(
		secretId,
		secretKey,
	)

	ipv6 := getMyIPV6()

	fmt.Printf("ipv6地址: %s\n", ipv6)

	cpf := profile.NewClientProfile()
	cpf.HttpProfile.Endpoint = dnsPodDomain
	client, _ := dnspod.NewClient(credential, "", cpf)

	request := dnspod.NewDescribeRecordListRequest()

	request.Domain = common.StringPtr(domain)

	response, err := client.DescribeRecordList(request)
	if _, ok := err.(*errors.TencentCloudSDKError); ok {
		fmt.Printf("An API error has returned: %s", err)
		return
	}
	if err != nil {
		panic(err)
	}

	for _, subDomain := range subDomains {
		for _, record := range response.Response.RecordList {
			if *record.Value == ipv6 {
				continue
			}

			if *record.Name == subDomain {
				fmt.Printf("%d\n", *record.RecordId)
				updateDomain(credential, subDomain, *record.RecordId, ipv6)
			}
		}
	}

	var name string
	fmt.Scanln(&name)

}

func updateDomain(credential *common.Credential, subDomain string, recordId uint64, ipv6Str string) {
	cpf := profile.NewClientProfile()
	cpf.HttpProfile.Endpoint = dnsPodDomain
	client, _ := dnspod.NewClient(credential, "", cpf)

	request := dnspod.NewModifyRecordRequest()

	request.Domain = common.StringPtr(domain)
	request.SubDomain = common.StringPtr(subDomain)
	request.RecordType = common.StringPtr("AAAA")
	request.RecordLine = common.StringPtr("默认")
	request.Value = common.StringPtr(ipv6Str)
	request.RecordId = common.Uint64Ptr(recordId)

	response, err := client.ModifyRecord(request)
	if _, ok := err.(*errors.TencentCloudSDKError); ok {
		fmt.Printf("An API error has returned: %s", err)
		return
	}
	if err != nil {
		panic(err)
	}
	fmt.Printf("%s\n", response.ToJsonString())
}

func getMyIPV6() string {
	s, err := net.InterfaceAddrs()
	if err != nil {
		fmt.Printf("get ipv6 err %s", err)
		return ""
	}
	for _, a := range s {
		i := regexp.MustCompile(`(\w+:){7}\w+`).FindString(a.String())
		if strings.Count(i, ":") == 7 {
			return i
		}
	}
	fmt.Printf("not get ipv6 address")
	return ""
}
```

上面脚本ubuntun上跑问题不大，但是在我的群晖上不大好搞。群晖上不大想装个go环境，暂时也没想把它编译成可执行文件的想法。后面想到可以放到群晖的docker上跑，docker选择跟主机同样的网络就好。这样定时任务或开机时就可以获取到群晖主机所有的ipv6地址，剩下的就都一样了。

