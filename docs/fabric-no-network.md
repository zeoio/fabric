## fabric-gm自包含文档

此文档旨在，提高fabric的构建过程，去除了编译`make docker`时候的网络依赖，使编译可以快速，正常的通过，无需在翻墙。

当我们执行`make docker`的时候， 会先构建`gotools`。它会从网络里下载一些二进制文件。然后包含到镜像中。 所以我们现在先来去掉这部分的网络依赖。

这些需要下载的二进制文件，在`gotools.mk`中。可以从第12行到第19行，和从第29行到37行看出：

```shell
# go tool->path mapping
go.fqp.counterfeiter := github.com/maxbrunsfeld/counterfeiter
go.fqp.gocov         := github.com/axw/gocov/gocov
go.fqp.gocov-xml     := github.com/AlekSi/gocov-xml
go.fqp.goimports     := golang.org/x/tools/cmd/goimports
go.fqp.golint        := golang.org/x/lint/golint
go.fqp.manifest-tool := github.com/estesp/manifest-tool
go.fqp.misspell      := github.com/client9/misspell/cmd/misspell
go.fqp.mockery       := github.com/vektra/mockery/cmd/mocker

# Special override for protoc-gen-go since we want to use the version vendored with the project
gotool.protoc-gen-go:
	@echo "Building github.com/golang/protobuf/protoc-gen-go -> protoc-gen-go"
	GOBIN=$(abspath $(GOTOOLS_BINDIR)) go install ./vendor/github.com/golang/protobuf/protoc-gen-go

# Special override for ginkgo since we want to use the version vendored with the project
gotool.ginkgo:
	@echo "Building github.com/onsi/ginkgo/ginkgo -> ginkgo"
	GOBIN=$(abspath $(GOTOOLS_BINDIR)) go install ./vendor/github.com/onsi/ginkgo/ginkgo

```

所以我们现在需要先手动的下载这些二进制文件：

```shell
go get github.com/maxbrunsfeld/counterfeiter
go get github.com/axw/gocov/gocov
go get github.com/AlekSi/gocov-xml
go get golang.org/x/tools/cmd/goimports
go get golang.org/x/lint/golint
go get github.com/estesp/manifest-tool
go get github.com/client9/misspell/cmd/misspell
go get github.com/vektra/mockery/cmd/mocker
go get github.com/golang/protobuf/protoc-gen-go
go get github.com/onsi/ginkgo/ginkgo
```

注意在下载这些二进制文件的时候，如果出现网络错误，请先翻墙，在重新下载。下载好的文件，会默认的保存到`$GOPATH/bin`目录下。

所以现在，我们先在代码目录下，新建一个`build/bin`目录，然后把这些二进制文件复制到此目录中。

```shell
make  -p build/bin

cp $GOPATH/bin/counterfeiter build/bin/
cp $GOPATH/bin/gocov build/bin/
...
cp $GOPATH/bin/protoc-gen-go build/bin/
cp $GOPATH/bin/ginkgo build/bin/
```

我记得有个简单的shell命令，可以一行的复制指定的文件，但是忘记了 : (

我们弄好这些依赖的二进制文件之后， 就修改`makefile`文件，把它们都包含进来。在第255行添加下面的代码：

```shell
## 原来的代码
$(BUILD_DIR)/docker/gotools: gotools.mk
	@echo "Building dockerized gotools"
	@mkdir -p $@/bin $@/obj
	@$(DRUN) \
		-v $(abspath $@):/opt/gotools \
		-w /opt/gopath/src/$(PKGNAME) \
		$(BASE_DOCKER_NS)/fabric-baseimage:$(BASE_DOCKER_TAG) \
		make -f gotools.mk GOTOOLS_BINDIR=/opt/gotools/bin GOTOOLS_GOPATH=/opt/gotools/obj

## 修改后的代码
$(BUILD_DIR)/docker/gotools: gotools.mk
	@echo "Building dockerized gotools"
	@mkdir -p $@/bin $@/obj
	@$(DRUN) \
		-v $(abspath $@):/opt/gotools \
		-v $(abspath build/bin):/opt/gotools/tmp \
		-w /opt/gopath/src/$(PKGNAME) \
		$(BASE_DOCKER_NS)/fabric-baseimage:$(BASE_DOCKER_TAG) \
		make -f gotools.mk GOTOOLS_BINDIR=/opt/gotools/bin GOTOOLS_GOPATH=/opt/gotools/obj
```

把我们准备好的二进制文件，挂载到docker容器中。然后复制这些文件，到指定的golang二进制目录中。所以我们现在来修改`gotools.mk`文件，执行此操作。注释掉第22行，并将其修改成如下：

```shell
## 修改之前的
.PHONY: gotools-install
gotools-install: $(patsubst %,$(GOTOOLS_BINDIR)/%, $(GOTOOLS))

## 修改之后的
.PHONY: gotools-install
gotools-install: 
	cp /opt/gotools/tmp/* $(GOTOOLS_BINDIR)

```

到现在为止，我们修改好了第一步。但是在`makefile`中，还有一处需要修改。在代码的第226行，我们先把需要的文件下载下来，并把它放到`build`目录下。

```shell
curl -fL https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/chaintool-1.1.3/hyperledger-fabric-chaintool-1.1.3.jar > build/chaintool
```

上面那个链接是怎么来的？ 直接运行`make docker`，就可以看到，懒得再分析makefile代码。之后，我们修改第226行的代码，使用下载好的文件。

```shell
## 原来的代码
$(BUILD_DIR)/%/chaintool: Makefile
	@echo "Installing chaintool"
	@mkdir -p $(@D)
	curl -fL $(CHAINTOOL_URL) > $@
	chmod +x $@

## 现在的代码
$(BUILD_DIR)/%/chaintool: Makefile
	@echo "Installing chaintool"
	@mkdir -p $(@D)
	cp build/chaintool $@
	chmod +x $@
```

到目前为止，我们修改了2处，还有最后一处需要修改。在文件`images/tools/Dockerfile.in`中。在代码第16行通过`apt-get`安装了`jq`，我们先手动的下载`jq`的deb包，然后在添加进来进行安装。`jq`包依赖一个`libonig2`的包。所以也需要把这个包下载下来。

我们先创建一个目录，来存放这些下载的包文件：

```shell
mkdir build/deb
```

然后从网上手动的下载那两个包文件，放到`build/deb/`目录下。

之后，我们修改这个dockerfile文件。在第16行处添加：

```shell
## 原来的代码
RUN apt-get update && apt-get install -y jq

## 现在的代码
ADD build/deb/*.deb /tmp/
RUN dpkg -i /tmp/libonig2.deb
RUN dpkg -i /tmp/jq.deb
```

一切都已经搞定了, 大家可以自己去测试一下。
