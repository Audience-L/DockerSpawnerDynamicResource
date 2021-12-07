# DockerSpawnerDynamicResource

jupyter dockerspawner创建容器动态设置资源，可读数据库，并支持gpu
# 如果需要gpu调度,docker版本建议>19

# install python_on_whales
pip install python_on_whales

# 添加依赖包
from python_on_whales.docker_client import DockerClient

# 重写client,添加python_on_whales客户端
@property
def client(self):
    """single global client instance"""
    cls = self.__class__
    str = 'python_on_whales'
    if cls._client is None:
        kwargs = {"version": "auto"}
        if self.tls_config:
            kwargs["tls"] = docker.tls.TLSConfig(**self.tls_config)
        kwargs.update(kwargs_from_env())
        kwargs.update(self.client_kwargs)
        client = docker.APIClient(**kwargs)
        cls._client = client
    elif str in cls._client.__str__():
        kwargs = {"version": "auto"}
        if self.tls_config:
            kwargs["tls"] = docker.tls.TLSConfig(**self.tls_config)
        kwargs.update(kwargs_from_env())
        kwargs.update(self.client_kwargs)
        client = docker.APIClient(**kwargs)
        cls._client = client

    return cls._client

@property
def docker_gpu_client(self):
    cls = self.__class__
    cls._client = DockerClient()

    return cls._client

#support_gpu
def _gpu_docker(self, method, *args, **kwargs):
    """wrapper for calling docker methods

    to be passed to ThreadPoolExecutor
    """
    m = getattr(self.docker_gpu_client, method)
    return m(*args, **kwargs)

def gpu_docker(self, method, *args, **kwargs):
    """Call a docker method in a background thread

    returns a Future
    """
    return self.executor.submit(self._gpu_docker, method, *args, **kwargs)

# 在create_object函数中添加以下代码
envs = create_kwargs["host_config"]["Binds"]
docker_volume = [tuple(env.split(':', 2)) for env in envs]
create_kwargs = dict(
    image='jupyter/base-notebook:hub-1.5.0',
    envs=create_kwargs["environment"],
    name=create_kwargs["name"],
    command=create_kwargs["command"],
    volumes=docker_volume,
    networks=create_kwargs["host_config"]["NetworkMode"],
    cpus=1,
    memory="1G",
    #support gpu 
    #gpus=f"\"device=1\"",
    #use docker random ports
    publish_all=True
)

# 删除原始调度函数,替换为下面的函数
res = self.gpu_docker("create", **create_kwargs)
obj = {self.object_id_key: str(res.result())}