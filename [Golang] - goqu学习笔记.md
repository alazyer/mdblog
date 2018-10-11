# goqu学习笔记

## goqu是什么

Github上说是一个SQL Builder，个人不是很理解，获取可以简单理解为用来和数据库进行交互的一个库肯定没错，哈哈



既然是和数据库交互的，那肯定就会涉及到：

- 与数据库建立连接
- 查询数据库
- 插入数据库
- 更新数据库
- 删除数据库
- 会话管理

## 与数据库建立连接

```go
package main

import (
    "fmt"
    "database/sql"
    "gopkg.in/doug-martin/goqu.v4"
    _ "github.com/go-sql-driver/mysql"
    _ "gopkg.in/doug-martin/goqu.v4/adapters/postgres"
    _ "github.com/lib/pq"
)

func NewPostgreDb() *goqu.Database{
    // host=%v port=%v user=%v password='%v' dbname=%v connect_timeout=%d sslmode=%v
    pgDb, err := sql.Open("postgres", "user=postgres dbname=goqupostgres sslmode=disable ")
	if err != nil {
    	panic(err.Error())
	}
	db := goqu.New("postgres", pgDb)
    return db
}

func NewMysqlDb() *goqu.Database{
    // username:password@host/database?params
    pgDb, err := sql.Open("mysql", "root:123456@goqupostgres/dbname")
	if err != nil {
    	panic(err.Error())
	}
	db := goqu.New("mysql", pgDb)
    return db
}
```

## 查询数据库

- `goqu.Ex`，表达式字典，字典中条件and连接
- `goqu.ExOr`，表达式字典，字典中条件or连接
- `goqu.Op`，支持的操作字典。支持`['eq', 'neq', 'is', 'isnot', 'gt', 'gte', 'lt', 'lte', 'in', 'notin', 'like', 'notlike', 'ilike', 'notilike', 'between', 'notbetween']`
- `goqu.I`，生成更复杂的`IdentifierExpression`对象，是对`goqu.Ex`的升级，`goqu.I("my_schema.table.col"); goqu.I("table.col"); goqu.I("col")`
- `goqu.L`，如果`IdentifierExperssion`还不能满足要求的话，就可以使用这个生成一个`LiteralExperssion`。`goqu.L("col IN (?, ?, ?)", "a", "b", "c")`
- `goqu.Or`，作用和`goqu.ExOr`类似，只不过`goquExOr`在表达式内用，而`goqu.Or`是在表达式之间使用。与`goqu.Or`对应的还有`goqu.And`

```go
db := NewPostgreDb()
 
// select * from items where col1 = 'a' and col2 = 1 and col3 in ('a', 'b', 'c');
items := db.From("items").Where(
    goqu.Ex{
        "col1": "a",
        "col2": 1,
        "col3": []string{"a", "b", "c"},
    }
)

items := db.From("items").Where(
    goqu.I("col1").Eq("a"),
    goqu.I("col2").Eq(1),
    goqu.I("col3").In([]string{"a", "b", "c"}),
)

items := db.From("items").Where(
    goqu.L("col1=?", "a"),
    goqu.L("col2=?", 1),
    goqu.L("col3 in (?, ?, ?)", "a", "b", "c"),
)

// select * from items where col1 = 'a' or col2 = 1 or col3 in ('a', 'b', 'c');
items := db.From("items").Where(
    goqu.ExOr{
        "col1": "a",
        "col2": 1,
        "col3": []string{"a", "b", "c"},
    }
)

items := db.From("items").Where(
    goqu.Or(
        goqu.I("col1").Eq("a"),
        goqu.I("col2").Eq(1),
        goqu.I("col3").In([]string{"a", "b", "c"}),
    )
)

// select * from items where col1 != 'a' and col2 = 1 and col3 not in ('a', 'b', 'c');
items := db.From("items").Where(
    goqu.Ex{
        "col1": goqu.Op{"neq": "a"},
        "col2": 1,
        "col6": goqu.Op{"notin", []string{"a", "b", "c"}},
    }
)

// select * from items where col1 = 'a' or (col2 = 1 and col3 not in ('a', 'b', 'c'));
items := db.From("items").Where(
    goqu.Or(
        goqu.I("col1").Eq("a"),
        goqu.And(
            goqu.I("col2").Eq(1),
            goqu.I("col6").NotIn([]string{"a", "b", "c"})
        )
    )
)
```

## 插入数据库

```go
users := []goqu.Record{
    {"first_name": "Bob", "last_name":"Yukon", "created": time.Now()},
    {"first_name": "Sally", "last_name":"Yukon", "created": time.Now()},
    {"first_name": "Jimmy", "last_name":"Yukon", "created": time.Now()},
}
if _, err := db.From("user").Insert(users).Exec(); err != nil{
    fmt.Println(err.Error())
    return
}
```

## 更新数据库

```go
update := db.From("user").
    Where(goqu.I("status").Eq("inactive")).
    Update(goqu.Record{"password": nil, "updated": time.Now()})

if _, err := update.Exec(); err != nil{
    fmt.Println(err.Error())
    return
}
```

## 删除数据库

```go
delete := db.From("invoice").
    Where(goqu.Ex{"status":"paid"}).
    Delete()
if _, err := delete.Exec(); err != nil{
    fmt.Println(err.Error())
    return
}
```

## 会话管理

```
// transaction with TxDatabase.Commit and TxDatabase.Rollback
func DeleteTemplate(uuid string) error {
	var (
		tx  *goqu.TxDatabase
		err error
	)
	
	if tx, err = db.Begin(); err != nil {
		return err
	}
	defer func() {
		if err != nil {
			tx.Rollback()
		} else {
			tx.Commit()
		}
	}()

	if _, err = db.From("invoice").Where(goqu.Ex{"id": uuid}).Delete().Exec(); err != nil {
		return err
	}
	
	return nil
}

// transaction with TxDatabase.Wrap
func DeleteTemplate(uuid string) error {
	var (
		tx  *goqu.TxDatabase
		err error
	)
	
	if tx, err = db.Begin(); err != nil {
		return err
	}
	
	err = tx.Wrap(func() error{
        delete = db.From("invoice").Where(goqu.Ex{"id": uuid}).Delete()
        if _, err = delete.Exec(); err != nil {
            return err
        }
        return nil
	})
	
	return err
}
```

## Others

除了上面提到的，还有一些其他的方法：

- `Count`，获取查询结果数量
- `Prepared`，用于延时转换。一般`db.From().Select().Order().Prepared()`，在返回结果上调用`Where`
- `ScanStructs`，用于将数据库中多条返回结果和给定的slice指针绑定
- `ScanStruct`，功能和`ScanStructs`类似，只不过只取一条记录（返回结果比一定只有一条记录）
- `ScanVals`、`ScanVal`和对应的`ScanStructs`、`ScanStruct`类似，只不过对应数据库中列，后二者对应行

```go
// --------------- Count ---------------
count, err := db.From("user").Count()
if err != nil{
    fmt.Println(err.Error())
    return
}
fmt.Printf("\nCount:= %d", count)

// --------------- Prepared ---------------
preparedDs := db.From("items").Prepared(true)

sql, args, _ := preparedDs.Where(goqu.Ex{
	"col1": "a",
	"col2": 1,
	"col3": true,
	"col4": false,
	"col5": []string{"a", "b", "c"},
}).ToSql()
fmt.Println(sql, args)

sql, args, _ = preparedDs.ToInsertSql(
	goqu.Record{"name": "Test1", "address": "111 Test Addr"},
	goqu.Record{"name": "Test2", "address": "112 Test Addr"},
)
fmt.Println(sql, args)

sql, args, _ = preparedDs.ToUpdateSql(
	goqu.Record{"name": "Test", "address": "111 Test Addr"},
)
fmt.Println(sql, args)

sql, args, _ = preparedDs.
	Where(goqu.Ex{"id": goqu.Op{"gt": 10}}).
	ToDeleteSql()
fmt.Println(sql, args)

// Output:
// SELECT * FROM "items" WHERE (("col1" = ?) AND ("col2" = ?) AND ("col3" IS TRUE) AND ("col4" IS FALSE) AND ("col5" IN (?, ?, ?))) [a 1 a b c]
// INSERT INTO "items" ("address", "name") VALUES (?, ?), (?, ?) [111 Test Addr Test1 112 Test Addr Test2]
// UPDATE "items" SET "address"=?,"name"=? [111 Test Addr Test]
// DELETE FROM "items" WHERE ("id" > ?) [10]

// --------------- ScanStructs ---------------
type User struct{
    FirstName string `db:"first_name"`
    LastName  string `db:"last_name"`
}

var users []User
// SELECT "first_name", "last_name" FROM "user";
if err := db.From("user").ScanStructs(&users); err != nil{
    fmt.Println(err.Error())
    return
}
fmt.Printf("\n%+v", users)

var users []User
// SELECT "first_name" FROM "user";
if err := db.From("user").Select("first_name").ScanStructs(&users); err != nil{
    fmt.Println(err.Error())
    return
}
fmt.Printf("\n%+v", users)

// --------------- ScanStruct ---------------
var user User
//SELECT "first_name", "last_name" FROM "user" LIMIT 1;
found, err := db.From("user").ScanStruct(&user)
if err != nil{
    fmt.Println(err.Error())
    return
}
if !found {
    fmt.Println("No user found")
} else {
    fmt.Printf("\nFound user: %+v", user)
}

// --------------- ScanVals ---------------
var ids []int64
if err := db.From("user").Select("id").ScanVals(&ids); err != nil{
    fmt.Println(err.Error())
    return
}
fmt.Printf("\n%+v", ids)

// --------------- ScanVal ---------------
var id int64
found, err := db.From("user").Select("id").ScanVal(&id)
if err != nil{
    fmt.Println(err.Error())
    return
}
if !found{
    fmt.Println("No id found")
}else{
    fmt.Printf("\nFound id: %d", id)
}

```

