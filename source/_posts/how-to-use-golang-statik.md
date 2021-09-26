---
title: 使用golang的statik工具打包，并访问特定资源（文本、图片、二进制等）
date: 2021-09-24 00:34:55
author: ice
img: /images/statik-1.jpg
top: true
cover: true
coverImg: /images/statik-1.jpg
password: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
toc: true
mathjax: false
summary: golang的statik工具使用方法
categories: hexo
tags:
  - golang工具库
  - travis-ci
---

## 1. statik是什么？
### 1.1 官网的介绍
官网：https://github.com/rakyll/statik 是这么描述的："statik allows you to embed a directory of static files into your Go binary to be later served from an http.FileSystem.
意思是这个statik工具能够将一些静态文件嵌入到go的可执行二进制中，并且能通过http协议来访问这些静态文件。

### 1.2 流程图
![statik使用](/images/statik使用.png)

## 2. 怎么使用
### 2.1 安装
```
go get github.com/rakyll/statik
```

### 2.2 使用方式
使用起来很简单，通过`-src`指定一个目录就可以把整个目录的所有文件打包起来，写入一个叫statik.go的文件里，如果不存在这个文件，则通过`go generate -x`来生成。需要注意，这里的"目录里的所有文件"，是递归的，比如下边的public目录，还有子目录tmp，tmp目录里有一个文件f，则也会包含文件f。
```
statik -src=/path/to/your/project/public
```

## 3. 实际案例
### 3.1 使用实例1
> 将某个目录下一个文本文件和一张图片，打包到二进制中，启动二进制后通过http访问这两个文件。

首先，我们看下目录结构。
![statik实例1](/images/statik-1.png)
关注的重点是`main.go`里的逻辑，第一行是一个go 1.4引入的go generate语法，在`run`主函数之前先执行`go generate -x`，`-x`是打印执行的命令；main函数分2步：第1步是生成一个FileSystem实例，fs.New()这一行代码实际会读取statik目录下的statik.go里的二进制，这些二进制实际上就是静态文件；第2步是暴露一个http端口，用于通过http协议和statik自带的FileSystem访问这2个静态资源。
```
//go:generate statik -src=./public

package main

import (
	"log"
	"net/http"

	_ "github.com/rakyll/statik/example/statik"
	"github.com/rakyll/statik/fs"
)

// Before buildling, run go generate.
// Then, run the main program and visit http://localhost:8080/public/hello.txt
func main() {
	statikFS, err := fs.New()
	if err != nil {
		log.Fatal(err)
	}

	http.Handle("/public/", http.StripPrefix("/public/", http.FileServer(statikFS)))
	http.ListenAndServe(":8080", nil)
}
```

访问效果：
```
❯ curl -v  http://localhost:8080/public/hello.txt
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> GET /public/hello.txt HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< Accept-Ranges: bytes
< Content-Length: 13
< Content-Type: text/plain; charset=utf-8
< Last-Modified: Tue, 08 Dec 2020 15:44:30 GMT
< Date: Thu, 23 Sep 2021 16:51:49 GMT
< 
Hello World

* Connection #0 to host localhost left intact
* Closing connection 0

```

### 3.2 使用实例2.
> 将二进制B嵌入到二进制A中，然后执行A时也能执行B的逻辑。

这种场景笔者在工作当中碰到了，工作中用到的内部后端golang框架，自带了一些命令，如`xxx create`是生成工程目录结构。有个需求是这样的：增加一个`xxx sql2struct`命令，这个命令能够将输入的sql语句，一键转化为golang的struct。笔者在xorm命令行工具看到了`xorm reverse`命令有类似的功能，于是考虑怎么将这个能力融入到内部框架中。
先看下xorm reverse的文档，看着是可以根据建表语句来生成go语言的struct，于是想到一个思路：能否通过statik工具，将xorm打入到内部框架中的命令行工具`xxx`中。
```
>xorm help reverse
usage: xorm reverse [-s] driverName datasourceName tmplPath [generatedPath] [tableFilterReg]

according database's tables and columns to generate codes for Go, C++ and etc.

    -s                Generated one go file for every table
    driverName        Database driver name, now supported four: mysql mymysql sqlite3 postgres
    datasourceName    Database connection uri, for detail infomation please visit driver's project page
    tmplPath          Template dir for generated. the default templates dir hasprovide 1 template
    generatedPath     This parameter is optional, if blank, the default value is models, then will
                      generated all codes in models dir
    tableFilterReg    Table name filter regexp
```

动手吧，先组织一个目录结构，根目录放了`xxx`的主逻辑，assets目录放了2个文本文件a（内容是aaa）和b，然后是一个叫`embedmain`的二进制可执行程序，注意`_ "git.code.oa.com/iceewei/icee_test/common/statik/testStatikFS/statik"`这一行非常关键，是将打包好的二进制内容引入，fs.New()调用时会用到二进制内容，我们执行main函数时做2件事：执行embedmain的逻辑、读取文本文件a的内容并打印，b文件暂时不用。
```
❯ tree
.
├── assets
│   ├── a
│   ├── embedmain
│   └── tmp
│       └── b
└── main.go

2 directories, 4 files
```

`xxx`表示内部框架，对应`main.go`，这里有个细节：笔者是将内嵌的二进制`embedmain`先通过statik的FileSytem释放到本地的`/tmp`目录，然后通过`exec.Command函数`来调用`embedmain`的代码逻辑，最后删除掉这个多余的文件。
```
//go:generate statik -src=./assets
//go:generate go fmt statik/statik.go

package main

import (
	"fmt"
	_ "git.code.oa.com/iceewei/icee_test/common/statik/testStatikFS/statik"
	"github.com/rakyll/statik/fs"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"os/exec"
)

const (
	EMBED_BINARY_DIR = "/tmp/"
	EMBED_BINARY_PATH = "/tmp/embedmain"
)

// Before buildling, run go generate.
func main() {

	// 1. 初始化statik文件系统，携带了statik.go里注册的二进制内容。
	statikFS, err := fs.New()
	if err != nil {
		log.Fatalf("new statik file system failed.")
	}

	// 2. task1: 执行二进制B三个子步骤：新建临时目录放入二进制、执行、移除二进制。
	prepare(statikFS)
	execBinary()
	deleteTmpBinary()

	// 3. task2: 读一个文本文件，打印内容。
	file, err := statikFS.Open("/a")
	if err != nil {
		log.Fatalf("open file a failed, err=%v", err)
	}

	content, err := ioutil.ReadAll(file)
	if err != nil {
		log.Fatalf("read file a failed, err=%v", err)
	}
	fmt.Printf("content: %s\n", content)
}

func deleteTmpBinary() {
	err := os.Remove(EMBED_BINARY_PATH)
	fmt.Printf("remove tmp binary succeed? %t\n", err == nil)
}

func prepare(statikFS http.FileSystem) {

	file, err := statikFS.Open("/embedmain")
	if err != nil {
		log.Fatalf("open err: %v\n", err)
	}
	content, err := ioutil.ReadAll(file)
	if err != nil {
		log.Fatalf("read binary err: %v\n", err)
	}

	if exists, _ := pathExists(EMBED_BINARY_DIR); exists == false {
		err = os.Mkdir(EMBED_BINARY_DIR, os.ModePerm)
		if err != nil {
			log.Fatalf("mkdir EMBED_BINARY_DIR:%s failed: %v\n", EMBED_BINARY_DIR, err)
		} else {
			err = os.Chmod(EMBED_BINARY_DIR, os.ModePerm)
			if err != nil {
				log.Fatalf("cmmod binary dir %s to %v, err=%v", EMBED_BINARY_DIR, os.ModePerm, err)
			}
		}
	}

	err = ioutil.WriteFile(EMBED_BINARY_PATH, content, 0777)
	if err != nil {
		log.Fatalf("write to %s failed: %v\n", EMBED_BINARY_PATH, err)
	}

	err = os.Chmod(EMBED_BINARY_PATH, os.ModePerm)
	if err != nil {
		log.Fatalf("cmmod binary path %s to %v, err=%v", EMBED_BINARY_PATH, os.ModePerm, err)
	}
	fmt.Printf("write to %s sucessfully...\n", EMBED_BINARY_PATH)
}

func execBinary() {
	output, err := exec.Command(EMBED_BINARY_PATH).CombinedOutput()
	fmt.Println(string(output), err)
}

func pathExists(path string) (bool, error) {
	_, err := os.Stat(path)
	if err == nil {
		return true, nil
	}
	if os.IsNotExist(err) {
		return false, nil
	}
	return false, err
}
```

而`xorm`用以下函数模拟。要实现的效果就是执行`xxx`后能自动调起xorm里的代码，即打印"running embed main func."。
```
package main

import (
	"fmt"
)

func main() {
	fmt.Println("running embed main func.")
}
```

首先生成statik.go，执行`go generate -x`即可，然后go run main.go即可看到效果：
```
❯ go generate -x
statik -src=./assets
go fmt statik/statik.go
❯ go run main.go
write to /tmp/embedmain sucessfully...
running embed main func.
 <nil>
remove tmp binary succeed? true
content: aaa

```

## 4. 参考文档
- golang 中使用 statik 将静态资源编译进二进制文件中：https://blog.fatedier.com/2016/08/01/compile-assets-into-binary-file-with-statik-in-golang/
- statik官网：https://github.com/rakyll/statik