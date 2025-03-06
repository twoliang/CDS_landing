# web项目补充

## logic

这一层一般不直接写函数，而是写方法，具体如下：

```go
package user

import (
	"fmt"
	"project/internal/dao"
	"project/internal/model/entity"
	"project/internal/pkt"
	"project/internal/service"
)

type sUser struct {
}

func init() {
	service.InitUser(New())
}

func New() *sUser {
	return &sUser{}
}

func (s *sUser) Register(username, password, email string) error {
	user := entity.User{Username: username, Password: password, Email: email}
	return dao.CreateUser(user)
}

func (s *sUser) Login(username, password string) (string, error) {
	var (
		user *entity.User
		err  error
	)

	user, err = dao.GetUserByUsername(username)
	if err != nil {
		return "", fmt.Errorf("用户名错误: %v", err)
	}
	if user.Password != password {
		return "", fmt.Errorf("密码错误")
	}
	return pkt.GenerateToken(user)
}

```

同时，需要初始化一下service层的数据，并在service层抽象接口，填充这一层的方法，后续controller层调用是调用service层的方法。

## service

这一层主要抽象接口，具体代码如下：

```go
package service

type IUser interface {
	Register(username, password, email string) error
	Login(username, password string) (string, error)
}

var (
	iUser IUser
)

func User() IUser {
	return iUser
}

func InitUser(i IUser) {
	iUser = i
}

```

这里定义全局变量，用于初始化并返回数据，最后抽象接口，里面存放在logic设置好的两个方法。

## controller

这一层直接调用service层暴露的接口中的方法，实现抽象：

```go
package user

import (
	"github.com/gin-gonic/gin"
	"net/http"
	"project/internal/model/entity"
	"project/internal/service"
)

func RegisterHandler(c *gin.Context) {
	var (
		req entity.User
		err error
	)
	if err = c.ShouldBind(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	} else if err = service.User().Register(req.Username, req.Password, req.Email); err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"code": 0, "message": "注册成功"})

}

func LoginHandler(c *gin.Context) {
	var (
		req   entity.User
		err   error
		token string
	)
	if err = c.ShouldBind(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	token, err = service.User().Login(req.Username, req.Password)
	if err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "无效token"})
		return
	}
	c.JSON(http.StatusOK, gin.H{"token": token})
}

```

> 注意调用注册和登录方法的位置。

## dao

这一层主要做数据库的相关操作，自己是做了数据库的初始化，并传入到全局变量DB中，但实际可以先传入到私有变量db中，后续额外封装一个公共的DB方法，返回db，具体代码如下：

```go
package dao

import (
	"context"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"project/hack/config"
)

var db *gorm.DB

func InitDB() (errs error) {
	var (
		dsn string
		err error
	)
	dsn = config.Config.Database.DSN
	db, err = gorm.Open(mysql.Open(dsn), &gorm.Config{})

	if err != nil {
		return
	}

	return
}

func DB(ctx context.Context) *gorm.DB {//额外封装
	return db.WithContext(ctx)
}

```

## 进一步的改进

对于结构体，可以单独见一个文件来存放，后续使用直接调用，解耦各个文件，让各个文件的作用专一。

中间件的逻辑，一般也是单独新建一个文件夹，后新建文件处理。

## 上下文的使用

### **1. 传入上下文的主要作用**
上下文（`context.Context`）在数据库操作中的主要作用包括以下几点：

#### **(1) 超时控制**
- 数据库操作（如查询、插入、更新等）可能会因为网络延迟、数据库负载高等原因耗时过长。通过上下文的超时机制，可以为数据库操作设置一个最大执行时间。
- 如果操作在超时时间内没有完成，GORM 会自动取消操作，避免程序长时间等待。

**示例：**
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

db := dao.DB(ctx)
if err := db.First(&user, "id = ?", 1).Error; err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        log.Println("Query timed out")
    } else {
        log.Println("Query failed:", err)
    }
}
```
- 在这个例子中，`context.WithTimeout` 为数据库查询设置了 5 秒的超时时间。如果查询超过 5 秒仍未完成，GORM 会收到超时信号并取消查询。

#### **(2) 取消操作**
- 在某些场景下，可能需要手动取消正在进行的数据库操作。例如，当用户取消了一个 HTTP 请求时，与该请求相关的数据库操作也应该被取消，以避免资源浪费。
- 上下文的取消信号可以实现这一功能。

**示例：**
```go
ctx, cancel := context.WithCancel(context.Background())
go func() {
    select {
    case <-ctx.Done():
        log.Println("Database operation was canceled")
    }
}()

db := dao.DB(ctx)
if err := db.First(&user, "id = ?", 1).Error; err != nil {
    log.Println("Query failed:", err)
}

// 取消操作
cancel()
```
- 在这个例子中，调用 `cancel()` 会触发上下文的取消信号，GORM 会收到信号并取消正在进行的数据库操作。

#### **(3) 传递请求范围的值**
- 上下文还可以用于在请求链中传递请求范围的值（如用户身份信息、请求 ID 等）。这些值可以在数据库操作中使用，例如记录日志或进行权限检查。

**示例：**
```go
ctx := context.WithValue(context.Background(), "request_id", "12345")
db := dao.DB(ctx)
if err := db.First(&user, "id = ?", 1).Error; err != nil {
    log.Println("Query failed:", err)
}

// 在日志中记录请求 ID
log.Printf("Request ID: %v, User: %+v", ctx.Value("request_id"), user)
```
- 在这个例子中，通过 `context.WithValue` 将请求 ID 传递到数据库操作中，可以在日志中记录请求 ID，方便问题追踪。

> [!IMPORTANT]
>
> 这里是重点，用于日志操作。

### context.BackGround

- **`context.Background()` 的作用**：创建一个空的、永远不会被取消的根上下文，作为其他上下文的起点。
- **使用场景**：用于 HTTP 请求、后台任务、初始化等场景，作为上下文链的起点。
- **重要性**：它是上下文机制的基础，所有其他上下文都是通过它派生出来的。

 