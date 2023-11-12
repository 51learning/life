
### 自动应答安装

安装

```bash
curl -sSL https://ap-southeast-3-hwcloudcli.obs.ap-southeast-3.myhuaweicloud.com/cli/latest/hcloud_install.sh -o ./hcloud_install.sh && bash ./hcloud_install.sh -y
```

初始化并运行

```bash
[root@testcce-26771 ~]# hcloud configure init
This will delete all configuration information. Continue? (y/N): y
Starting initialization. 'Secret Access Key' is anonymized. To obtain the parameter, see 'https://support.huaweicloud.com/intl/en-us/usermanual-hcli/hcli_09.html'.
Access Key ID [required]: SNVJ1HU1TQ0DB3RG1PHJ
Secret Access Key [required]: ****
Secret Access Key (again): ****
Region: la-north-2

********************************************************
*****                                              *****
*****          Initialization successful           *****
*****                                              *****
********************************************************

[root@testcce-26771 ~]# hcloud ECS NovaListServers
{
  "servers": [
    {
      "name": "testcce-26771",
      "links": [
        {
          "rel": "self",
          "href": "https://ecs.la-north-2.myhuaweicloud.com/v2.1/0e0e5cabfa80f2892f17c00390299e63/servers/8d0c9e65-1fc8-45ea-bdc7-9c42467f760b"
        },
        {
          "rel": "bookmark",
          "href": "https://ecs.la-north-2.myhuaweicloud.com/0e0e5cabfa80f2892f17c00390299e63/servers/8d0c9e65-1fc8-45ea-bdc7-9c42467f760b"
        }
      ],
      "id": "8d0c9e65-1fc8-45ea-bdc7-9c42467f760b"
    }
  ]
}
```

自定义 [Koocli 输出格式](https://support.huaweicloud.com/intl/zh-cn/usermanual-hcli/hcli_05_11.html) ，可以输出为表格格式，同进也可以通过json过滤输出某些想要的内容。


### 通过docker方式安装运行

编写dockerfile

```docker
FROM ubuntu:latest
RUN apt-get update -y && apt-get install curl -y
# Install KooCLI with one click.
RUN curl -sSL https://ap-southeast-3-hwcloudcli.obs.ap-southeast-3.myhuaweicloud.com/cli/latest/hcloud_install.sh -o ./hcloud_install.sh && bash ./hcloud_install.sh -y
WORKDIR hcloud
ENTRYPOINT ["/usr/local/bin/hcloud"]
```

build镜像并运行

```bash
docker build --no-cache -t hcloudcli .
docker run -it -d --name hcloudcli hcloudcli
docker run --rm -it hcloudcli ${command}
docker run --rm -it -v /root/.hcloud/:/root/.hcloud/ hcloudcli ${command}
alias hcloud='docker run --rm -it hcloudcli'
```


