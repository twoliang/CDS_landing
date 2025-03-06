# gin框架图文入门

## 服务器实例

```go
package main
 
import "github.com/gin-gonic/gin"
 
func main() {
    r := gin.Default()
    r.GET("/", func(c *gin.Context) {
       c.JSON(200, gin.H{
          "message": "Hello, Gin!",
       })//封装
    })
    r.Run(":8080")
}
```

## 路由定义

### 不同方法的路由

get请求：

```go
r.GET("/users", func(c *gin.Context) {
   // 处理 GET 请求逻辑
})
```

post请求：

```go
r.POST("/users", func(c *gin.Context) {
   // 处理 POST 请求逻辑
})
```

put请求：

```go
r.PUT("/users/:id", func(c *gin.Context) {
   // 处理 PUT 请求逻辑
})
```

delete请求：

```go
r.DELETE("/users/:id", func(c *gin.Context) {
   // 处理 DELETE 请求逻辑
})
```

> ### 实际应用场景
>
> 1. **GET**：
>    - 获取用户列表：`GET /users`
>    - 获取单个用户信息：`GET /users/123`
> 2. **POST**：
>    - 创建新用户：`POST /users`
>    - 提交表单数据：`POST /form`
> 3. **PUT**：
>    - 更新用户信息：`PUT /users/123`
>    - 替换文章内容：`PUT /articles/456`
> 4. **DELETE**：
>    - 删除用户：`DELETE /users/123`
>    - 删除文章：`DELETE /articles/456`
>
> ------
>
> ### 总结
>
> - **GET** 用于获取数据，是安全和幂等的。
> - **POST** 用于创建数据，是非安全和非幂等的。
> - **PUT** 用于更新数据（完整替换），是非安全但幂等的。
> - **DELETE** 用于删除数据，是非安全但幂等的。

### 路由参数

```go
r.GET("/users/:id", func(c *gin.Context) {
   id := c.Param("id")
   // 使用参数 id 进行处理
})
```

**`c.Param("id")`**

- `c.Param("id")` 是从上下文中获取路由参数的方法。
- 在这个例子中，`id` 是动态匹配的路由参数。`c.Param("id")` 会从请求的 URL 中提取出 `:id` 部分的值。
- 例如，如果请求的 URL 是 `/users/456`，那么 `id` 的值就是 `"456"`。

### 查询参数

```go
r.GET("/search", func(c *gin.Context) {
   keyword := c.Query("keyword")
   // 使用查询参数 keyword 进行处理
})
```

**`c.Query("keyword")`**

- `c.Query("keyword")` 是从上下文中获取查询参数（URL 中的 `?key=value` 部分）的方法。
- 在这个例子中，`keyword` 是查询参数的名称。例如，如果请求的 URL 是 `http://localhost:8080/search?keyword=golang`，那么 `c.Query("keyword")` 的返回值就是 `"golang"`。
- 如果查询参数 `keyword` 不存在，`c.Query("keyword")` 会返回一个空字符串 `""`。

## 中间件

### 全局中间件

比如记录日志请求：

```go
r.Use(func(c *gin.Context) {
   // 在请求处理前记录请求开始时间
   start := time.Now()
   c.Next()
   // 在请求处理后记录请求耗时并输出日志
   latency := time.Since(start)
   log.Printf("请求路径：%s，耗时：%s", c.Request.URL.Path, latency)
})
```

- `c.Next()` 是中间件的核心。它会暂停当前中间件的执行，继续调用下一个中间件或最终的路由处理器。
- 如果没有调用 `c.Next()`，请求处理将不会继续，后续的中间件和路由处理器都不会执行。

> 中间件的执行流程是基于 `c.Next()` 的调用。Gin 的中间件机制类似于一个洋葱模型，中间件会按照注册的顺序依次执行，直到遇到 `c.Next()`，然后按逆序返回。

### 路由组中间件

```go
api := r.Group("/api", func(c *gin.Context) {
   // 特定于/api 路由组的中间件逻辑
   c.Next()
})
api.GET("/users", func(c *gin.Context) {
   // /api/users 的处理逻辑
})
```

**`r.Group` 方法**

- `r` 是一个 `*gin.Engine` 类型的对象，表示 Gin 的 Web 服务器实例。
- `r.Group` 是 Gin 提供的一个方法，用于创建一个路由组。路由组允许你为一组路由共享相同的前缀和中间件。
- `r.Group` 的第一个参数是路由组的前缀，例如 `"/api"`。
- 第二个参数是一个中间件函数，类型为 `func(c *gin.Context)`，它会在路由组中的每个请求处理之前执行。

2. **中间件逻辑**

```go
func(c *gin.Context) {
    // 特定于/api 路由组的中间件逻辑
    c.Next()
}
```

- 这是一个匿名函数，作为路由组的中间件。
- 在这个中间件中，你可以执行一些特定于 `/api` 路由组的逻辑，例如：
  - 验证用户身份（如 API 密钥验证）。
  - 记录请求日志。
  - 设置特定的请求头。
- 调用 `c.Next()` 是必须的，它会暂停当前中间件的执行，继续调用下一个中间件或路由处理器。

3. **路由组的定义**

```go
api := r.Group("/api", func(c *gin.Context) {
    // 特定于/api 路由组的中间件逻辑
    c.Next()
})
```

- `api` 是一个 `*gin.RouterGroup` 类型的对象，表示 `/api` 路由组。
- 所有在 `api` 路由组中定义的路由都会自动继承 `/api` 前缀，并且会执行指定的中间件。

4. **在路由组中定义路由**

```go
api.GET("/users", func(c *gin.Context) {
    // /api/users 的处理逻辑
})
```

- 这里定义了一个 GET 请求的路由 `/users`，但它实际上是 `/api/users`，因为 `/api` 是路由组的前缀。
- 这个路由的处理逻辑会处理所有访问 `http://localhost:8080/api/users` 的 GET 请求。

### 单个路由中间件

```go
// 定义中间件
	authMiddleware := func(c *gin.Context) {//authMiddleware 是一个中间件函数
		auth := c.GetHeader("Authorization")
		if auth != "secret-key" {
			c.JSON(http.StatusForbidden, gin.H{"error": "无权访问"})
			c.Abort()
			return
		}
		c.Next()
	}

	// 为 /admin 路由添加中间件
	r.GET("/admin", authMiddleware, func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"message": "欢迎访问管理员页面"})
	})
```

## 请求处理

### 获取请求体数据

```go
type User struct {
   Name string `json:"name"`
   Age  int    `json:"age"`
}
r.POST("/users", func(c *gin.Context) {
   var user User
   if err := c.ShouldBindJSON(&user); err!= nil {
      c.JSON(400, gin.H{"error": err.Error()})
      return
   }
   // 处理用户数据
})
```

> 中间件（Middleware）是一种在请求处理过程中插入的函数，它可以在请求到达最终的路由处理器之前或之后执行一些逻辑。中间件通常用于执行一些通用的任务，例如：
>
> - **身份验证**：检查用户是否有权限访问某个资源。
> - **日志记录**：记录请求的详细信息。
> - **请求修改**：修改请求头或请求体。
> - **响应修改**：修改响应头或响应体。
> - **错误处理**：捕获和处理错误。

 c.ShouldBindJSON(&user)：

> - 声明一个 `User` 类型的变量 `user`，用于存储请求体中的 JSON 数据。
> - 使用 `c.ShouldBindJSON(&user)` 将请求体中的 JSON 数据绑定到 `user` 变量。
>   - 如果绑定成功，`user` 变量将包含 JSON 数据中的字段值。
>   - 如果绑定失败（例如，请求体不是有效的 JSON 格式），`ShouldBindJSON` 会返回一个错误。
> - 如果绑定失败，返回一个 400 状态码（Bad Request），并返回错误信息。

### 响应数据

```go
//返回json响应
r.GET("/users", func(c *gin.Context) {
   users := []User{
      {Name: "Alice", Age: 25},
      {Name: "Bob", Age: 30},
   }
   c.JSON(200, users)
})
```

> - 使用 `c.JSON` 方法将 `users` 切片序列化为 JSON 格式，并返回给客户端。
> - `200` 是 HTTP 状态码，表示请求成功。
> - `users` 是要返回的数据，Gin 会自动将其序列化为 JSON 格式。

```go
//返回html响应
r.GET("/", func(c *gin.Context) {
   c.HTML(200, "index.tmpl", gin.H{
      "title": "Gin 示例",
   })
})
```

> **`c.HTML` 方法**：用于渲染 HTML 模板并返回 HTTP 响应。
>
> - **第一个参数**：HTTP 状态码，`200` 表示请求成功。
> - **第二个参数**：模板文件的名称，`"index.tmpl"` 表示模板文件的名称为 `index.tmpl`。
> - **第三个参数**：传递给模板的数据，`gin.H` 是一个 `map[string]interface{}` 类型，用于存储模板中需要使用的数据。

## 错误处理

### 自定义错误响应

```go
type ErrorResponse struct {
   Code    int    `json:"code"`
   Message string `json:"message"`
}
r.GET("/error", func(c *gin.Context) {
   c.JSON(400, ErrorResponse{Code: 400, Message: "错误请求"})
})
```

### 全局错误处理

```go
//gin内置错误处理机制
r.Use(func(c *gin.Context) {
   defer func() {
      if err := recover(); err!= nil {
         c.JSON(500, ErrorResponse{Code: 500, Message: "内部服务器错误"})
      }
   }()
   c.Next()
})
```

> - **`r.Use` 方法**：用于注册全局中间件，这些中间件会对所有路由生效。
> - **用途**：
>   - 日志记录
>   - 身份验证和授权
>   - 错误处理
>   - 跨域请求支持
>   - 修改请求和响应
>   - 性能监控
> - **中间件的执行顺序**：中间件会按照注册的顺序依次执行，直到遇到 `c.Next()`，然后按逆序返回。

## 部署

### 独立部署

```go
 go build
./your_app_name
```

### 容器化部署

