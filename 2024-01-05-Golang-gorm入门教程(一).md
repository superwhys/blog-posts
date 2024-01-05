---
layout: 	post
title: 	    Golang GORM入门教程(一) 
subTitle:   Golang GORM 模型声明
date: 		2024-01-05
author:     Yong
tags:
    - Go
    - Gorm
---

# 1. 介绍
gorm 是 golang 中最流行的 orm 框架，为 go 语言使用者提供了简便且丰富的数据库操作 api.

gorm 本身也支持多种数据库类型，在本文中，统一以 sqlite 作为操作的数据库类型.

- 开源地址：https://github.com/go-gorm/gorm
- 中文教程：https://gorm.io/zh_CN/docs/

# 2. 安装

```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/sqlite
```

# 3. 模型
## 3.1. 模型定义
模型是标准的 struct，由 Go 的基本数据类型、实现了 `Scanner` 和 `Valuer` 接口的自定义类型及其指针或别名组成

例如：
```go
type User struct {
  ID           uint
  Name         string
  Email        *string
  Age          uint8
  Birthday     *time.Time
  MemberNumber sql.NullString
  ActivatedAt  sql.NullTime
  CreatedAt    time.Time
  UpdatedAt    time.Time
}
```

在gorm中还提供了一个基础模型 `gorm.Model`, 它提供了基础的`ID`, `CreatedAt`, `UpdatedAt`, `DeletedAt` 四个字段

我们可以选择将它嵌套在我们的自定义额模型上

```go
type Model struct {
	ID        uint `gorm:"primarykey"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt DeletedAt `gorm:"index"`
}
```

## 3.2. 模型的一些约定
在gorm中，我们更倾向于约定优于配置

默认情况下，GORM 使用 `ID` 作为主键

使用结构体名的 `蛇形复数` 作为表名

字段名的 `蛇形` 作为列名

并使用 `CreatedAt`、`UpdatedAt` 字段追踪创建、更新时间

如果遵循 GORM 的约定，可以少写许多配置和代码

### 使用ID作为主键
默认情况下，GORM 会使用 ID 作为表的主键。
```go
type User struct {
  ID   string // 默认情况下，名为 `ID` 的字段会作为表的主键
  Name string
}
```

也可以通过标签 primaryKey 将其它字段设为主键

```go
// 将 `UUID` 设为主键
type Animal struct {
  ID     int64
  UUID   string `gorm:"primaryKey"`
  Name   string
  Age    int64
}
```

### 复数表名
GORM 使用结构体名的 `蛇形命名` 作为表名。对于结构体 `User`，根据约定，其表名为 `users`

#### TableName
我们可以实现 `Tabler` 接口来更改默认表名，例如：
```go
type Tabler interface {
    TableName() string
}

// TableName 会将 User 的表名重写为 `profiles`
func (User) TableName() string {
  return "profiles"
}

```

#### 临时指定表名
我们还可以使用 Table 方法临时指定表名，例如：
```go
// 根据 User 的字段创建 `deleted_users` 表
db.Table("deleted_users").AutoMigrate(&User{})

// 从另一张表查询数据
var deletedUsers []User
db.Table("deleted_users").Find(&deletedUsers)
// SELECT * FROM deleted_users;

db.Table("deleted_users").Where("name = ?", "jinzhu").Delete(&User{})
// DELETE FROM deleted_users WHERE name = 'jinzhu';
```

#### 命名策略
GORM 允许用户通过覆盖默认的命名策略更改默认的命名约定

命名策略被用于构建：`TableName`、`ColumnName`、`JoinTableName`、`RelationshipFKName`、`CheckerName`、`IndexName`。

详细的命名策略配置,查看 [GORM命名策略配置](https://gorm.io/zh_CN/docs/gorm_config.html#%E5%91%BD%E5%90%8D%E7%AD%96%E7%95%A5)

###  列名
根据约定，数据表的列名使用的是 struct 字段名的 蛇形命名
```go
type User struct {
  ID        uint      // 列名是 `id`
  Name      string    // 列名是 `name`
  Birthday  time.Time // 列名是 `birthday`
  CreatedAt time.Time // 列名是 `created_at`
}
```

我们可以使用 `column` 标签或 `命名策略` 来覆盖列名
```go
type Animal struct {
  AnimalID int64     `gorm:"column:beast_id"`         // 将列名设为 `beast_id`
  Birthday time.Time `gorm:"column:day_of_the_beast"` // 将列名设为 `day_of_the_beast`
  Age      int64     `gorm:"column:age_of_the_beast"` // 将列名设为 `age_of_the_beast`
}
```

### 时间戳追踪
#### CreatedAt
对于有 `CreatedAt` 字段的模型，创建记录时，如果该字段值为零值，则将该字段的值设为当前时间
```go
db.Create(&user) // 将 `CreatedAt` 设为当前时间

user2 := User{Name: "jinzhu", CreatedAt: time.Now()}
db.Create(&user2) // user2 的 `CreatedAt` 不会被修改

// 想要修改该值，您可以使用 `Update`
db.Model(&user).Update("CreatedAt", time.Now())
```

你可以通过将 autoCreateTime 标签置为 false 来禁用时间戳追踪，例如：
```go
type User struct {
  CreatedAt time.Time `gorm:"autoCreateTime:false"`
}
```

#### UpdatedAt
对于有 `UpdatedAt` 字段的模型，更新记录时，将该字段的值设为当前时间。

创建记录时，如果该字段值为零值，则将该字段的值设为当前时间

如果更新的时候使用了 `UpdateColumn` 指定更新某一列，UpdatedAt也不会被修改
```go
db.Save(&user) // 将 `UpdatedAt` 设为当前时间

db.Model(&user).Update("name", "jinzhu") // 会将 `UpdatedAt` 设为当前时间

db.Model(&user).UpdateColumn("name", "jinzhu") // `UpdatedAt` 不会被修改

user2 := User{Name: "jinzhu", UpdatedAt: time.Now()}
db.Create(&user2) // 创建记录时，user2 的 `UpdatedAt` 不会被修改

user3 := User{Name: "jinzhu", UpdatedAt: time.Now()}
db.Save(&user3) // 更新时，user3 的 `UpdatedAt` 会修改为当前时间
```

你可以通过将 autoUpdateTime 标签置为 false 来禁用时间戳追踪，例如：
```go
type User struct {
  UpdatedAt time.Time `gorm:"autoUpdateTime:false"`
}
```

GORM 还支持拥有多种类型的时间追踪字段。可以根据 UNIX（毫/纳）秒，

我们可以使用 `autoUpdateTime` 标签来指定使用哪种时间类型
```go
type User struct {
  CreatedAt time.Time // 在创建时，如果该字段值为零值，则使用当前时间填充
  UpdatedAt int       // 在创建时该字段值为零值或者在更新时，使用当前时间戳秒数填充
  Updated   int64 `gorm:"autoUpdateTime:nano"` // 使用时间戳纳秒数填充更新时间
  Updated   int64 `gorm:"autoUpdateTime:milli"` // 使用时间戳毫秒数填充更新时间
  Created   int64 `gorm:"autoCreateTime"`      // 使用时间戳秒数填充创建时间
}
```

## 3.3. 字段权限控制
在定义的模型中，可导出的字段在进行CRUD的时候拥有所有权限

当然, GORM 允许我们用标签控制字段级别的权限。这样就可以让一个字段的权限被指定为`只读`、`只写`、`只创建`、`只更新`或者`被忽略`

当我们使用 GORM Migrator 创建表时，不会创建被忽略的字段
```go
type User struct {
  Name string `gorm:"<-:create"` // 允许读和创建
  Name string `gorm:"<-:update"` // 允许读和更新
  Name string `gorm:"<-"`        // 允许读和写（创建和更新）
  Name string `gorm:"<-:false"`  // 允许读，禁止写
  Name string `gorm:"->"`        // 只读（除非有自定义配置，否则禁止写）
  Name string `gorm:"->;<-:create"` // 允许读和写
  Name string `gorm:"->:false;<-:create"` // 仅创建（禁止从 db 读）
  Name string `gorm:"-"`  // 通过 struct 读写会忽略该字段
  Name string `gorm:"-:all"`        // 通过 struct 读写、迁移会忽略该字段
  Name string `gorm:"-:migration"`  // 通过 struct 迁移会忽略该字段
}
```

## 3.4. 模型嵌套
对于匿名字段，GORM 会将其字段包含在父结构体中，例如：
```go
type User struct {
  gorm.Model
  Name string
}
// 等效于
type User struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`
  Name string
}
```

对于正常的结构体字段，我们也可以通过标签 `embedded` 将其嵌入，例如：
```go
type Author struct {
    Name  string
    Email string
}

type Blog struct {
  ID      int
  Author  Author `gorm:"embedded"`
  Upvotes int32
}
// 等效于
type Blog struct {
  ID    int64
  Name  string
  Email string
  Upvotes  int32
}
```

并且，我们可以使用标签 `embeddedPrefix` 来为 `db` 中的字段名添加前缀，例如：
```go
type Blog struct {
  ID      int
  Author  Author `gorm:"embedded;embeddedPrefix:author_"`
  Upvotes int32
}
// 等效于
type Blog struct {
  ID          int64
  AuthorName string
  AuthorEmail string
  Upvotes     int32
}
```

## 3.5. 字段标签
关于 gorm 模型字段支持的标签

我们可以看官方文档 [gorm字段标签](https://gorm.io/zh_CN/docs/models.html#%E5%AD%97%E6%AE%B5%E6%A0%87%E7%AD%BE)

