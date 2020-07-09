## 镜像源总汇

| 机构   | 官方网址                                          |
| ------ | ------------------------------------------------- |
| 清华   | https://mirrors.tuna.tsinghua.edu.cn/help/centos/ |
| 中科大 | http://mirrors.ustc.edu.cn/help/centos.html       |
| 阿里云 | https://developer.aliyun.com/mirror/              |
| 163    | http://mirrors.163.com/                           |

## RHEL/CentOS社区镜像源

| 机构   | 说明                                                         | 官网网址                                  |
| :----- | ------------------------------------------------------------ | ----------------------------------------- |
| EPEL   | 为 RHEL 及衍生发行版如 CentOS、Scientific Linux 等提供高质量软件包的项目 | https://fedoraproject.org/wiki/EPEL/zh-cn |
| ELRepo | RHEL/CentOS内核镜像社区                                      | http://elrepo.org/tiki/tiki-index.php     |

## CentOS7.x 升级内核

```shell
$ yum list kernel #查看小版本列表
$ yum update kernel -y # 升级小版本
#===
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org # 载入公钥
$ rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm # 升级安装ELRepo
载入elrepo-kernel元数据：yum --disablerepo=\* --enablerepo=elrepo-kernel repolist
$ yum --disablerepo=\* --enablerepo=elrepo-kernel list kernel* # 查看可用的rpm包
$ yum --disablerepo=\* --enablerepo=elrepo-kernel install  kernel-ml.x86_64  -y # 安装最新版本的kernel，li稳定版，ml最新版
$ yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64  -y # 删除旧版本工具包
$ yum --disablerepo=\* --enablerepo=elrepo-kernel install kernel-ml-tools.x86_64  -y  # 安装新版本工具包
$ awk -F \' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg # 查看内核插入顺序
$ grub2-editenv list # 查看当前实际启动顺序
$ grub2-set-default 'CentOS Linux (4.20.12-1.el7.elrepo.x86_64) 7 (Core)' # 设置默认启动
$ reboot
```

