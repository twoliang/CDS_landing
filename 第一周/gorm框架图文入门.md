# gorm框架图文入门

## 定义模型

```go
type User struct {
	ID        uint   `gorm:"primaryKey"`
	Name      string `gorm:"size:100;not null"`
	Age       int    `gorm:"not null"`
	Email     string `gorm:"unique;not null"`
	CreatedAt time.Time
}
```

- `CreatedAt` 是结构体的一个字段，类型为 `time.Time`，表示用户的创建时间。
- 这个字段没有 GORM 标签，但 GORM 默认会将其识别为时间戳字段，并自动管理其值（通常在创建记录时自动设置为当前时间）。

## 初始化数据库连接

```go
dsn := "username:password@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
if err != nil {
    panic("failed to connect database")
}
```

## 自动迁移

```go
db.AutoMigrate(&User{})
```

> 这里传入指针而不是结构体的原因如下
>
> - **设计灵活性**：使用指针作为参数，使得 GORM 可以动态地处理结构体的变化，并且兼容各种结构体类型。
>
> **反射机制的需要**
>
> Go 语言的反射机制允许程序在运行时检查变量的类型和值，并对其进行操作。反射的一个重要特性是它可以通过指针来修改变量的值。在 GORM 中，`AutoMigrate` 方法需要通过反射来获取结构体的字段信息，并根据这些信息生成数据库表的定义。
>
> - **结构体本身（值类型）**：如果直接传入结构体实例（如 `User{}`），反射只能获取到该实例的字段值，而无法获取到结构体的类型信息（如字段名、字段标签等）。
> - **结构体指针**：通过传入结构体的指针（如 `&User{}`），GORM 可以通过反射获取到结构体的类型信息（如字段名、字段标签等），从而正确地生成数据库表的定义。
> - Gorm 需要直接修改 users 切片的内容（例如，分配内存、填充数据等）。如果不使用 &，Gorm 只能操作 users 的副本，而不是原始切片。也就会导致查询结果不会填充到原始 users 切片中。

## CRUD

创建：

```go
user := User{Name: "Alice", Age: 30, Email: "alice@example.com"}
db.Create(&user)
```

查询：

> ### **`db.Where("name = ?", "Alice").First(&user)`**
>
> 这行代码的作用是根据条件查询数据库中的记录。具体解释如下：
>
> - **`db.Where` 方法**：
>   - `db.Where` 是 GORM 提供的一个方法，用于添加查询条件。
>   - 它接受一个查询条件字符串和相应的参数，生成 SQL 查询语句中的 `WHERE` 子句。
> - **`"name = ?"` 参数**：
>   - 这是一个查询条件字符串，`?` 是一个占位符，表示查询条件的值将通过参数传入。
> - **`"Alice"` 参数**：
>   - 这是查询条件的值，表示查询条件为 `name = 'Alice'`。
> - **`db.First` 方法**：
>   - `db.First` 用于查询满足条件的第一条记录，并将结果赋值到传入的指针变量中（这里是 `&user`）。

> - **`db.First` 方法**：
>   - `db.First` 是 GORM 提供的一个方法，用于查询数据库中满足条件的第一条记录。
>   - 如果查询成功，查询结果会被赋值到传入的指针变量中（这里是 `&user`）。
>   - 如果查询失败（例如没有找到记录），GORM 会返回一个错误。
> - **`&user` 参数**：
>   - `&user` 是一个指向 `user` 的指针。GORM 通过指针将查询结果直接赋值给 `user` 变量。
> - **`1` 参数**：
>   - `1` 是主键的值。这里假设 `User` 表的主键字段是 `ID`，GORM 会根据 `ID = 1` 的条件查询数据库。

```go
var user User
db.First(&user, 1) // 根据主键查询
db.Where("name = ?", "Alice").First(&user)
```

更新：

```go
db.Model(&user).Update("Age", 31)
// 或者批量更新
db.Model(&user).Updates(User{Age: 32, Email: "newalice@example.com"})
```

删除：

```go
db.Delete(&user, 1)
```

> 自动迁移：
>
> 自动建表、更新表结构。
>
> 自动迁移的局限性：
>
> - **不会删除字段**：如果从结构体中移除了某个字段，自动迁移不会从表中删除对应的列。
> - **不处理复杂变更**：对于一些复杂的表结构变更，如修改列的数据类型、重命名列等，自动迁移可能无法正确处理，需要手动编写 SQL 语句来完成。

> 在 GORM（Go 语言的 ORM 库）中，预加载（Preloading）是一种非常有用的技术，主要用于解决数据库查询中的 N + 1 查询问题，下面详细介绍其作用：
>
> 1. **解决 N + 1 查询问题**
>
> 在进行关联查询时，如果不使用预加载，通常会先执行一个查询获取主表的所有记录，然后针对每条主表记录，再分别执行一个查询去获取关联表的相关记录。这样，如果主表有 N 条记录，就会执行 1 个主表查询和 N 个关联表查询，总共 N + 1 次查询，这会导致性能问题，尤其是当主表记录数量较多时。
>
> 预加载通过一次查询将主表和关联表的数据一起获取，避免了多次查询数据库，从而显著提高查询性能。
>
> 2. **提高代码的可读性和可维护性**
>
> 使用预加载可以将关联数据的查询逻辑集中在一处，使代码更加简洁和易于理解。开发者不需要在多个地方分别处理主表和关联表的查询，只需要在主查询中使用 `Preload` 方法即可一次性获取所需的所有数据。
>
> 3. **减少数据库连接开销**
>
> 由于预加载减少了数据库查询次数，也就相应地减少了数据库连接的开销。数据库连接的建立和释放是有一定成本的，减少连接次数可以提高数据库的整体性能和效率。

## 一对多关系

假设一个用户有多个订单

```go
type Order struct {
    ID     uint `gorm:"primaryKey"`
    Item   string
    UserID uint
}

type User struct {
    ID     uint `gorm:"primaryKey"`
    Name   string
    Orders []Order
}
```

查询用户及其订单

```go
var user User
db.Preload("Orders").First(&user, 1)
```

> 如果已经预加载过了，并且数据没有改动，可以利用user.Orders来直接使用

## 多对多关系

如文章和标签存在多对多关系

```go
type Article struct {
    ID    uint `gorm:"primaryKey"`
    Title string
    Tags  []Tag `gorm:"many2many:article_tags;"`
}

type Tag struct {
    ID   uint `gorm:"primaryKey"`
    Name string
}
```

插入和查询多对多

```go
article := Article{
    Title: "GORM 教程",
    Tags: []Tag{
        {Name: "GoLang"},
        {Name: "ORM"},
    },
}
db.Create(&article)

var art Article
db.Preload("Tags").First(&art, article.ID)
```

## hook函数与中间件

GORM 提供了钩子函数，可以在数据操作的不同阶段执行自定义逻辑（例如 BeforeCreate、AfterCreate、BeforeUpdate、AfterUpdate 等）。

如在创建用户前打印日志：

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
    println("即将创建用户：", u.Name)
    return
}
```

这种机制有助于数据验证、记录操作日志、自动设置字段（如创建时间、更新时间）等。

> **钩子方法的顺序**：
>
> - GORM 提供了多个钩子方法，它们在不同的阶段被调用。例如：
>
>   - `BeforeCreate`：在创建记录之前被调用。
>   - `AfterCreate`：在创建记录之后被调用。
>   - `BeforeUpdate`、`AfterUpdate`、`BeforeDelete`、`AfterDelete` 等。
>
>   **返回错误**：
>
> - 如果在 `BeforeCreate` 方法中返回一个非空的 `error`，GORM 会终止创建操作并返回该错误。
>
> **事务支持**：
>
> - 钩子方法中的 `tx *gorm.DB` 参数支持事务操作。如果你需要在钩子方法中执行数据库操作，可以通过 `tx` 来完成。
>
> **日志记录**：
>
> - 在钩子方法中，你可以使用日志记录来跟踪操作的执行情况。这有助于调试和监控程序的行为。

## 性能优化

在实际项目中使用 GORM 时，可以考虑以下几点来提高性能和代码质量：

- **合理使用 Preload**：对于复杂关联查询，预加载（Preload）可以减少 N+1 查询问题，但在数据量较大时要注意性能；
- **事务管理**：对于需要保证操作[原子性](https://zhida.zhihu.com/search?content_id=253488216&content_type=Article&match_order=1&q=原子性&zhida_source=entity)的场景，使用事务（`db.Transaction()`）来管理操作；
- **批量操作**：尽可能使用批量插入和更新，减少数据库连接次数；
- **日志与调试**：开启 GORM 日志模式（`db.Debug()`）可以帮助调试，但在生产环境中建议关闭；
- **连接池配置**：合理配置数据库连接池参数（如最大连接数、[空闲连接数](https://zhida.zhihu.com/search?content_id=253488216&content_type=Article&match_order=1&q=空闲连接数&zhida_source=entity)、连接超时）以适应高并发需求。



## hook与中间件区别

**hook 函数**

- Hook 函数（钩子函数）是一种编程机制，它允许在某个程序的特定执行点插入自定义代码。这些特定执行点通常是在系统或框架的关键操作前后，比如事件触发、函数调用前后等。当程序执行到这些预设的点时，就会调用相应的 hook 函数，从而实现对程序流程的定制化扩展。
- 例如，在一些 Web 开发框架中，可能会提供在路由处理函数执行前和执行后插入 hook 函数的功能，开发者可以在这些 hook 函数中添加日志记录、权限验证等操作。

**中间件**

- 中间件是一种特殊的函数或组件，主要用于处理请求和响应的流程。在 Web 开发中，中间件通常位于客户端请求和服务器处理逻辑之间，它可以对请求进行预处理（如解析请求头、验证身份等），也可以对响应进行后处理（如添加响应头、压缩响应数据等）。
- 以 Gin 框架为例，中间件函数接收一个 `*gin.Context` 参数，通过这个上下文对象可以访问请求信息和设置响应信息，并且可以通过调用 `c.Next()` 方法将请求传递给下一个中间件或最终的处理函数。
