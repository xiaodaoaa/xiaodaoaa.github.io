在CentOS 7上安装最新版本的Docker，你可以按照以下步骤操作：

1. **卸载旧版本**（如果有）
   如果之前安装过旧版本的Docker，请先将其卸载：
   ```bash
   sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
   ```

2. **设置存储库**
   安装一些必要的包，以便`yum`可以使用存储库通过HTTPS来安装Docker：
   ```bash
   sudo yum install -y yum-utils
   ```
   
   使用官方存储库：
   ```bash
   sudo yum-config-manager \
       --add-repo \
       https://download.docker.com/linux/centos/docker-ce.repo
   ```
   
   或者，为了更快的下载速度，可以使用阿里云提供的镜像源：
   ```bash
   sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   ```

3. **安装Docker引擎**
   安装最新版本的Docker CE和containerd：
   ```bash
   sudo yum install docker-ce docker-ce-cli containerd.io
   ```

4. **启动Docker**
   安装完成后，启动Docker服务，并设置开机自启：
   ```bash
   sudo systemctl start docker
   sudo systemctl enable docker
   ```

5. **验证安装**
   检查Docker是否正确安装并运行：
   ```bash
   docker --version
   ```

6. **配置非root用户使用Docker**（可选）
   默认情况下，只有root用户和docker组内的用户才能执行docker命令。如果你希望普通用户也能运行docker命令，需要将该用户添加到docker组中：
   ```bash
   sudo usermod -aG docker your-user
   ```
   更改组成员资格后，你需要注销并重新登录以使更改生效。

以上步骤应该能让你顺利地在CentOS 7上安装最新版的Docker。如果在安装过程中遇到任何问题，请确保你的系统是最新的，并且检查是否有任何错误信息提供线索。