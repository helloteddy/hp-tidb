# High Performance TiDB 第一章练习

操作系统版本：CentOS Linux release 7.6.1810 (Core)

tidb的组件包含：tidb-server、pd、tikv。所以要运行一个tidb需要编译三个组件。
tidb-server和pd都是go语言，但是tikv是rust。
需要安装go语言、rust语言的环境。

### Go 语言环境安装
执行命令：
```
yum install goland
```
安装完成后查看go 版本
```
[root@3 ~]# go version
go version go1.13.14 linux/amd64
```
设置环境变量
```
mkdir ~/workspace
echo 'export GOPATH="$HOME/workspace"' >> ~/.bashrc
source ~/.bashrc
```
设置go env
```
go env -w GOPROXY=https://goproxy.io,direct
go env -w GO111MODULE=on
```
查看设置后的go env
```
go env
```

### rust语言环境安装
先设置临时变量，加速安装
```
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
```
执行安装命令
```
curl https://sh.rustup.rs -sSf | sh
```
刷新环境变量
```
source $HOME/.cargo/env
```
检查cargo 、rust的安装
```
cargo —version
rustc --version
```
修改cargo 源
```
vi ~/.cargo/config
```
增加以下内容
```
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'ustc'
[source.ustc]
registry = "https://mirrors.ustc.edu.cn/crates.io-index"
```
至此环境搭建完成。

### 编译组件。
#### 编译pd
下载源代码
git clone https://github.com/pingcap/pd.git
开始编译pd
make build


#### 编译tidb
下载源代码
```
git clone https://github.com/pingcap/tidb.git
```
开始编译tidb
```
make build
```


#### 编译tikv
下载源代码
```
git clone https://github.com/tikv/tikv.git
```
编译tikv
```
make build
```
编译中间需要下载相关组件，特别慢，需要耐心



### 启动组件

#### pd组件启动命令
```
nohup bin/pd-server --name=pd1 --client-urls=http://127.0.0.1:2379 --peer-urls=http://127.0.0.1:2380 --data-dir=/Users/xiexiong/data/pd > pd.log 2>&1 &
```

#### tikv 组件启动命令
```
nohup bin/tikv-server -C bin/tikv-config.toml --addr 127.0.0.1:20160 --advertise-addr 127.0.0.1:20160 --pd 127.0.0.1:2379 --data-dir /Users/xiexiong/data/tikv > tikv.log 2>&1 &
```
tikv-config.toml 参考https://github.com/tikv/tikv/blob/master/etc/config-template.toml
启动多个实例时记得修改地址端口

#### tidb组件启动
```
nohup bin/tidb-server -store tikv -path '127.0.0.1:2379/pd?cluster=1' -P 4001 --data-dir /Users/xiexiong/data/tidb > tidb.log 2>&1 &
```

启动完成之后可以通过mysql 来访问tidb了
```
mysql -uroot -P4000
```


### 作业：使得tidb每启动一个事物时都会输出：hello transaction

需要修改的组件是tidb组件
参考文章https://pingcap.com/blog-cn/tidb-source-code-reading-3/
1. 处理客户端传输过来的请求的是server/conn.go文件里面的各种handle方法
2. 跟踪方法会走到Session.go 的Execute方法中，进而走到ExecuteStmt方法中
3. 阅读ExecuteStmt方法可以找到生成事物的方法PrepareTxnCtx方法
4. 在方法中加入输出所需字符即完成
修改部分代码如下：
```go
func (s *session) PrepareTxnCtx(ctx context.Context) {
   if s.txn.validOrPending() {
      return
   }
   log.Info("hello transaction")
   is := domain.GetDomain(s).InfoSchema()
   s.sessionVars.TxnCtx = &variable.TransactionContext{
      InfoSchema:    is,
      SchemaVersion: is.SchemaMetaVersion(),
      CreateTime:    time.Now(),
      ShardStep:     int(s.sessionVars.ShardAllocateStep),
   }
    …. 省略
}
```

重新编译tidb代码，之后重新运行各个组件。可以在tidb.log查看对应的输出
```
[2020/08/16 22:43:49.913 +08:00] [INFO] [server.go:235] ["server is running MySQL protocol"] [addr=0.0.0.0:4000]
[2020/08/16 22:43:49.913 +08:00] [INFO] [http_status.go:82] ["for status and metrics report"] ["listening on addr"=0.0.0.0:10080]
[2020/08/16 22:43:49.913 +08:00] [INFO] [session.go:2169] ["hello transaction"]
[2020/08/16 22:43:49.915 +08:00] [INFO] [domain.go:1094] ["init stats info time"] ["take time"=2.157604ms]
[2020/08/16 22:43:52.913 +08:00] [INFO] [session.go:2169] ["hello transaction"]
[2020/08/16 22:43:52.913 +08:00] [INFO] [session.go:2169] ["hello transaction"]
[2020/08/16 22:43:52.913 +08:00] [INFO] [session.go:2169] ["hello transaction"]
[2020/08/16 22:43:52.914 +08:00] [INFO] [session.go:2169] ["hello transaction"]
[2020/08/16 22:43:55.910 +08:00] [INFO] [session.go:2169] ["hello transaction"]
```

任务完成
