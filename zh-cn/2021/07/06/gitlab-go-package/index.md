# 5分钟爽文：如何使用用gitlab作为go的依赖仓库


<!--more-->

## 一、前言
本来`goproxy.cn`是很好用的，但是公司目前只能使用`goproxy.baidu.com`，这个大部分用的时候都没有问题，但是发现偶尔有几个仓库死活拉不下来，但是换成`goproxy.cn`确实实在在能拉下来。所以想着为什么不搭建一个公司自己的`goproxy`呢？在解决这个问题的过程当中，却衍生出了另外一个想法：为什么不用公司自己的`gitlab`作为我们自己开发的包的仓库呢？本来以为这是个比较麻烦的工作，最后发现还挺简单的，该文就是用来记录整个过程。

## 二、内容
1. ### 准备gitlab环境
   `docker`运行`gitlab`镜像，注意必须给`docker``4G`以上内存，`cpu`也设置高点，最好`4c`，这样能操作流畅些，不然可能遇到`gitlab`页面访问不进去：
   ```shell
    docker run -d  -p 443:443 -p 80:80 -p 222:22 --name gitlab --restart always -v ~/config:/etc/gitlab  gitlab/gitlab-ce
   ```
   然后修改`gitlab`的配置文件，才能正常访问`gitlab`页面:
   - 先找到自己主机的`ip`地址，我这里就是`192.168.2.5`
   - 修改主机`host`，添加一条`DNS`记录：
      ```shell
      192.168.2.5 gitlab.com 
      ``` 
   - 修改`gitlab`配置文件
      ```shell
      vim ~/config/gitlab.rb
      ```
      然后修改
      ```shell
      external_url 'http://gitlab.com'
      ...
      gitlab_rails['gitlab_shell_ssh_port'] = 222
      ...
      gitlab_rails['gitlab_ssh_host'] = '192.168.2.5'
      ```
   - 最终重启容器  
     ```shell
     docker restart gitlab
     ```  
   - 浏览器访问: `http://gitlab.com`，如果机器性能不够的话，可能要等一会儿才行。当发现页面整个为空，可以换个浏览器访问看看，我就遇着`google`访问不行，换成`safari`可以。
   - 你可能进去的是这个页面，发现此时你并不知道密码，如果不是这个页面的话，就不用继续往下看`准备gitlab环境`了。下面要做的是重置`gitlab`用户名密码。
     
     - 首先进入容器： 
     ```shell
      docker exec -it gitlab /bin/bash
     ``` 
     - 然后重置`root`的密码，这里我设置为`admin123`
     ```shell
     # gitlab-rails console
       --------------------------------------------------------------------------------
        Ruby:         ruby 2.7.2p137 (2020-10-01 revision 5445e04352) [x86_64-linux]
        GitLab:       14.0.2 (bac4ee4a9e2) FOSS
        GitLab Shell: 13.19.0
        PostgreSQL:   12.6
        Loading production environment (Rails 6.1.3.2)
      --------------------------------------------------------------------------------
      irb(main):001:0>u= User.where(id: 1).first 
      => #<User id:1 @root>
      irb(main):001:0>u.password='admin123'
      => "admin123"
      irb(main):001:0>u.save!
      Enqueued ActionMailer::DeliveryJob (Job ID: 99118288-b58b-4d52-94c1-28979bcb63e8) to Sidekiq(mailers) with arguments: "DeviseMailer", "password_change", "deliver_now", gid://gitlab/User/1
      => true
      irb(main):001:0>quit
     ```
     - 此时管理员账号的密码已经重新设置了，登录即可。
2. ### 然后准备一个在`gitlab`创建一个库`gitlab.com/project_group/dependency_util`供演示如何在代码里面获取这个仓库。
   . 首先创建一个`group`:[`project_group`](http://gitlab.com/dashboard/groups):
   
   ![group](./new%20group.png)
   
   权限选择`public`方便测试。
   
   . 创建示例仓库[`dependency_util`](http://gitlab.com/project_group)：

   ![project](./new_project.png)

   权限选择`public`方便测试。
  
    . 然后`clone`该仓库，先初始化仓库，使用`mod`:
    ```shell
    git clone http://gitlab.com/project_group/dependency_util.git && cd dependency_util
    go mod init gitlab.com/project_group/dependency_utils
    ``` 
    根据`go`的`mod`的规则，包的地址必须是`gitlab.com/project_group/dependency_utils`,换成其他的都会在后面拉取包会出错。
    
    继续编辑`go`代码:
      - 拉取`error`仓库`go get github.com/pkg/error`
      - 然后新增文件`util.go`,编辑代码：
        ```go
        package dependency_utils

        import "github.com/pkg/errors"

        func IsString(i interface{}) error {
          if _, ok := i.(string); !ok {
            return errors.New("不是string！")
          } else {
            return nil
          }
        }
        ``` 
      - 提交代码并打上`tag`:`v0.0.1`。
      - `push`到`gitlab`去。
      - 此时示例的包已经准备好。
 3. ### 然后测试`gitlab`仓库托管的`go`的包。
    如法炮制创建一个项目`test-my-proxy`,然后用`mod`初始化项目
    ```shell
    go mod init project_group/test-my-proxy
    ```
    创建一个`main.go`文件,项目结构如下:
    ```
    .
    ├── go.mod
    ├── go.sum
    └── main.go
    ```
    编辑`main.go`文件如下:
    ```go
    package main

    import (
      "encoding/json"
      "fmt"
      "github.com/gin-gonic/gin"
      "io/ioutil"
      "net/http"
      "strings"
      "time"
    )
    import du "gitlab.com/project_group/dependency_util"

    func main() {

      r := gin.Default()
      r.POST("/validaInt", func(c *gin.Context) {
        var param struct {
          Value interface{} `json:"value"`
        }
        c.BindJSON(&param)

        err := du.IsString(param.Value)
        if err != nil {
          c.JSON(500, gin.H{
            "message": err.Error(),
          })
          return
        }

        c.JSON(200, gin.H{
          "message": "成功",
        })
        return
      })
      go r.Run()
      timer := time.NewTimer(2 * time.Second)
      select {
      case <-timer.C:
        var params = map[string]interface{}{
          "value": "1",
        }
        bytes, _ := json.Marshal(params)
        response, err := http.Post("http://localhost:8080/validaInt", "application/json", strings.NewReader(string(bytes)))
        if err != nil {
          break
        }
        if err != nil {
          fmt.Println(err)
          break
        }
        bytes, err = ioutil.ReadAll(response.Body)
        if err != nil {
          fmt.Println(err)
          break
        }
        fmt.Println(string(bytes))
        break
      }
    }
    ```
    此段代码的逻辑就是，启动一个`gin`的`server`服务，等2秒后`server`服务监听了`8080`端口之后，发送请求到`server`,注意`validaInt`这个接口就使用了刚刚我们在`gitlab`上托管的第三方包`gitlab.com/project_group/dependency_util`。

    此时还没有完成，需要做一下几个操作:
    - 设置`https://goproxy.cn`是为了在天朝拉取`github.com/gin-gonic/gin`，设置`direct`是为了拉取我们托管在自己`gitlab`的包。
      ```shell
      go env -w GOPROXY=https://goproxy.cn,direct
      ```
    - 关闭`GOSUMDB`，不让校验我们`gitlab`的哈希值。
      ```shell
      go env -w GOSUMDB=off
      ``` 
    - 最后拉取该项目依赖的包
      ```shell
      go mod tidy
      ``` 
    - 最后执行代码测试是否成功:
      ```shell
      go run main.go
      ``` 
      结果如下:
      ```shell
      [GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

      [GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
      - using env:   export GIN_MODE=release
      - using code:  gin.SetMode(gin.ReleaseMode)

      [GIN-debug] POST   /validaInt                --> main.main.func1 (3 handlers)
      [GIN-debug] Environment variable PORT is undefined. Using port :8080 by default
      [GIN-debug] Listening and serving HTTP on :8080
      [GIN] 2021/07/07 - 22:21:16 | 200 |      306.48µs |             ::1 | POST     "/validaInt"
      {"message":"成功"}
      ```
     - 此时查看该项目的`go.mod`:
        ```shell
        module project_group/test-my-proxy

        go 1.16

        require (
          github.com/gin-gonic/gin v1.7.2
          gitlab.com/project_group/dependency_util v0.0.1
        )
        ```
        已经把包`dependency_util@v0.0.1`同步到`go.mod`中了。
## 三、总结         
上面详细列出了用`gitlab`作为公司项目的`包`托管，这个是非常有意义的。而且很多公司内网是访问不了外网，这个时候可以像搭建`maven`和`npm`代理一样，可以搭建`goproxy`代理，推荐用`github`项目`https://github.com/goproxy/goproxy.cn`。
- 欢迎关注我的微信公众号:
![云原生玩码部落](https://img-blog.csdnimg.cn/20201213215616541.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA5MjczNDA=,size_16,color_FFFFFF,t_70)
