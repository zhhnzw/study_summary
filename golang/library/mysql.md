## MySQL

### 使用标准库

```go
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/go-sql-driver/mysql" // _ 开头import的含义是匿名导入，会执行导入包的 init 初始化函数。这句代码的功能即是加载了mysql驱动，驱动加载好后，database/sq标准库才能正常使用。
)

var db *sql.DB

func initMySQL() (err error) {
	// DSN:Data Source Name
	dsn := "root:root1234@tcp(127.0.0.1:13306)/sql_demo"
	// 去初始化全局的db对象而不是新声明一个db变量
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}
	// 做完错误检查之后，确保db不为nil
	// 尝试与数据库建立连接（校验dsn是否正确）
	err = db.Ping()
	if err != nil {
		fmt.Printf("connect to db failed, err:%v\n", err)
		return
	}
  // database/sql 包已经维护了连接池，可供多个goroutine并发使用
	// 数值需要业务具体情况来确定
	//db.SetConnMaxLifetime(time.Second*10) 连接最长存活时间
	db.SetMaxOpenConns(100) // 最大连接数
	db.SetMaxIdleConns(10)  // 最大空闲连接数
	return
}

type user struct {
	id   int
	age  int
	name string
}

// 预处理查询示例
func prepareQueryDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
  // 预处理适用场景：批量的执行同一条SQL语句，只是查询的字段值不同时，提前让服务器编译，一次编译多次执行，节省后续编译的成本，如下，当还需要查询stmt.Query(1)、stmt.Query(2) …… 时，增删改查都适用。
	rows, err := stmt.Query(0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	defer rows.Close()
	// 循环读取结果集中的数据
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			return
		}
		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
	}
}

// 任何时候都不应该自己拼接SQL语句，不能相信用户输入的数据一定是安全的。
// 不可以通过fmt包（或其他方式）直接拼接SQL，通过database/sql包的QueryRow、Prepare、Exec来传参是可以的
// sql注入示例
func sqlInjectDemo(name string) {
	sqlStr := fmt.Sprintf("select id, name, age from user where name='%s'", name)
	fmt.Printf("SQL:%s\n", sqlStr)
	var u user
	err := db.QueryRow(sqlStr).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Printf("exec failed, err:%v\n", err)
		return
	}
	fmt.Printf("user:%#v\n", u)
}

// 事务操作示例
func transactionDemo() {
	tx, err := db.Begin() // 开启事务
	if err != nil {
		if tx != nil {
			tx.Rollback() // 回滚
		}
		fmt.Printf("begin trans failed, err:%v\n", err)
		return
	}
	sqlStr1 := "Update user set age=30 where id=?"
	ret1, err := tx.Exec(sqlStr1, 2)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec sql1 failed, err:%v\n", err)
		return
	}
	affRow1, err := ret1.RowsAffected()
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec ret1.RowsAffected() failed, err:%v\n", err)
		return
	}

	sqlStr2 := "Update user set age=40 where id=?"
	ret2, err := tx.Exec(sqlStr2, 3)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec sql2 failed, err:%v\n", err)
		return
	}
	affRow2, err := ret2.RowsAffected()
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec ret1.RowsAffected() failed, err:%v\n", err)
		return
	}

	// 当affRow1 == 1 && affRow2 == 1
	fmt.Println(affRow1, affRow2)
	if affRow1 == 1 && affRow2 == 1 {
		fmt.Println("事务提交啦...")
		tx.Commit() // 提交事务
	} else {
		tx.Rollback()
		fmt.Println("事务回滚啦...")
	}

	fmt.Println("exec trans success!")
}

func main() {
	if err := initMySQL(); err != nil {
		fmt.Printf("connect to db failed, err:%v\n", err)
	}
	// Close() 用来释放掉数据库连接相关的资源
	defer db.Close() // 注意这行代码要写在上面err判断的下面
	fmt.Println("connect to db success")

	//prepareQueryDemo()
	//prepareInsertDemo()
  //SQL注入示例
  //sqlInjectDemo("xxx ' or 1=1#")
	//select id, name, age from user where name='xxx ' or 1=1#'   
  //#在SQL中代表注释，注释掉了后面的'符号，1=1恒成立，所以以上SQL语句等价于 select id, name, age from user
	//sqlInjectDemo("xxx' union select * from user #")
	//sqlInjectDemo("xxx' and (select count(*) from user) <10 #")
	transactionDemo()
	fmt.Println("查询结束了...")
}

```

database/sql **定义**了一套接口，规定了要实现的功能，在具体的驱动中再来**实现**这些接口功能，如以上的mysql，这样设计，在调用 database/sql 的接口时对于不同的驱动就可以复用同一套代码（其他驱动分别实现这些接口功能即可，如PostgreSQL、Microsoft SQL Server等），这个设计充分体现了 golang 面相接口开发的特点。

### 第三方包sqlx

```go
type User struct {
  Id   int		`db:"id"`
	Name string `db:"name"`
	Age  int    `db:"age"`
}
// 查询单条数据示例
func queryRowDemo() {
	sqlStr := "select id, name, age from user where id=?"
	var u user
	err := db.Get(&u, sqlStr, 1)
	if err != nil {
		fmt.Printf("get failed, err:%v\n", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.ID, u.Name, u.Age)
}
// 查询多条数据示例
func queryMultiRowDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	var users []user
	err := db.Select(&users, sqlStr, 0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	fmt.Printf("users:%#v\n", users)
}
// QueryByIDs 根据给定ID查询
func QueryByIDs(ids []int)(users []User, err error){
	// 动态填充id
	query, args, err := sqlx.In("SELECT name, age FROM user WHERE id IN (?)", ids)
	if err != nil {
		return
	}
	// sqlx.In 返回带 `?` bindvar的查询语句, 使用Rebind()重新绑定它
	query = DB.Rebind(query)
	err = DB.Select(&users, query, args...)
	return
}
// 使用NamedExec实现批量插入，不需要拼接占位符？和数据了，方便很多
func BatchInsertUsers(users []*User) error {
	_, err := DB.NamedExec("INSERT INTO user (name, age) VALUES (:name, :age)", users)
	return err
}

```