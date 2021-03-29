---
title:  “Tensorflow Serving”
mathjax: true
layout: post
date:   2019-07-01 08:00:12 +0800
categories: deeplearning.ai
---

在开始深度学习的项目，半年没有更新自己的博客了。最近在Tensorflow Serving遇到一点问题，于是记录一下吧。

Tensorflow Serving是部署Tensorflow模型的一个项目，对外提供了两种接口。第一种是REST API接口，于是可以通过Http发送预测请求：
```shell
# Query the model using the predict API
curl -d '{"instances": [1.0, 2.0, 5.0]}' \
    -X POST http://localhost:8501/v1/models/half_plus_two:predict
```


另一种接口是gRPC API接口。gRPC是一套由Google开发的远程过程调用系统(Remote Process Call)，客户端应用可以像调用本地对象一样
直接调用另一台不同机器上服务器端应用的方法。gRPC采用Protocol Buffer，是一个高效的序列化/反序列化协议。要测试Tensorflow Serving的
Inference性能，感觉gPRC API接口应该是一个更好的调用方式。

- 安装测试Protobuf和gRPC

# Prerequisite

```shell
$ apt install autoconf libtool gcc g++ cmake pkg-config
```

# Install Protobuf 3.7.0 & gRPC v1.20.0

```shell
# $(curl -L https://grpc.io/release) # v1.20.0
$ git clone -b v1.20.0 https://github.com/grpc/grpc
$ cd grpc
$ git submodule update --init

# Install protobuf
$ cd third_party/protobuf
$ git submodule update --init --recursive
$ ./autogen.sh
$ ./configure
$ make -j8 && make install
$ ldconfig

# Install grpc
$ cd ../../
$ make -j8 && make install
```

protobuf和grpc安装完毕，开始测试[grpc helloworld](https://github.com/grpc/grpc/tree/master/examples/cpp/helloworld):

```shell
.
├── BUILD
├── README.md
├── cpp
│   ├── helloworld
│   │   ├── CMakeLists.txt
│   │   ├── Makefile
│   │   ├── greeter_client.cc
│   │   └── greeter_server.cc
└── protos
    ├── helloworld.proto
```

`cmake`会遇到如下错误：
```
CMake Error at CMakeLists.txt:65 (find_package):
  Could not find a package configuration file provided by "Protobuf" with any
  of the following names:

    ProtobufConfig.cmake
    protobuf-config.cmake

  Add the installation prefix of "Protobuf" to CMAKE_PREFIX_PATH or set
  "Protobuf_DIR" to a directory containing one of the above files.  If
  "Protobuf" provides a separate development package or SDK, be sure it has
  been installed.
```

[google forum protobuf](https://groups.google.com/forum/#!topic/protobuf/BoWqjv1e84s)的讨论中，有人提供了解决方案，实测可行。
但是后面仍然会遇到gprc-config.cmake的问题，于是换条路用Makefile编译。简单修改一下Makefile后运行`make`：

```shell
$ make
protoc -I ../../protos --cpp_out=. ../../protos/helloworld.proto
g++ -std=c++11 `pkg-config --cflags protobuf grpc`  -c -o helloworld.pb.o helloworld.pb.cc
protoc -I ../../protos --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` ../../protos/helloworld.proto
g++ -std=c++11 `pkg-config --cflags protobuf grpc`  -c -o helloworld.grpc.pb.o helloworld.grpc.pb.cc
g++ -std=c++11 `pkg-config --cflags protobuf grpc`  -c -o greeter_client.o greeter_client.cc
g++ helloworld.pb.o helloworld.grpc.pb.o greeter_client.o -L/usr/local/lib `pkg-config --libs protobuf grpc++ grpc` -Wl,--no-as-needed -lgrpc++_reflection -Wl,--as-needed -ldl -o greeter_client
g++ -std=c++11 `pkg-config --cflags protobuf grpc`  -c -o greeter_server.o greeter_server.cc
g++ helloworld.pb.o helloworld.grpc.pb.o greeter_server.o -L/usr/local/lib `pkg-config --libs protobuf grpc++ grpc` -Wl,--no-as-needed -lgrpc++_reflection -Wl,--as-needed -ldl -o greeter_server

$ ls
CMakeCache.txt  Makefile           greeter_client.o   greeter_server.o       helloworld.grpc.pb.o  helloworld.pb.o
CMakeFiles      greeter_client     greeter_server     helloworld.grpc.pb.cc  helloworld.pb.cc      helloworld.proto
CMakeLists.txt  greeter_client.cc  greeter_server.cc  helloworld.grpc.pb.h   helloworld.pb.h
```

运行结果：

```shell
$ ./greeter_server
Server listening on 0.0.0.0:50051

$ ./greeter_client
Greeter received: Hello world
```

- Tensorflow Serving gRPC API

`serving/tensorflow_serving/example/resnet_client.cc`给出了一个C++客户端例子。`resnet_clent.cc`依赖`tensorflow/core`和`tensorflow_serving/apis`，
一开始的思路是先编译tensorflow和tensorflow_serving。

# Install bazel 0.25.2

```shell
$ ./bazel-0.25.2-installer-linux-x86_64.sh --user
```

# Tensorflow v1.14.0 C++ 

```shell
$ git clone -b v1.14.0 https://github.com/tensorflow/tensorflow.git

$ cd tensorflow
$ ./configure
$ bazel build --config=monolithic //tensorflow:libtensorflow_cc.so

# copy library file
cp ./bazel-bin/tensorflow/libtensorflow_cc.so /usr/local/lib/libtensorflow_cc.so

# copy header file
mkdir -p /usr/local/include/tensorflow
cp -RL ./tensorflow/core /usr/local/include/tensorflow
cp -RL ./tensorflow/cc /usr/local/include/tensorflow

mkdir -p /usr/local/include/third_party
cp -RL ./third_party/eigen3 /usr/local/include/third_party

cp -RLf ./bazel-tensorflow/external/eigen_archive/unsupported /usr/local/include

# pb.cc & pb.h
cp -RL ./bazel-genfiles/tensorflow/cc /usr/local/include/tensorflow
cp -RL ./bazel-genfiles/tensorflow/core /usr/local/include/tensorflow

cp -RL ./bazel-tensorflow/external/eigen_archive/Eigen /usr/local/include/Eigen
```

# Tensorflow Serving 1.14.0 C++

```shell
git clone -b 1.14.0 https://github.com/tensorflow/serving.git
cd serving
tools/run_in_docker.sh bazel build -c opt tensorflow_serving/...

cp -RL bazel-bin/tensorflow_serving/ /usr/local/lib/tensorflow_serving
cp -RL bazel-genfiles/tensorflow_serving/ /usr/local/include/tensorflow_serving
```

编译成功，然而在链接mnist_client.o的时候，即使我设置了LIBRARY_PATH和LD_LIBRARY_PATH，依然报错如下：

```shell
resnet_client.o: In function `tensorflow::TensorProto* google::protobuf::Arena::InternalHelper<tensorflow::TensorProto>::Construct<google::protobuf::Arena*&>(void*, google::protobuf::Arena*&)':
resnet_client.cc:(.text._ZN6google8protobuf5Arena14InternalHelperIN10tensorflow11TensorProtoEE9ConstructIJRPS1_EEEPS4_PvDpOT_[_ZN6google8protobuf5Arena14InternalHelperIN10tensorflow11TensorProtoEE9ConstructIJRPS1_EEEPS4_PvDpOT_]+0x44): undefined reference to `tensorflow::TensorProto::TensorProto(google::protobuf::Arena*)'
resnet_client.o: In function `tensorflow::serving::PredictionService::Stub::experimental_async::~experimental_async()':
resnet_client.cc:(.text._ZN10tensorflow7serving17PredictionService4Stub18experimental_asyncD2Ev[_ZN10tensorflow7serving17PredictionService4Stub18experimental_asyncD5Ev]+0xf): undefined reference to `vtable for tensorflow::serving::PredictionService::Stub::experimental_async'
collect2: error: ld returned 1 exit status
```

还不知道如何解决链接错误。绕过这个问题的一个方案是将`mnist_clent.cc`放到`serving/tensorflow_serving/example`目录下，修改BUILD文件，然后运行`bazel build //tensorflow_serving/example:mnist_client_cc`。

mnist_client.cc文件:
```cpp
/* Copyright 2017 Google Inc. All Rights Reserved.
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
==============================================================================*/

#include <fstream>
#include <iostream>

#include "grpcpp/create_channel.h"
#include "grpcpp/security/credentials.h"
#include "google/protobuf/map.h"
#include "tensorflow/core/framework/tensor.h"
#include "tensorflow/core/platform/types.h"
#include "tensorflow/core/util/command_line_flags.h"
#include "tensorflow_serving/apis/prediction_service.grpc.pb.h"

using grpc::Channel;
using grpc::ClientContext;
using grpc::Status;

using tensorflow::serving::PredictRequest;
using tensorflow::serving::PredictResponse;
using tensorflow::serving::PredictionService;

typedef google::protobuf::Map<tensorflow::string, tensorflow::TensorProto> OutMap;

class ServingClient {
 public:
  ServingClient(std::shared_ptr<Channel> channel)
      : stub_(PredictionService::NewStub(channel)) {}

  tensorflow::string callPredict(const tensorflow::string& model_name,
                                 const tensorflow::string& model_signature_name,
                                 const tensorflow::string& file_path) {
    PredictRequest predictRequest;
    PredictResponse response;
    ClientContext context;

    predictRequest.mutable_model_spec()->set_name(model_name);
    predictRequest.mutable_model_spec()->set_signature_name(
        model_signature_name);

    google::protobuf::Map<tensorflow::string, tensorflow::TensorProto>& inputs =
        *predictRequest.mutable_inputs();

    tensorflow::TensorProto proto;

    std::ifstream imageFile(file_path, std::ios::binary);

    if (!imageFile.is_open()) {
      std::cout << "Failed to open " << file_path << std::endl;
      return "";
    }

    std::filebuf* pbuf = imageFile.rdbuf();
    auto fileSize = 28 * 28;

    char* image = new char[fileSize]();

    // first 4 bytes of mnist binary file are: magic number, number of images, number of rows, number of columns
    pbuf->pubseekpos(4 * sizeof(int), std::ios::in);
    pbuf->sgetn(image, fileSize);
    imageFile.close();

    float *val = new float[fileSize]();
    for(int i = 0; i < fileSize; i++)
        val[i] = image[i] / 255.0;

    proto.set_dtype(tensorflow::DataType::DT_FLOAT);
    for(int i = 0; i < fileSize; i++)
        proto.add_float_val(val[i]);

    proto.mutable_tensor_shape()->add_dim()->set_size(1);
    proto.mutable_tensor_shape()->add_dim()->set_size(fileSize);

    inputs["images"] = proto;

    Status status = stub_->Predict(&context, predictRequest, &response);

    delete[] image;
    delete[] val;

    if (status.ok()) {
      std::cout << "call predict ok" << std::endl;
      std::cout << "outputs size is " << response.outputs_size() << std::endl;
      OutMap& map_outputs = *response.mutable_outputs();
      OutMap::iterator iter;
      int output_index = 0;

      for (iter = map_outputs.begin(); iter != map_outputs.end(); ++iter) {
        tensorflow::TensorProto& result_tensor_proto = iter->second;
        tensorflow::Tensor tensor;
        bool converted = tensor.FromProto(result_tensor_proto);
        if (converted) {
          std::cout << "the result tensor[" << output_index
                    << "] is:" << std::endl
                    << tensor.SummarizeValue(10) << std::endl;
        } else {
          std::cout << "the result tensor[" << output_index
                    << "] convert failed." << std::endl;
        }
        ++output_index;
      }
      return "Done.";
    } else {
      std::cout << "gRPC call return code: " << status.error_code() << ": "
                << status.error_message() << std::endl;
      return "gRPC failed.";
    }
  }

 private:
  std::unique_ptr<PredictionService::Stub> stub_;
};

int main(int argc, char** argv) {
  tensorflow::string server_port = "localhost:9000";
  tensorflow::string image_file = "/tmp/t10k-images-idx3-ubyte";
  tensorflow::string model_name = "mnist";
  tensorflow::string model_signature_name = "predict_images";
  std::vector<tensorflow::Flag> flag_list = {
      tensorflow::Flag("server_port", &server_port,
                       "the IP and port of the server"),
      tensorflow::Flag("image_file", &image_file, "the path to the image"),
      tensorflow::Flag("model_name", &model_name, "name of model"),
      tensorflow::Flag("model_signature_name", &model_signature_name,
                       "name of model signature")};

  tensorflow::string usage = tensorflow::Flags::Usage(argv[0], flag_list);
  const bool parse_result = tensorflow::Flags::Parse(&argc, argv, flag_list);
  if (!parse_result || image_file.empty()) {
    std::cout << usage;
    return -1;
  }

  ServingClient guide(
      grpc::CreateChannel(server_port, grpc::InsecureChannelCredentials()));
  std::cout << "calling predict using file: " << image_file << "  ..."
            << std::endl;
  std::cout << guide.callPredict(model_name, model_signature_name, image_file)
            << std::endl;
  return 0;
}
```

另外一个可能更好的方案是建一个bazel项目，文件结构如下，然后运行`bazel build --define=grpc_no_ares=true :mnist_client_cc`。
```shell
.
├── BUILD
├── mnist_client.cc
├── tf_repo.bzl
└── WORKSPACE
```

WORKSPACE文件：
```
workspace(name = "mnist_client")

load("//:tf_repo.bzl", "tensorflow_http_archive")

tensorflow_http_archive(
    name = "org_tensorflow",
    sha256 = "c4da79385dfbfb30c1aaf73fae236bc6e208c3171851dfbe0e1facf7ca127a6a",
    git_commit = "87989f69597d6b2d60de8f112e1e3cea23be7298",
)

load("@bazel_tools//tools/build_defs/repo:git.bzl", "git_repository")

git_repository(
    name = "tf_serving",
    commit = "34d9e855d94bb87614603e0ae91e90674eea68be",
    remote = "https://github.com/tensorflow/serving.git",
)

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# START: Upstream TensorFlow dependencies
# TensorFlow build depends on these dependencies.
# Needs to be in-sync with TensorFlow sources.
http_archive(
    name = "io_bazel_rules_closure",
    sha256 = "ddce3b3a3909f99b28b25071c40b7fec7e2e1d1d1a4b2e933f3082aa99517105",
    strip_prefix = "rules_closure-316e6133888bfc39fb860a4f1a31cfcbae485aef",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/rules_closure/archive/316e6133888bfc39fb860a4f1a31cfcbae485aef.tar.gz",
        "https://github.com/bazelbuild/rules_closure/archive/316e6133888bfc39fb860a4f1a31cfcbae485aef.tar.gz",  # 2019-03-21
    ],
)

http_archive(
    name = "bazel_skylib",
    sha256 = "2c62d8cd4ab1e65c08647eb4afe38f51591f43f7f0885e7769832fa137633dcb",
    strip_prefix = "bazel-skylib-0.7.0",
    urls = ["https://github.com/bazelbuild/bazel-skylib/archive/0.7.0.tar.gz"],
)
# END: Upstream TensorFlow dependencies

# Please add all new TensorFlow Serving dependencies in workspace.bzl.
load("@tf_serving//tensorflow_serving:workspace.bzl", "tf_serving_workspace")

tf_serving_workspace()

# Specify the minimum required bazel version.
load("@org_tensorflow//tensorflow:version_check.bzl", "check_bazel_version_at_least")

check_bazel_version_at_least("0.24.1")
```

tf_repo.bzl文件:
```
""" TensorFlow Http Archive

Modified http_archive that allows us to override the TensorFlow commit that is
downloaded by setting an environment variable. This override is to be used for
testing purposes.

Add the following to your Bazel build command in order to override the
TensorFlow revision.

build: --action_env TF_REVISION="<git commit hash>"

  * `TF_REVISION`: tensorflow revision override (git commit hash)
"""

_TF_REVISION = "TF_REVISION"

def _tensorflow_http_archive(ctx):
    git_commit = ctx.attr.git_commit
    sha256 = ctx.attr.sha256

    override_git_commit = ctx.os.environ.get(_TF_REVISION)
    if override_git_commit:
        sha256 = ""
        git_commit = override_git_commit

    strip_prefix = "tensorflow-%s" % git_commit
    urls = [
        "https://mirror.bazel.build/github.com/tensorflow/tensorflow/archive/%s.tar.gz" % git_commit,
        "https://github.com/tensorflow/tensorflow/archive/%s.tar.gz" % git_commit,
    ]
    ctx.download_and_extract(
        urls,
        "",
        sha256,
        "",
        strip_prefix,
    )

tensorflow_http_archive = repository_rule(
    implementation = _tensorflow_http_archive,
    attrs = {
        "git_commit": attr.string(mandatory = True),
        "sha256": attr.string(mandatory = True),
    },
)
```

BUILD文件:
```
cc_binary(
    name = "mnist_client_cc",
    srcs = [
        "mnist_client.cc",
    ],
    deps = [
        "@tf_serving//tensorflow_serving/apis:prediction_service_proto",
        "@grpc//:grpc++",
        "@org_tensorflow//tensorflow/core:framework",
        "@org_tensorflow//tensorflow/core:lib",
        "@protobuf_archive//:protobuf_lite",
    ],
)
```

测试mnist_client_cc

```shell
calling predict using file: /tmp/t10k-images-idx3-ubyte  ...
call predict ok
outputs size is 1
the result tensor[0] is:
[0.0518000722 0.112475552 0.0884227455 0.0608940311 0.0767830536 0.31831342 0.0678093731 0.147692293 0.0164375585 0.0593718737]
Done.
```

参考:  
[bazel Installzation](https://docs.bazel.build/versions/master/install-ubuntu.html#install-on-ubuntu)  
[grpc cmake](https://www.cnblogs.com/jason1990/p/10381320.html)  
[Creating Tensorflow C++ headers and libraries](http://blog.blitzblit.com/2017/06/11/creating-tensorflow-c-headers-and-libraries/)  
[Tensorflow Serving Installation](https://www.tensorflow.org/tfx/serving/setup)  
[Tensorflow Serving填坑记](https://oyeblog.com/2018/tensorflow_serving/)  
[C++静态库与动态库](https://www.cnblogs.com/skynet/p/3372855.html)  
[详解bazel](https://blog.csdn.net/u013510838/article/details/80102438)