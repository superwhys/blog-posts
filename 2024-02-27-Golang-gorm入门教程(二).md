---
layout: 	post
title: 	    Golang GORM入门教程(二) 
subTitle:   Golang GORM CRUD 操作
date: 		2024-02-27
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

# 3. 连接到数据库
`GORM` 官方支持的数据库类型有：MySQL, PostgreSQL, SQLite, SQL Server 和 TiDB
在本系列教程，我会统一都使用 `SQLite` 数据库进行操作
```go
import (
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

func main() {
    db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}
}
```

# 4. CRUD 操作
我统一使用下面的 model 进行演示
```go
type Product struct {
	gorm.Model
	Code  string
	Price uint
}
```
在连接上数据库之后，我们可以使用 `AutoMigrate` 对数据库进行迁移来自动生成数据库表

## (1). 创建
```go
product := Product{Code: "000001", Price: 100}

result := db.Create(&product) // 通过数据的指针来创建

product.ID             // 返回插入数据的主键
result.Error        // 返回 error
result.RowsAffected // 返回插入记录的条数
```

我们还可以使用 `Create()` 创建多项记录
```go
products := []*Product{
    Product{Code: "000002", Price: 101},
    Product{Code: "000003", Price: 101},
}

result := db.Create(&products) // 传递切片以插入多行数据

result.Error        // 返回 error
result.RowsAffected // 返回插入记录的条数
```

创建记录的时候，我们还可以指定只为某些字段赋值
```go
db.Select("Code", "CreatedAt").Create(&product)
// INSERT INTO `products` (`code`,`created_at`) VALUES ("000001", "2024-02-27 11:05:21.775")
```

我们还可以选择忽略某些字段
```go
db.Omit("Code", "CreatedAt").Create(&product)
// INSERT INTO `products` (`price`,`updated_at`) VALUES ("100", "2024-02-27 11:05:21.775")
```

在批量插入的时候，我们还可以指定插入的批次大小
```go
var products = []Product{{Code: "000001"}, ...., {Code: "000001000"}}

// batch size 100
db.CreateInBatches(&products, 100)
```

## (2). 查询
### 查询单个对象
`GORM` 提供了 `First`、`Take`、`Last` 方法，以便从数据库中检索单个对象。

当查询数据库时它添加了 `LIMIT` 1 条件，且没有找到记录时，它会返回 `ErrRecordNotFound` 错误。
```go
// 获取第一条记录（主键升序）
db.First(&product)
// SELECT * FROM products ORDER BY id LIMIT 1;

// 获取一条记录，没有指定排序字段
db.Take(&product)
// SELECT * FROM products LIMIT 1;

// 获取最后一条记录（主键降序）
db.Last(&product)
// SELECT * FROM products ORDER BY id DESC LIMIT 1;

result := db.First(&product)
result.RowsAffected // 返回找到的记录数
result.Error        // returns error or nil

// 检查 ErrRecordNotFound 错误
errors.Is(result.Error, gorm.ErrRecordNotFound)
```

注意，对于 `First` 和 `Last` 只有当目标 struct 是指针或者通过 db.Model() 指定 model 时，该方法才有效。
```go
result := &Product{}
db.Table("products").First(&result)

// it doesn't work
result := map[string]interface{}{}
db.Table("products").First(&result)

// work with take
result := map[string]interface{}{}
db.Table("products").Take(&result)
```

### 根据主键检索
如果主键是数字类型，您可以使用 `内联条件` 来检索对象。 当使用字符串时，我们可以使用参数占位来注意避免`SQL注入`
```go
db.First(&product, 10)
// SELECT * FROM products WHERE id = 10;

db.First(&product, "10")
// SELECT * FROM products WHERE id = 10;

var products []*Product
db.Find(&products, []int{1,2,3})
// SELECT * FROM products WHERE id IN (1,2,3);

db.First(&product, "id = ?", "1b74413f-f3b8-409f-ac47-e8c062e3472a")
// SELECT * FROM products WHERE id = "1b74413f-f3b8-409f-ac47-e8c062e3472a";
```

当目标对象有一个主键值时，将使用主键构建查询条件
```go
var product = Product{ID: 10}
db.First(&product)
// SELECT * FROM products WHERE id = 10;

var result Product
db.Model(Product{ID: 10}).First(&result)
// SELECT * FROM products WHERE id = 10;
```

### 检索全部对象
```go
// Get all records
result := db.Find(&products)
// SELECT * FROM products;

result.RowsAffected // returns found records count, equals `len(products)`
result.Error        // returns error
```

### 条件查询
#### String条件
```go
// Get first matched record
db.Where("code = ?", "000001").First(&product)
// SELECT * FROM products WHERE code = '000001' ORDER BY id LIMIT 1;

// Get all matched records
db.Where("code <> ?", "000001").Find(&products)
// SELECT * FROM products WHERE code <> '000001';

// IN
db.Where("code IN ?", []string{"000001", "000002"}).Find(&products)
// SELECT * FROM products WHERE code IN ('000001','000002');

// LIKE
db.Where("code LIKE ?", "%001%").Find(&products)
// SELECT * FROM products WHERE code LIKE '%001%';

// AND
db.Where("id >= ? AND id <= ?", "5", "10").Find(&products)
// SELECT * FROM products WHERE id >= 5 AND id <= 10;

// Time
db.Where("updated_at > ?", lastWeek).Find(&products)
// SELECT * FROM products WHERE updated_at > '2000-01-01 00:00:00';

// BETWEEN
db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&products)
// SELECT * FROM products WHERE created_at BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';
```

如果对象设置了主键，条件查询将不会覆盖主键的值，而是用 `And` 连接条件。
```go
var product = Product{ID: 10}
db.Where("id = ?", 20).First(&product)
// SELECT * FROM products WHERE id = 10 and id = 20 ORDER BY id ASC LIMIT 1
```

#### Struct & Map 条件
```go
// Struct
db.Where(&Product{code: "000001"}).First(&product)
// SELECT * FROM products WHERE code = "000001" ORDER BY id LIMIT 1;

// Map
db.Where(map[string]interface{}{"code": "000001"}).Find(&products)
// SELECT * FROM products WHERE code = "000001";

// Slice of primary keys
db.Where([]int64{20, 21, 22}).Find(&products)
// SELECT * FROM products WHERE id IN (20, 21, 22);
```

#### 指定结构体查询字段
在只用结构体条件进行查询时，我们还可以指定使用哪些字段进行查询
```go
db.Where(&product{code: "000001"}, "code", "price").Find(&products)
// SELECT * FROM products WHERE code = "000001" AND price = 0;

db.Where(&product{code: "000001"}, "price").Find(&products)
// SELECT * FROM products WHERE price = 0;
```

#### 内联查询
查询条件可以以与`Where`类似的方式内联到 `First` 和 `Find` 等方法中。
```go
// Get by primary key if it were a non-integer type
db.First(&product, "id = ?", "10")
// SELECT * FROM products WHERE id = '10';

// Plain SQL
db.Find(&product, "code = ?", "000001")
// SELECT * FROM products WHERE code = "000001";

db.Find(&products, "id >= ? AND id <= ?", 10, 20)
// SELECT * FROM products WHERE id >= 10 AND id <= 20;

// Struct
db.Find(&products, product{Code: "000001"})
// SELECT * FROM products WHERE code = "000001";

// Map
db.Find(&products, map[string]interface{}{"code": "000001"})
// SELECT * FROM products WHERE code = "000001";
```

#### Not 条件
```go
db.Not("code = ?", "000001").First(&product)
// SELECT * FROM products WHERE NOT code = "000001" ORDER BY id LIMIT 1;
```

#### Or 条件
```go
db.Where("price = ?", "100").Or("price = ?", "200").Find(&products)
// SELECT * FROM products WHERE price = 100 OR price = 200;
```

#### 选择特定字段
```go
db.Select("code", "price").Limit(10).Find(&products)
// SELECT code, age FROM products LIMIT 10;
```

## (3). 更新

### 保存所有字段
我们可以使用 `Save` 来进行更新操作，但是它会保存所有的字段，包括零值
```go
db.First(&product)

product.Code = "10000001"
product.Price = 1111
db.Save(&product)
// UPDATE products SET code='10000001', price=1111, updated_at = '2024-02-27 12:34:10' WHERE id=1;
```

### 更新单个列
```go
// 根据条件更新
db.Model(&Proeuct{}).Where("code = ?", "10000001").Update("price", "2222")
// UPDATE products SET price=2222, updated_at='2024-02-27 12:34:10' WHERE code="10000001";

// product 的 ID 是 `111`
db.Model(&product).Update("price", "3333")
// UPDATE products SET price=3333, updated_at='2024-02-27 12:34:10' WHERE id=111;

// 根据条件和 model 的值进行更新
db.Model(&product).Where("code = ?", "10000001").Update("price", "3333")
// UPDATE products SET price=3333, updated_at='2024-02-27 12:34:10' WHERE id=111 AND code=10000001;
```

### 更新多列
`Updates` 方法支持 `struct` 和 `map[string]interface{}` 参数。当使用 `struct` 更新时，默认情况下 `GORM` 只会更新非零值的字段
```go
// 根据 `struct` 更新属性，只会更新非零值的字段
db.Model(&product).Updates(product{code: "10000001", price: 20})
// UPDATE products SET code='10000001', price=20, updated_at = '2024-02-24 12:34:10' WHERE id = 111;

// 根据 `map` 更新属性
db.Model(&product).Updates(map[string]interface{}{"code": "10000001", "age": 18})
// UPDATE products SET code='10000001', age=18, updated_at='2024-02-24 12:34:10' WHERE id=111;
```

### 其他
如果您想要在更新时选择、忽略某些字段，您可以使用 `Select`、`Omit`

## (4). 删除
### 删除一条记录
删除一条记录时，删除对象需要指定主键，否则会触发 `批量删除`，
```go
// product 的 ID 是 `10`
db.Delete(&product)
// DELETE from products where id = 10;

// 带额外条件的删除
db.Where("price = ?", "2222").Delete(&product)
// DELETE from products where id = 10 AND price = 2222;
```

### 根据主键删除
`GORM` 允许通过主键(可以是复合主键)和内联条件来删除对象，它可以使用数字也可以是字符串
```go
db.Delete(&Proeuct{}, 10)
// DELETE FROM products WHERE id = 10;

db.Delete(&product{}, "10")
// DELETE FROM products WHERE id = 10;

db.Delete(&products, []int{1,2,3})
// DELETE FROM products WHERE id IN (1,2,3);
```

### 批量删除
如果指定的值不包括主属性，那么 GORM 会执行批量删除，它将删除所有匹配的记录
```go
db.Where("code LIKE ?", "%0000%").Delete(&Product{})
// DELETE from products where code LIKE "%0000%";

db.Delete(&Product{}, "code LIKE ?", "%0000%")
// DELETE from products where code LIKE "%0000%";
```

### 软删除
如果你的模型包含了 `gorm.DeletedAt` 字段（该字段也被包含在`gorm.Model`中），那么该模型将会自动获得软删除的能力！

当调用 `Delete` 时，GORM并不会从数据库中删除该记录，而是将该记录的 `DeleteAt` 设置为当前时间，而后的一般查询方法将无法查找到此条记录。

```go
// product's ID is `111`
db.Delete(&product)
// UPDATE products SET deleted_at="2024-02-27 10:23" WHERE id = 111;

// Batch Delete
db.Where("price = ?", 1000).Delete(&product{})
// UPDATE products SET deleted_at="2024-02-27 10:23" WHERE price = 1000;

// Soft deleted records will be ignored when querying
db.Where("price = ?", 1000).Find(&product)
// SELECT * FROM products WHERE price = 1000 AND deleted_at IS NULL;
```

#### 查询被软删除的记录
你可以使用`Unscoped`来查询到被软删除的记录
```go
db.Unscoped().Where("code = ?", 10000001).Find(&products)
// SELECT * FROM products WHERE code = 10000001;
```

### 永久删除
你可以使用`Unscoped`来永久删除匹配的记录
```go
// id 为10
db.Unscoped().Delete(&product)
// DELETE FROM products WHERE id=10;
```
