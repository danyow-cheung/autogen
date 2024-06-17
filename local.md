# 2024.6.13
因为直接拉取docker镜像的时候,网络连接问题
```
docker build -f .devcontainer/full/Dockerfile -t autogen_full_img https://github.com/microsoft/autogen.git#main
```
直接clone到本地然后再
```
# 还遇到一些小错误 https://stackoverflow.com/questions/46517241/docker-build-requires-exactly-1-arguments
cd autogen 
docker build -f .devcontainer/full/Dockerfile -t autogen_full_image .
# 无非还是一些网络问题,docker换源,直接拉取基镜像etc
docker pull python:3.11-slim-bookworm

```



结果新的问题已经出现
```
E: Unable to locate package curl
Unable to install curl! Your base system has a problem; please check your default OS's package repositories because curl should work.
```
不死心又试一次
```
docker build -f .devcontainer/full/Dockerfile -t autogen_full_img https://github.com/microsoft/autogen.git#main
```

# 2024.6.14
当前镜像构建完成,并上传到harbor
```
docker tag autogen_full_img:latest 192.168.0.249:9989/autogen/autogen_full_img:v0
docker push 192.168.0.249:9989/autogen/autogen_full_img:v0
```

开始构建容器应用
```
# 模板docker命令
docker run -it -v $(pwd)/myapp:/home/autogen/autogen/myapp autogen_base_img:latest python /home/autogen/autogen/myapp/main.py
# custom, 挂载目录并且运行文件
docker run -it -v /home/hs/common/code/agnet_etc/autogen_code/myapp:/home/autogen/autogen/myapp autogen_base_img:latest python /home/autogen/autogen/myapp/main.py

# 上面那句有部分报错,现在这个是开启容器然后在容器里面进行操作
docker run -it -v /home/hs/common/code/agnet_etc/autogen_code/myapp:/home/autogen/autogen/myapp autogen_full_img:latest 

```

还是很方便的,直接在`/home/hs/common/code/agnet_etc/autogen_code/myapp`进行修改,容器里面就对应变化


遇到一个报错
```
docker.errors.DockerException: Error while fetching server API version: ('Connection aborted.', FileNotFoundError(2, 'No such file or directory'))
```
换了一个还是错,后面发现是因为代码的问题,尝试了one api 是可以调用的

大概都走一下这个的流程就差不多了 
https://microsoft.github.io/autogen/docs/tutorial

主要工作量在 `main.py`


在代码自动执行的部分的时候,出现报错
```
docker.errors.DockerException: Error while fetching server API version: ('Connection aborted.', FileNotFoundError(2, 'No such file or directory'))
```

# 2024.6.17
增加了这一部分,https://github.com/docker/docker-py/issues/3256#issuecomment-2121000434.但是没啥效果.

用宿主机跑 `code_main.py`出现错误
```
docker.errors.ImageNotFound: 404 Client Error for http+docker://localhost/v1.43/images/create?tag=latest&fromImage=3.11-slim-bookworm: Not Found ("pull access denied for 3.11-slim-bookworm, repository does not exist or may require 'docker login': denied: requested access to the resource is denied")
```
然后解决了,看`code_main.py/local_ext_work`函数


在对llm做extra的处理的时候,发现一个问题
```
# 在使用one-api的时候,可能是部分的函数是没有进行补充的.所以报错
openai.BadRequestError: Error code: 400 - {'error': {'message': 'bad response status code 400 (request id: 2024061714253748439463076224324)', 'type': 'upstream_error', 'param': '400', 'code': 'bad_response_status_code'}}
```
使用openai的接口就可行.


主要是`ConversableAgent`类下面的`initiate_chat`中的参数`summary_method="reflection_with_llm"`
主要是这个
https://github.com/microsoft/autogen/blob/10b8fa548b5b8d6f8bfd15d392e7b7d8addb7cf5/autogen/agentchat/conversable_agent.py#L1102
类似的解决方法
- https://github.com/microsoft/autogen/issues/2474
- https://github.com/microsoft/autogen/pull/2527

修改的源码部分大概是: /home/hs/common/code/agnet_etc/autogen/autogen/agentchat/conversable_agent.py

又改又改找到了更接近的地方 `/home/hs/common/code/agnet_etc/autogen/autogen/oai/client.py`OpenAIClient 但是这个构造都是按照openai的体系来写的,所以报错.只支持这两 [OpenAI, AzureOpenAI]

