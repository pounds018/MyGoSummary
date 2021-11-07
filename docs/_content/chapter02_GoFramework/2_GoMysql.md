## 2.1 连接数据库:

Go语言的`database/sql`包提供了保证SQL或者类SQL数据库的泛用接口,但是并没有提供具体的数据库驱动,驱动就是各个数据库厂商提供的连接数据库的工具.

**下载依赖:**

下载依赖的时候需要到`GOPATH`目录下去下载

```go
go get -u github.com/go-sql-driver/mysql
```

### 2.1.1  获取DB连接池:

```go
func Open(driverName, dataSourceName string) (*DB, error)
```

说明: 

- `dirverName`: 数据库驱动的名字,比如: `mysql`
- `dataSourceName`: 数据源,完整格式为:  `userName:password@tcp(host:port)/database`一般包至少括数据库文件名和（可能的）连接信息。
- Open函数可能只是验证其参数，而不创建与数据库的连接。如果要检查数据源的名称是否合法，应调用`DB的Ping方法`。
- 返回的DB可以安全的被多个go程同时使用，并会维护自身的闲置连接池, `也就是说: DB是一个链接池对象`

```go
func connectMysql(){
	// 数据源 userName:password@tcp(host:port)/database
	dsn := "root:root1@tcp(192.168.200.129:33066)/go_study"
	// 连接数据库
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}
    // 这里应该要写在 err != nil的后面
	defer func(db *sql.DB) {
		err := db.Close()
		if err != nil {

		}
	}(db)
}
```

### 2.1.2 创建连接:

Open函数,只是用来校验数据源参数是否符合约定,并返回一个连接池,但是`不会去连接到数据库`,连接到数据库 需要使用`DB.Ping`方法.

<font color=red>为了 `DB`这个连接池能够被所有的方法使用 需要将其定义为全局变量.</font>

```go
// 通常都是将DB设置为全局变量来使用的
var db *sql.DB
func initDb() (err error){
   // 数据源 userName:password@tcp(host:port)/database
   dsn := "root:root@tcp(192.168.200.129:3306)/go_study"
   // 连接数据库,这里不要使用短变量声明,不然不能赋值给全局变量
   db, err = sql.Open("mysql", dsn)
   if err != nil{
      return
   }
   // 从连接池里面搞一个连接,连接到数据库,密码等校验也是在这里做的
   err = db.Ping()
   if err != nil {
      return
   }
   return nil
}
```

其中`sql.DB`是表示连接的数据库对象（结构体实例），它保存了连接数据库相关的所有信息。它内部维护着一个具有零到多个底层连接的连接池，它可以安全地被多个`goroutine`同时使用。

```go
type DB struct {
   // Atomic access only. At top of struct to prevent mis-alignment
   // on 32-bit platforms. Of type time.Duration.
   waitDuration int64 // Total time waited for new connections.

   connector driver.Connector
   // numClosed is an atomic counter which represents a total number of
   // closed connections. Stmt.openStmt checks it before cleaning closed
   // connections in Stmt.css.
   numClosed uint64

   mu           sync.Mutex // protects following fields
   freeConn     []*driverConn
   connRequests map[uint64]chan connRequest
   nextRequest  uint64 // Next key to use in connRequests.
   numOpen      int    // number of opened and pending open connections
   // Used to signal the need for new connections
   // a goroutine running connectionOpener() reads on this chan and
   // maybeOpenNewConnections sends on the chan (one send per needed connection)
   // It is closed during db.Close(). The close tells the connectionOpener
   // goroutine to exit.
   openerCh          chan struct{}
   closed            bool
   dep               map[finalCloser]depSet
   lastPut           map[*driverConn]string // stacktrace of last conn's put; debug only
   maxIdleCount      int                    // zero means defaultMaxIdleConns; negative means 0
   maxOpen           int                    // <= 0 means unlimited
   maxLifetime       time.Duration          // maximum amount of time a connection may be reused
   maxIdleTime       time.Duration          // maximum amount of time a connection may be idle before being closed
   cleanerCh         chan struct{}
   waitCount         int64 // Total number of connections waited for.
   maxIdleClosed     int64 // Total number of connections closed due to idle count.
   maxIdleTimeClosed int64 // Total number of connections closed due to idle time.
   maxLifetimeClosed int64 // Total number of connections closed due to max connection lifetime limit.

   stop func() // stop cancels the connection opener.
}
```

**设置与数据库最大连接数:**

```go
func (db *DB) SetMaxOpenConns(n int)
```

`SetMaxOpenConns`设置与数据库建立连接的最大数目。 如果n大于0且小于最大闲置连接数，会将最大闲置连接数减小到匹配最大开启连接数的限制。 如果n<=0，不会限制最大开启连接数，默认为0（无限制）。

**设置连接池中最大限制连接数:**

```go
func (db *DB) SetMaxIdleConns(n int)
```

`SetMaxIdleConns`设置连接池中的最大闲置连接数。 如果n大于最大开启连接数，则新的最大闲置连接数会减小到匹配最大开启连接数的限制。 如果n<=0，不会保留闲置连接。

## 2.2 crud:

建表:

```go
CREATE TABLE `t_go_study_user` (
  `id` varchar(255) NOT NULL COMMENT '用户编号',
  `name` varchar(255) DEFAULT NULL COMMENT '用户名',
  `gender` smallint(6) DEFAULT NULL COMMENT '用户性别: 0(男),1(女)',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='go语言操作Mysql数据库练习';
```

实体结构体:

```go
type User struct {
   id string
   name string
   password string
   gender int16
   createTime time.Time
   updateTime time.Time
}
```

### 2.2.1 查询:

**单行查询:**

单行查询`db.QueryRow()`执行一次查询，并期望返回最多一行结果（即Row）。`QueryRow`总是返回非nil的值，直到返回值(Row)的Scan方法被调用时，才会返回被延迟的错误。（如：未找到结果）

```go
func (db *DB) QueryRow(query string, args ...interface{}) *Row
```

```go
// 查询单条数据示例
func queryRowDemo() {
	sqlStr := "select id, name, age from user where id=?"
	var u user
	// 非常重要：确保QueryRow之后调用Scan方法，否则持有的数据库链接不会被释放
	err := db.QueryRow(sqlStr, 1).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Printf("scan failed, err:%v\n", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
}
```

说明:

1. 当查询完毕之后,一定要调用 返回值Row的Scan方法,把数据读取出来,否则会一直持有数据库连接
2. `Scan函数`中接受参数是为了给参数赋值, 所以是要改变参数的,应该传指针进去
3. go语言中 `mysql的timestamp类型`自动被转换成了`[]int8`,如果要转换成`time.Time`,就需要在连接数据库的时候将带上`parseTime=true`

**多行查询:**

```go
func selectMore(sqlStr string,param... string) (res *sql.Rows) {
   res = &sql.Rows{}
   rows, err := db.Query(sqlStr,param[0])
   if err != nil {
      fmt.Println("sleect more error:",err)
   }
   res = rows
    
   	for rows.Next() {
		var resUser User
		err := more.Scan(&resUser.Id, &resUser.Name, &resUser.Password, &resUser.Gender, &resUser.CreateTime, &resUser.UpdateTime)
		if err != nil {
			fmt.Println("读取数据失败:",err)
		}
	}
   // 一定要关闭掉rows对象,不然不能释放连接,如果在方法之外处理rows的话可以在处理之后关闭
   defer func(rows *sql.Rows) {
      err := rows.Close()
      if err != nil {
         fmt.Println("close rows fialed: ",err)
      }
   }(rows)
   return
}
```

### 2.2.2 插入数据

插入、更新和删除操作都使用`Exec`方法。

```go
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
```

Exec执行一次命令（包括查询、删除、更新、插入等），返回的Result是对已执行的SQL命令的总结。参数args表示query中的占位参数。

具体插入数据示例代码如下：

```go
// 插入数据
func insertRowDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "王五", 38)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	theID, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Printf("get lastinsert ID failed, err:%v\n", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", theID)
}
```

### 2.2.3 更新数据

具体更新数据示例代码如下：

```go
// 更新数据
func updateRowDemo() {
	sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 39, 3)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("update success, affected rows:%d\n", n)
}
```

### 2.2.4 删除数据

具体删除数据的示例代码如下：

```go
// 删除数据
func deleteRowDemo() {
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 3)
	if err != nil {
		fmt.Printf("delete failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("delete success, affected rows:%d\n", n)
}
```

## 2.3 预编译:

`Mysql`可以将`sql语句`先进行一个记录,使用`?`作为占位符占据某个数据的位置,然后当真正查询的时候将数据填入.

`database/sql`中使用下面的`Prepare`方法来实现预处理操作。

```go
func (db *DB) Prepare(query string) (*Stmt, error)
```

`Prepare`方法会先将`sql`语句发送给`MySQL`服务端，返回一个准备好的状态用于之后的查询和命令。返回值可以同时执行多个查询和命令。

**查询预处理:**

```go
// 预处理查询示例
func prepareQueryDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
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
```

**修改预处理:**

```go
// 预处理插入示例
func prepareInsertDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
	_, err = stmt.Exec("小王子", 18)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	_, err = stmt.Exec("沙河娜扎", 18)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	fmt.Println("insert success.")
}
```

## 2.4 MySQL事务:

### 2.4.1 相关方法:

开始事务

```go
func (db *DB) Begin() (*Tx, error)
```

提交事务

```go
func (tx *Tx) Commit() error
```

回滚事务

```go
func (tx *Tx) Rollback() error
```

### 2.4.2 示例:

```go
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
   sqlStr1 := "Update t_go_study_user set name = '张三' where id=?"
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

   sqlStr2 := "Update t_go_study_user set name = '张三' where id=?"
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
```

## 2.5 三方库sqlx:

在项目中我们通常可能会使用`database/sql`连接`MySQL数据库`。`sqlx`可以认为是Go语言内置`database/sql`的超集，它在优秀的内置`database/sql`基础上提供了一组扩展。这些扩展中除了大家常用来查询的`Get(dest interface{}, ...) error`和`Select(dest interface{}, ...) error`外还有很多其他强大的功能。

### 2.5.1 依赖:

```go
go get github.com/jmoiron/sqlx
```

### 2.5.2 基本使用:

#### 1. 连接:

```go
var db *sqlx.DB

func initDB() (err error) {
	dsn := "user:password@tcp(127.0.0.1:3306)/sql_test?charset=utf8mb4&parseTime=True"
	// 也可以使用MustConnect连接不成功就panic
	db, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		fmt.Printf("connect DB failed, err:%v\n", err)
		return
	}
	db.SetMaxOpenConns(20)
	db.SetMaxIdleConns(10)
	return
}
```

#### 2. 查询:

```go
// 查询单条数据示例
func queryRowDemo() {
   sqlStr := "select id, name, gender from t_go_study_user where id=?"
   var u User
   err := sqlxDB.Get(&u, sqlStr, 1)
   if err != nil {
      fmt.Printf("get failed, err:%v\n", err)
      return
   }
   fmt.Printf("id:%d name:%s age:%d\n", u.Id, u.Name, u.Gender)
}

// 查询多条数据示例
func queryMultiRowDemo() {
   sqlStr := "select id, name, gender from t_go_study_user where id=?"
   var users []User
   err := sqlxDB.Select(&users, sqlStr, 0)
   if err != nil {
      fmt.Printf("query failed, err:%v\n", err)
      return
   }
   fmt.Printf("users:%#v\n", users)
}
```

#### 3. 修改操作:

```go
// 插入数据
func insertRowDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "沙河小王子", 19)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	theID, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Printf("get lastinsert ID failed, err:%v\n", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", theID)
}

// 更新数据
func updateRowDemo() {
	sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 39, 6)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("update success, affected rows:%d\n", n)
}

// 删除数据
func deleteRowDemo() {
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 6)
	if err != nil {
		fmt.Printf("delete failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("delete success, affected rows:%d\n", n)
}
```

#### 4. 名称绑定:

`DB.NamedExec`方法用来绑定SQL语句与结构体或map中的同名字段。

```go
func insertUserDemo()(err error){
	sqlStr := "INSERT INTO user (name,age) VALUES (:name,:age)"
	_, err = db.NamedExec(sqlStr,
		map[string]interface{}{
			"name": "七米",
			"age": 28,
		})
	return
}
```

与`DB.NamedExec`同理，这里是支持查询。

```go
func namedQuery(){
	sqlStr := "SELECT * FROM user WHERE name=:name"
	// 使用map做命名查询
	rows, err := db.NamedQuery(sqlStr, map[string]interface{}{"name": "七米"})
	if err != nil {
		fmt.Printf("db.NamedQuery failed, err:%v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next(){
		var u user
		err := rows.StructScan(&u)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			continue
		}
		fmt.Printf("user:%#v\n", u)
	}

	u := user{
		Name: "七米",
	}
	// 使用结构体命名查询，根据结构体字段的 db tag进行映射
	rows, err = db.NamedQuery(sqlStr, u)
	if err != nil {
		fmt.Printf("db.NamedQuery failed, err:%v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next(){
		var u user
		err := rows.StructScan(&u)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			continue
		}
		fmt.Printf("user:%#v\n", u)
	}
}
```

#### 5. 事务操作:

对于事务操作，我们可以使用`sqlx`中提供的`db.Beginx()`和`tx.Exec()`方法。示例代码如下：

```go
func transactionDemo2()(err error) {
	tx, err := db.Beginx() // 开启事务
	if err != nil {
		fmt.Printf("begin trans failed, err:%v\n", err)
		return err
	}
	defer func() {
		if p := recover(); p != nil {
			tx.Rollback()
			panic(p) // re-throw panic after Rollback
		} else if err != nil {
			fmt.Println("rollback")
			tx.Rollback() // err is non-nil; don't change it
		} else {
			err = tx.Commit() // err is nil; if Commit returns error update err
			fmt.Println("commit")
		}
	}()

	sqlStr1 := "Update user set age=20 where id=?"

	rs, err := tx.Exec(sqlStr1, 1)
	if err!= nil{
		return err
	}
	n, err := rs.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr1 failed")
	}
	sqlStr2 := "Update user set age=50 where i=?"
	rs, err = tx.Exec(sqlStr2, 5)
	if err!=nil{
		return err
	}
	n, err = rs.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr1 failed")
	}
	return err
}
```

#### 6. 批量操作:

`sqlx.In`是`sqlx`提供的一个非常方便的函数。

**sqlx.In的批量插入示例**

`表结构`

为了方便演示插入数据操作，这里创建一个`user`表，表结构如下：

```sql
CREATE TABLE `user` (
    `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(20) DEFAULT '',
    `age` INT(11) DEFAULT '0',
    PRIMARY KEY(`id`)
)ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;
```

`结构体`

定义一个`user`结构体，字段通过tag与数据库中user表的列一致。

```go
type User struct {
	Name string `db:"name"`
	Age  int    `db:"age"`
}
```

`bindvars（绑定变量）`

查询占位符`?`在内部称为**bindvars（查询占位符）**,它非常重要。你应该始终使用它们向数据库发送值，因为它们可以防止SQL注入攻击。`database/sql`不尝试对查询文本进行任何验证；它与编码的参数一起按原样发送到服务器。除非驱动程序实现一个特殊的接口，否则在执行之前，查询是在服务器上准备的。因此`bindvars`是特定于数据库的:

- MySQL中使用`?`
- PostgreSQL使用枚举的`$1`、`$2`等bindvar语法
- SQLite中`?`和`$1`的语法都支持
- Oracle中使用`:name`的语法

`bindvars`的一个常见误解是，它们用来在sql语句中插入值。它们其实仅用于参数化，不允许更改SQL语句的结构。例如，使用`bindvars`尝试参数化列或表名将不起作用：

```go
// ？不能用来插入表名（做SQL语句中表名的占位符）
db.Query("SELECT * FROM ?", "mytable")
 
// ？也不能用来插入列名（做SQL语句中列名的占位符）
db.Query("SELECT ?, ? FROM people", "name", "location")
```

**自己拼接语句实现批量插入**

比较笨，但是很好理解。就是有多少个User就拼接多少个`(?, ?)`。

```go
// BatchInsertUsers 自行构造批量插入的语句
func BatchInsertUsers(users []*User) error {
	// 存放 (?, ?) 的slice
	valueStrings := make([]string, 0, len(users))
	// 存放values的slice
	valueArgs := make([]interface{}, 0, len(users) * 2)
	// 遍历users准备相关数据
	for _, u := range users {
		// 此处占位符要与插入值的个数对应
		valueStrings = append(valueStrings, "(?, ?)")
		valueArgs = append(valueArgs, u.Name)
		valueArgs = append(valueArgs, u.Age)
	}
	// 自行拼接要执行的具体语句
	stmt := fmt.Sprintf("INSERT INTO user (name, age) VALUES %s",
		strings.Join(valueStrings, ","))
	_, err := DB.Exec(stmt, valueArgs...)
	return err
}
```

**使用sqlx.In实现批量插入**

前提是需要我们的结构体实现`driver.Valuer`接口：

```go
func (u User) Value() (driver.Value, error) {
	return []interface{}{u.Name, u.Age}, nil
}
```

使用`sqlx.In`实现批量插入代码如下：

```go
// BatchInsertUsers2 使用sqlx.In帮我们拼接语句和参数, 注意传入的参数是[]interface{}
func BatchInsertUsers2(users []interface{}) error {
	query, args, _ := sqlx.In(
		"INSERT INTO user (name, age) VALUES (?), (?), (?)",
		users..., // 如果arg实现了 driver.Valuer, sqlx.In 会通过调用 Value()来展开它
	)
	fmt.Println(query) // 查看生成的querystring
	fmt.Println(args)  // 查看生成的args
	_, err := DB.Exec(query, args...)
	return err
}
```

**使用NamedExec实现批量插入**

**注意** ：该功能需1.3.1版本以上，并且1.3.1版本目前还有点问题，sql语句最后不能有空格和`;`，详见[issues/690](https://github.com/jmoiron/sqlx/issues/690)。

使用`NamedExec`实现批量插入的代码如下：

```go
// BatchInsertUsers3 使用NamedExec实现批量插入
func BatchInsertUsers3(users []*User) error {
	_, err := DB.NamedExec("INSERT INTO user (name, age) VALUES (:name, :age)", users)
	return err
}
```

把上面三种方法综合起来试一下：

```go
func main() {
	err := initDB()
	if err != nil {
		panic(err)
	}
	defer DB.Close()
	u1 := User{Name: "七米", Age: 18}
	u2 := User{Name: "q1mi", Age: 28}
	u3 := User{Name: "小王子", Age: 38}

	// 方法1
	users := []*User{&u1, &u2, &u3}
	err = BatchInsertUsers(users)
	if err != nil {
		fmt.Printf("BatchInsertUsers failed, err:%v\n", err)
	}

	// 方法2
	users2 := []interface{}{u1, u2, u3}
	err = BatchInsertUsers2(users2)
	if err != nil {
		fmt.Printf("BatchInsertUsers2 failed, err:%v\n", err)
	}

	// 方法3
	users3 := []*User{&u1, &u2, &u3}
	err = BatchInsertUsers3(users3)
	if err != nil {
		fmt.Printf("BatchInsertUsers3 failed, err:%v\n", err)
	}
}
```

**sqlx.In的查询示例**

关于`sqlx.In`这里再补充一个用法，在`sqlx`查询语句中实现In查询和FIND_IN_SET函数。即实现`SELECT * FROM user WHERE id in (3, 2, 1);`和`SELECT * FROM user WHERE id in (3, 2, 1) ORDER BY FIND_IN_SET(id, '3,2,1');`。

in查询

查询id在给定id集合中的数据。

```go
// QueryByIDs 根据给定ID查询
func QueryByIDs(ids []int)(users []User, err error){
	// 动态填充id
	query, args, err := sqlx.In("SELECT name, age FROM user WHERE id IN (?)", ids)
	if err != nil {
		return
	}
	// sqlx.In 返回带 `?` bindvar的查询语句, 我们使用Rebind()重新绑定它
	query = DB.Rebind(query)

	err = DB.Select(&users, query, args...)
	return
}
```

in查询和FIND_IN_SET函数

查询id在给定id集合的数据并维持给定id集合的顺序。

```go
// QueryAndOrderByIDs 按照指定id查询并维护顺序
func QueryAndOrderByIDs(ids []int)(users []User, err error){
	// 动态填充id
	strIDs := make([]string, 0, len(ids))
	for _, id := range ids {
		strIDs = append(strIDs, fmt.Sprintf("%d", id))
	}
	query, args, err := sqlx.In("SELECT name, age FROM user WHERE id IN (?) ORDER BY FIND_IN_SET(id, ?)", ids, strings.Join(strIDs, ","))
	if err != nil {
		return
	}

	// sqlx.In 返回带 `?` bindvar的查询语句, 我们使用Rebind()重新绑定它
	query = DB.Rebind(query)

	err = DB.Select(&users, query, args...)
	return
}
```

当然，在这个例子里面你也可以先使用`IN`查询，然后通过代码按给定的ids对查询结果进行排序。

## 2.6 GORM:

gorm是类似与java `mybatis`的一个ORM框架.

[官网地址](https://gorm.io/zh_CN/docs/)

###　2.6.1 特性:

- 全功能 ORM
- 关联 (Has One，Has Many，Belongs To，Many To Many，多态，单表继承)
- Create，Save，Update，Delete，Find 中钩子方法
- 支持 `Preload`、`Joins` 的预加载
- 事务，嵌套事务，Save Point，Rollback To Saved Point
- Context、预编译模式、DryRun 模式
- 批量插入，FindInBatches，Find/Create with Map，使用 SQL 表达式、Context Valuer 进行 CRUD
- SQL 构建器，Upsert，数据库锁，Optimizer/Index/Comment Hint，命名参数，子查询
- 复合主键，索引，约束
- Auto Migration
- 自定义 Logger
- 灵活的可扩展插件 API：Database Resolver（多数据库，读写分离）、Prometheus…

### 2.6.2 安装:

```go
go get -u gorm.io/gorm
go get -u gorm.io/driver/mysql // 数据库连接驱动依赖可以任意选择
```

### 2.6.3  模型定义:

#### 1. gorm的一些约定:

- 默认情况下，GORM 使用 `ID` 作为主键，使用结构体名的 `蛇形复数` 作为表名，字段名的 `蛇形` 作为列名，并使用 `CreatedAt`、`UpdatedAt` 字段追踪创建、更新时间. 将这个结构体(`gorm.Model`)嵌入到自定义的结构体, 即可实现这一约定.

  ```go
  // gorm.Model 的定义
  type Model struct {
    ID        uint           `gorm:"primaryKey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
  }
  ```

- 自定义约定:

  **表名:** gorm 默认使用结构体的`蛇形复数` 形式作为表名, 即 `结构体User`的表名为 `users`.

  - 固定表名:

    通过实现`Tabler接口`来更改默认的表名.

    ```go
    type Tabler interface {
        TableName() string
    }
    
    // TableName 会将 User 的表名重写为 `profiles`
    func (User) TableName() string {
      return "profiles"
    }
    ```

  - 动态表名:

    > **注意：** `TableName` 不支持动态变化，它会被缓存下来以便后续使用。

    想要使用动态表名，可以使用 `Scopes`，例如：

    ```go
    func UserTable(user User) func (tx *gorm.DB) *gorm.DB {
      return func (tx *gorm.DB) *gorm.DB {
        if user.Admin {
          return tx.Table("admin_users")
        }
    
        return tx.Table("users")
      }
    }
    // 动态表名
    db.Scopes(UserTable(user)).Create(&user)
    ```

  - 临时表名: `主要是用于实现子查询`

    您可以使用 `Table` 方法临时指定表名，例如：

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

  **命名策略**: 

  GORM 允许用户通过覆盖默认的`命名策略`更改默认的命名约定，命名策略被用于构建： `TableName`、`ColumnName`、`JoinTableName`、`RelationshipFKName`、`CheckerName`、`IndexName`。

  默认规则: 以列名为例子

  ```go
  type User struct {
    ID        uint      // 列名是 `id`
    Name      string    // 列名是 `name`
    Birthday  time.Time // 列名是 `birthday`
    CreatedAt time.Time // 列名是 `created_at`
  }
  ```

  修改方法: 可以使用 `column` 标签或 [`命名策略`](https://gorm.io/zh_CN/docs/conventions.html#naming_strategy) 来覆盖列名, 下面展示的就是`tag`的方式

  ```go
  type Animal struct {
    AnimalID int64     `gorm:"column:beast_id"`         // 将列名设为 `beast_id`
    Birthday time.Time `gorm:"column:day_of_the_beast"` // 将列名设为 `day_of_the_beast`
    Age      int64     `gorm:"column:age_of_the_beast"` // 将列名设为 `age_of_the_beast`
  }
  ```

  **模型相关时间字段**:

  `CreateAt`:

  对于有 `CreatedAt` 字段的模型，创建记录时，如果该字段值为零值，则将该字段的值设为当前时间

  ```go
  db.Create(&user) // 将 `CreatedAt` 设为当前时间
  
  user2 := User{Name: "jinzhu", CreatedAt: time.Now()}
  db.Create(&user2) // user2 的 `CreatedAt` 不会被修改
  
  // 想要修改该值，您可以使用 `Update`
  db.Model(&user).Update("CreatedAt", time.Now())
  ```

  `updateAt`:

  对于有 `UpdatedAt` 字段的模型，更新记录时，将该字段的值设为当前时间。创建记录时，如果该字段值为零值，则将该字段的值设为当前时间

  ```go
  db.Save(&user) // 将 `UpdatedAt` 设为当前时间
  
  db.Model(&user).Update("name", "jinzhu") // 会将 `UpdatedAt` 设为当前时间
  
  db.Model(&user).UpdateColumn("name", "jinzhu") // `UpdatedAt` 不会被修改
  
  user2 := User{Name: "jinzhu", UpdatedAt: time.Now()}
  db.Create(&user2) // 创建记录时，user2 的 `UpdatedAt` 不会被修改
  
  user3 := User{Name: "jinzhu", UpdatedAt: time.Now()}
  db.Save(&user3) // 更新世，user3 的 `UpdatedAt` 会修改为当前时间
  ```

  **注意** GORM 支持拥有多种类型的时间追踪字段。可以根据 UNIX（毫/纳）秒.  `autoCreateTime`、`autoUpdateTime` 标签就是用来自动填充时间的

  ```go
  type User struct {
    CreatedAt time.Time // 在创建时，如果该字段值为零值，则使用当前时间填充
    UpdatedAt int       // 在创建时该字段值为零值或者在更新时，使用当前时间戳秒数填充
    Updated   int64 `gorm:"autoUpdateTime:nano"` // 使用时间戳填纳秒数充更新时间
    Updated   int64 `gorm:"autoUpdateTime:milli"` // 使用时间戳毫秒数填充更新时间
    Created   int64 `gorm:"autoCreateTime"`      // 使用时间戳秒数填充创建时间
  }
  ```

#### 2. model嵌入结构体:

- `通过继承的方式, 嵌入model`:

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

- `通过tag嵌入model: 对于正常的结构体字段，可以通过标签 `embedded` 将其嵌入, 并且可以使用标签 `embeddedPrefix` 来为 db 中的字段名添加前缀，例如：

  ```go
  type Author struct {
      Name  string
      Email string
  }
  type Blog struct {
    ID      int
    Author  Author `gorm:"embedded;embeddedPrefix:author_"`
    Upvotes int32
  }
  // 等效于
  type Blog struct {
    ID          int64
    AuthorName  string
    AuthorEmail string
    Upvotes     int32
  }
  ```

#### 3. 字段标签:

声明 model 时，tag 是可选的，GORM 支持以下 tag： tag 名大小写不敏感，但建议使用 `camelCase` 风格

| 标签名                 | 说明                                                         |
| :--------------------- | :----------------------------------------------------------- |
| column                 | 指定 db 列名                                                 |
| type                   | `列数据库数据类型`，推荐使用兼容性好的通用类型，例如：所有数据库都支持 bool、int、uint、float、string、time、bytes 并且可以和其他标签一起使用，例如：`not null`、`size`, `autoIncrement`… 像 `varbinary(8)` 这样指定数据库数据类型也是支持的。在使用指定数据库数据类型时，它需要是完整的数据库数据类型，如：`MEDIUMINT UNSIGNED not NULL AUTO_INCREMENT` |
| size                   | 指定列大小，例如：`size:256`                                 |
| primaryKey             | 指定列为主键                                                 |
| unique                 | 指定列为唯一                                                 |
| default                | 指定列的默认值                                               |
| precision              | 指定列的精度                                                 |
| scale                  | 指定列大小                                                   |
| not null               | 指定列为 NOT NULL                                            |
| autoIncrement          | 指定列为自动增长                                             |
| autoIncrementIncrement | 自动步长，控制连续记录之间的间隔                             |
| embedded               | 嵌套字段                                                     |
| embeddedPrefix         | 嵌入字段的列名前缀                                           |
| autoCreateTime         | 创建时追踪当前时间，对于 `int` 字段，它会追踪秒级时间戳，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoCreateTime:nano` |
| autoUpdateTime         | 创建/更新时追踪当前时间，对于 `int` 字段，它会追踪秒级时间戳，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoUpdateTime:milli` |
| index                  | 根据参数创建索引，多个字段使用相同的名称则创建复合索引，查看 [索引](https://gorm.io/zh_CN/docs/indexes.html) 获取详情 |
| uniqueIndex            | 与 `index` 相同，但创建的是唯一索引                          |
| check                  | 创建检查约束，例如 `check:age > 13`，查看 [约束](https://gorm.io/zh_CN/docs/constraints.html) 获取详情 |
| <-                     | 设置字段写入的权限， `<-:create` 只创建、`<-:update` 只更新、`<-:false` 无写入权限、`<-` 创建和更新权限 |
| ->                     | 设置字段读的权限，`->:false` 无读权限                        |
| -                      | 忽略该字段，`-` 无读写权限                                   |
| comment                | 迁移时为字段添加注释                                         |

#### 4. 模型的高级选项:

gorm支持在模型中对字段配置`crud操作`的权限. `写是包含create和update两个权限`

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
}
```

> **注意：** 使用 GORM Migrator 创建表时，不会创建被忽略的字段



