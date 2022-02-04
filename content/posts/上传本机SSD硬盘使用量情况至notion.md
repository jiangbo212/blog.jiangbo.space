---
title: "上传本机SSD硬盘使用量情况至notion"
date: 2022-01-25T23:00:59+08:00
draft: false
categories: 技术
---

M1的Mac有个让人比较关注的点，就是它的ssd硬盘的写入量相比于以前有个很大的提高。写入量的提高可能会导致磁盘提前报废，之前也在网络上引起了很大的讨论。



我这台Mac买来的时候，因为是官翻机的缘故。当时已经有3个T的写入量了。开始的时候我几乎天天观察，每天坚持写入量。但是一连十几天都没什么大的变化。后面也就没怎么看了。最近再看，发现写入量已经达到16.5T了，这个变化有个大，毕竟也就买回来不到4个月。



为了能够不再被蒙在鼓里，我决定写个脚本，每天上传SSD硬盘的信息到notion中，这样就不会稀里糊涂的看到硬盘写入量飙升，详细代码如下：

```
package main

import (
	"bytes"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"os/exec"
	"strings"
	"time"
)

func main() {
	fmt.Println("Hello World")
	out, err := RunCommand("/opt/homebrew/Cellar/smartmontools/7.2/bin/smartctl", "-a", "/dev/disk0")
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(out)
	out = strings.ReplaceAll(out, "\x00", "")
	outs := strings.Split(out, "\n")
	var written string
	var read string
	for _, str := range outs {
		if strings.HasPrefix(str, "Data Units Written") {
			fmt.Println(str)
			writtens := strings.Split(str, "[")
			fmt.Println(writtens[1])
			written = strings.Replace(writtens[1], "]", "", 1)
		}
		if strings.HasPrefix(str, "Data Units Read") {
			fmt.Println(str)
			reads := strings.Split(str, "[")
			fmt.Println(reads[1])
			read = strings.Replace(reads[1], "]", "", 1)
		}
	}
	AddNotionTable(written, read)
}

func RunCommand(name string, arg ...string) (string, error) {
	cmd := exec.Command(name, arg...)
	log.Println(cmd.String())
	// 命令的错误输出和标准输出都连接到同一个管道
	stdout, err := cmd.StdoutPipe()
	cmd.Stderr = cmd.Stdout
	var out string = ""
	if err != nil {
		return out, err
	}
	if err = cmd.Start(); err != nil {
		return out, err
	}
	// 从管道中实时获取输出并打印到终端
	for {
		tmp := make([]byte, 1024)
		_, err := stdout.Read(tmp)
		// fmt.Print(string(tmp))
		out += string(tmp)
		if err != nil {
			break
		}
	}
	if err = cmd.Wait(); err != nil {
		return out, err
	}
	return out, nil
}

func AddNotionTable(written string, read string) {
	url := "https://api.notion.com/v1/pages"
	requestJson := `
	{
		"parent": {
			"type": "database_id",
      // 获取自己的database_id
			"database_id": ""
		},
		"properties": {
			"date": {
				"type": "title",
				"title": [
					{
						"type": "text",
						"text": {
							"content": "${date}"
						}
					}
				]
			},
			"written": {
				"type": "rich_text",
				"rich_text": [{
					"type": "text",
					"text": {
						"content": "${written}"
					}
				}]
			},
			"read": {
				"type": "rich_text",
				"rich_text": [{
					"type": "text",
					"text": {
						"content": "${read}"
					}
				}]
			}
		}
	}
	`
	requestJson = strings.Replace(requestJson, "${date}", time.Now().Format("2006-01-02"), 1)
	requestJson = strings.Replace(requestJson, "${written}", written, 1)
	requestJson = strings.Replace(requestJson, "${read}", read, 1)

	req, _ := http.NewRequest("POST", url, bytes.NewBuffer([]byte(requestJson)))
  // 获取自己的notion密钥
	req.Header.Set("Authorization", "Bearer ")
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Notion-Version", "2021-08-16")

	client := &http.Client{}
	resp, err := client.Do(req)
	if err != nil {
		fmt.Println(err)
	}
	defer resp.Body.Close()

	fmt.Println("status", resp.Status)
	// fmt.Println("response:", resp.Header)
	body, _ := ioutil.ReadAll(resp.Body)
	fmt.Println("response Body:", string(body))

}
```

最终的效果如下图

![效果图](/img/WX20220125-225909@2x-1024x667.png)
