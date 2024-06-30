这是一个GitHub Actions工作流配置文件，旨在自动化构建和推送Docker镜像到阿里云（Aliyun）容器镜像库。以下是代码逻辑的逐步解释：

### 工作流触发器
```yaml
on:
  workflow_dispatch:
  push:
    branches: [ main ]
```
- 该工作流可以手动触发（`workflow_dispatch`），或在推送到`main`分支时触发。

### 环境变量
```yaml
env:
  ALIYUN_REGISTRY: "${{ secrets.ALIYUN_REGISTRY }}"
  ALIYUN_NAME_SPACE: "${{ secrets.ALIYUN_NAME_SPACE }}"
  ALIYUN_REGISTRY_USER: "${{ secrets.ALIYUN_REGISTRY_USER }}"
  ALIYUN_REGISTRY_PASSWORD: "${{ secrets.ALIYUN_REGISTRY_PASSWORD }}"
```
- 使用GitHub Secrets来设置阿里云镜像库的凭证。

### 作业：构建
```yaml
jobs:
  build:
    name: Pull
    runs-on: ubuntu-latest
```
- 这个作业命名为`Pull`，运行在最新的Ubuntu环境下。

#### 步骤：显示释放磁盘空间前的情况
```yaml
    steps:
    - name: Before freeing up disk space
      run: |
        echo "Before freeing up disk space"
        echo "=============================================================================="
        df -hT
        echo "=============================================================================="
```
- 显示释放磁盘空间前的磁盘使用情况。

#### 步骤：最大化构建空间
```yaml
    - name: Maximize build space
      uses: easimon/maximize-build-space@master
      with:
        root-reserve-mb: 2048
        swap-size-mb: 128
        remove-dotnet: 'true'
        remove-haskell: 'true'
        # 如果空间还是不够用，可以把以下开启，清理出更多空间
        # remove-android: 'true'
        # remove-codeql: 'true'
        build-mount-path: '/var/lib/docker/'
```
- 通过删除不必要的软件包和调整系统设置来最大化可用磁盘空间。

#### 步骤：重启Docker
```yaml
    - name: Restart docker
      run: sudo service docker restart
```
- 重启Docker服务以应用新的磁盘空间配置。

#### 步骤：显示释放磁盘空间后的情况
```yaml
    - name: Free up disk space complete
      run: |
        echo "Free up disk space complete"
        echo "=============================================================================="
        df -hT
        echo "=============================================================================="
```
- 显示释放磁盘空间后的磁盘使用情况。

#### 步骤：检出代码
```yaml
    - name: Checkout Code
      uses: actions/checkout@v4
```
- 检出代码仓库中的代码。

#### 步骤：设置Docker Buildx
```yaml
    - name: Docker Setup Buildx
      uses: docker/setup-buildx-action@v3
```
- 设置Docker Buildx以支持多平台构建。

#### 步骤：构建并推送镜像到阿里云
```yaml
    - name: Build and push image Aliyun
      run: |
        docker login -u $ALIYUN_REGISTRY_USER -p $ALIYUN_REGISTRY_PASSWORD $ALIYUN_REGISTRY
        # 数据预处理,判断镜像是否重名
        declare -A duplicate_images
        declare -A temp_map
        while IFS= read -r line || [ -n "$line" ]; do
            # 忽略空行与注释
            [[ -z "$line" ]] && continue
            if echo "$line" | grep -q '^\s*#'; then
                continue
            fi
            
            # 获取镜像的完整名称，例如kasmweb/nginx:1.25.3（命名空间/镜像名:版本号）
            image=$(echo "$line" | awk '{print $NF}')
            # 将@sha256:等字符删除
            image="${image%%@*}"
            echo "image $image"
            # 获取镜像名:版本号  例如nginx:1.25.3
            image_name_tag=$(echo "$image" | awk -F'/' '{print $NF}')
            echo "image_name_tag $image_name_tag"
            # 获取命名空间 例如kasmweb,  这里有种特殊情况 docker.io/nginx，把docker.io当成命名空间，也OK
            name_space=$(echo "$image" | awk -F'/' '{if (NF==3) print $2; else if (NF==2) print $1; else print ""}')
            echo "name_space: $name_space"
            # 这里不要是空值影响判断
            name_space="${name_space}_"
            # 获取镜像名例如nginx
            image_name=$(echo "$image_name_tag" | awk -F':' '{print $1}')
            echo "image_name: $image_name"
            
            # 如果镜像存在于数组中，则添加temp_map
            if [[ -n "${temp_map[$image_name]}" ]]; then
                 # 如果temp_map已经存在镜像名，判断是不是同一命名空间
                 if [[ "${temp_map[$image_name]}" != $name_space  ]]; then
                    echo "duplicate image name: $image_name"
                    duplicate_images[$image_name]="true"
                 fi
            else
                # 存镜像的命名空间
                temp_map[$image_name]=$name_space
            fi       
        done < images.txt
        
        
        while IFS= read -r line || [ -n "$line" ]; do
            # 忽略空行与注释
            [[ -z "$line" ]] && continue
            if echo "$line" | grep -q '^\s*#'; then
                continue
            fi
        
            echo "docker pull $line"
            docker pull $line
            platform=$(echo "$line" | awk -F'--platform[ =]' '{if (NF>1) print $2}' | awk '{print $1}')
            echo "platform is $platform"
            # 如果存在架构信息 将架构信息拼到镜像名称前面
            if [ -z "$platform" ]; then
                platform_prefix=""
            else
                platform_prefix="${platform//\//_}_"
            fi
            echo "platform_prefix is $platform_prefix"
            # 获取镜像的完整名称，例如kasmweb/nginx:1.25.3（命名空间/镜像名:版本号）
            image=$(echo "$line" | awk '{print $NF}')

            # 获取 镜像名:版本号  例如nginx:1.25.3
            image_name_tag=$(echo "$image" | awk -F'/' '{print $NF}')
            # 获取命名空间 例如kasmweb  这里有种特殊情况 docker.io/nginx，把docker.io当成命名空间，也OK
            name_space=$(echo "$image" | awk -F'/' '{if (NF==3) print $2; else if (NF==2) print $1; else print ""}')
            # 获取镜像名例  例如nginx
            image_name=$(echo "$image_name_tag" | awk -F':' '{print $1}')
        
            name_space_prefix=""
            # 如果镜像名重名
            if [[ -n "${duplicate_images[$image_name]}" ]]; then
               #如果命名空间非空，将命名空间加到前缀
               if [[ -n "${name_space}" ]]; then
                  name_space_prefix="${name_space}_"
               fi
            fi
            
            # 将@sha256:等字符删除
            image_name_tag="${image_name_tag%%@*}"
            new_image="$ALIYUN_REGISTRY/$ALIYUN_NAME_SPACE/$platform_prefix$name_space_prefix$image_name_tag"
            echo "docker tag $image $new_image"
            docker tag $image $new_image
            echo "docker push $new_image"
            docker push $new_image
            echo "开始清理磁盘空间"
            echo "=============================================================================="
            df -hT
            echo "=============================================================================="
            docker rmi $image
            docker rmi $new_image
            echo "磁盘空间清理完毕"
            echo "=============================================================================="
            df -hT
            echo "=============================================================================="     
            
        done < images.txt
```
- 登录阿里云镜像库。
- 预处理数据，判断镜像是否重名。
- 逐行读取`images.txt`文件，拉取镜像并推送到阿里云。
- 清理磁盘空间，删除已拉取和推送的镜像。

### 主要逻辑
- 检查和释放磁盘空间。
- 设置Docker Buildx。
- 从`images.txt`文件读取镜像信息，拉取镜像并推送到阿里云。
- 处理重复的镜像名称。
- 在推送后清理磁盘空间。
