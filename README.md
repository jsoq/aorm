# Aorm
Operate Database So Easy For GoLang Developer

[How to use](#how-to-use)   
- [Define data struct](#define-data-struct)   
- [Connect database](#connect-database)   
- [Migrate](#migrate)
- [Basic CRUD](#basic-crud)   
  - [Insert one record](#insert-one-record)
  - [Find one record](#find-one-record)
  - [Select many record](#select-many-record)
  - [Update record](#update-record)
  - [Delete record](#delete-record)
- [Advanced Query](#advanced-query)
  - [Table](#table)
  - [Field](#field)
  - [Where](#where)
  - [Where Opts](#where-opts)
  - [JOIN](#join)
  - [GROUP](#group)
  - [Having](#having)
  - [Order](#order)
  - [Limit and Page](#limit-and-page)

## How to use

### Define data struct
```go
    type Person struct {
        Id          null.Int    `aorm:"primary;auto_increment;type:bigint" json:"id"`
        Name        null.String `aorm:"size:100;not null;comment:名字" json:"name"`
        Sex         null.Bool   `aorm:"index;comment:性别" json:"sex"`
        Age         null.Int    `aorm:"index;comment:年龄" json:"age"`
        Type        null.Int    `aorm:"index;comment:类型" json:"type"`
        CreateTime  null.Time   `aorm:"comment:创建时间" json:"createTime"`
        Money       null.Float  `aorm:"comment:金额" json:"money"`
        ArticleBody null.String `aorm:"type:text;comment:文章内容" json:"articleBody"`
    }
```

### Connect database
```go
    //connect
    db, err := sql.Open("mysql", "root:root@tcp(localhost:3306)/database_name?charset=utf8mb4&parseTime=True&loc=Local")
    if err != nil {
        panic(err)
    }
    defer db.Close()
    
    //ping test
    err1 := db.Ping()
    if err1 != nil {
        panic(err1)
    }
```

### Migrate
by `AutoMigrate` function, the table name will be `person`, underline style string with the struct name
```go
    aorm.Use(db).Opinion("ENGINE", "InnoDB").Opinion("COMMENT", "用户表").AutoMigrate(&Person{})
```
by `Migrate` function, You can also use other table name
```go
    aorm.Use(db).Opinion("ENGINE", "InnoDB").Opinion("COMMENT", "用户表").Migrate("person_1", &Person{})
```
by `ShowCreateTable` function, You can get the create table sql
```go
    showCreate := aorm.Use(db).ShowCreateTable("person")
    fmt.Println(showCreate)
```
like this
```sql
    CREATE TABLE `person` (                                                   
        `id` bigint(20) NOT NULL AUTO_INCREMENT,                                
        `name` varchar(100) NOT NULL COMMENT '名字',                            
        `sex` tinyint(4) DEFAULT NULL COMMENT '性别',                           
        `create_time` datetime DEFAULT NULL COMMENT '创建时间',                 
        `money` float DEFAULT NULL COMMENT '金额',                              
        `article_body` text COMMENT '文章内容',                                 
        `type` int(11) DEFAULT NULL COMMENT '类型',                             
        `age` int(11) DEFAULT NULL COMMENT '年龄',                              
        PRIMARY KEY (`id`),                                                     
        KEY `idx_person_sex` (`sex`),                                           
        KEY `idx_person_type` (`type`),                                         
        KEY `idx_person_age` (`age`)                                            
    ) ENGINE=InnoDB AUTO_INCREMENT=59 DEFAULT CHARSET=utf8mb4 COMMENT='用户表'
```

### Basic CRUD

#### Insert one record
```go
    id, errInsert := aorm.Use(db).Debug(true).Insert(&Person{
        Name:        null.StringFrom("alice"),
        Sex:         null.BoolFrom(false),
        Age:         null.IntFrom(18),
        Type:        null.IntFrom(0),
        CreateTime:  null.TimeFrom(time.Now()),
        Money:       null.FloatFrom(100.15),
        ArticleBody: null.StringFrom("this is a text body"),
    })
    if errInsert != nil {
        fmt.Println(errInsert)
    }
    fmt.Println(id) 
```
then get the sql and params like this
```sql
    INSERT INTO person (name,sex,age,type,create_time,money,article_body) VALUES (?,?,?,?,?,?,?)
    [alice false 18 0 2022-11-29 16:10:33.1677507 +0800 CST m=+0.008134801 100.15 this is a text body]
```

#### Find one record

```go
    var person Person
    errFind := aorm.Use(db).Debug(true).Where(&Person{Id: null.IntFrom(id)}).Find(&person)
    if errFind != nil {
        fmt.Println(errFind)
    }
    fmt.Println(person)
```
then get the sql and params like this
```sql
    SELECT * FROM person WHERE id = ? Limit ?,?                                                                                                           
    [60 0 1]
```

#### Select many record
```go
    var list []Person
    errSelect := aorm.Use(db).Debug(true).Where(&Person{Type: null.IntFrom(0)}).Select(&list)
    if errSelect != nil {
        fmt.Println(errSelect)
    }
    for i := 0; i < len(list); i++ {
        fmt.Println(list[i])
    }
```
then get the sql and params like this
```sql
    SELECT * FROM person WHERE type = ?
    [0]
```

#### Update record

```go
    countUpdate,errUpdate := aorm.Use(db).Where(&Person{Id: null.IntFrom(id)}).Update(&Person{ArticleBody: null.StringFrom("this is test")})
    if errUpdate != nil {
        fmt.Println(errUpdate)
    }
    fmt.Println(countUpdate)
```
then get the sql and params like this
```sql
    UPDATE person SET article_body=? WHERE id = ?
    [this is test 60]
```

#### Delete record

```go
    countDelete,errDelete := aorm.Use(db).Where(&Person{Id: null.IntFrom(id)}).delete()
    if errDelete != nil {
        fmt.Println(errDelete)
    }
    fmt.Println(countDelete)
```
then get the sql and params like this
```sql
    DELETE FROM person WHERE id = ?
    [60]
```

### Advanced Query
#### Table
by `Table` function, you can set table name easy
```go
    aorm.Use(db).Debug(true).Table("person_1").Insert(&Person{Name:null.StringFrom("test name")})
```
then get the sql and params like this
```sql
    INSERT INTO person_1 (name) VALUES (?)
    [test name]
```
#### Field
by `Field` function, you can set field name easy
```go
    var listByFiled []Person
    aorm.Use(db).Debug(true).Field("name,age").Where(&Person{Age:null.IntFrom(18)}).Select(&listByFiled)
```
then get the sql and params like this
```sql
    SELECT name,age FROM person WHERE age = ?
    [18]
```
#### Where
```go
    var listByWhere []Person
    
    var where []aorm.WhereItem
    where = append(where, aorm.WhereItem{Field: "type", Opt: aorm.Eq, Val: 0})
    where = append(where, aorm.WhereItem{Field: "age", Opt: aorm.In, Val: []int{18, 20}})
    where = append(where, aorm.WhereItem{Field: "money", Opt: aorm.Between, Val: []float64{100.1, 200.9}})
    where = append(where, aorm.WhereItem{Field: "article_body", Opt: aorm.Like, Val: []string{"%", "is", "%"}})
	
    aorm.Use(db).Debug(true).Table("person").WhereArr(where).Select(&listByWhere)
    for i := 0; i < len(listByWhere); i++ {
        fmt.Println(listByWhere[i])
    }
```
then get the sql and params like this
```sql
    SELECT * FROM person WHERE type = ? AND age IN (?,?) AND money BETWEEN (?) AND (?) AND article_body LIKE concat('%',?,'%')                            
    [0 18 20 100.1 200.9 is]
```
#### Where Opts
`aorm.Eq` same as `=`  
`aorm.Ne` same as `!=`  
`aorm.Gt` same as `>`   
`aorm.Ge` same as `>=`   
`aorm.Lt` same as `<`  
`aorm.Le` same as `<=`  

`aorm.In` same as `IN`  
`aorm.NotIn` same as `NOT IN`  
`aorm.Like` same as `LIKE`   
`aorm.NotLike` same as `NOT LIKE`  
`aorm.Between` same as `BETWEEN`  
`aorm.NotBetween` same as `NOT BETWEEN`
#### JOIN
```go
    var list2 []PersonVO
    var where []aorm.WhereItem
    where = append(where, aorm.WhereItem{Field: "o.type", Opt: aorm.Eq, Val: 0})
    where = append(where, aorm.WhereItem{Field: "o.age", Opt: aorm.In, Val: []int{18, 20}})
    aorm.Use(db).Debug(true).
        Table("person p").
        LeftJoin("user u","u.user_id=p.user_id").
        Field("o.*").
        Field("u.user_name").
        WhereArr(where).
        Select(&list2)
```
then get the sql and params like this
```sql
    SELECT o.*,u.user_name FROM person p LEFT JOIN user u ON u.user_id=p.user_id WHERE o.type = ? AND o.age IN (?,?)                            
    [0 18 20]
```
some other join function like this `RightJoin`, `Join`
#### Group
```go
    type PersonAge struct {
        Age         null.Int
        AgeCount    null.Int
    }

    var personAge PersonAge
    var where []aorm.WhereItem
    where = append(where, aorm.WhereItem{Field: "type", Opt: aorm.Eq, Val: 0})
    aorm.Use(db).Debug(true).
        Table("person").
        Field("age").
        Field("count(age) as age_count").
        Group("age").
        Where(where)
        Find(&personAge)
```
then get the sql and params like this
```sql
    SELECT age,count(age) as age_count FROM person WHERE type = ? GROUP BY age                      
    [0]
```
#### Having
```go
    type PersonAge struct {
        Age         null.Int
        AgeCount    null.Int
    }

    var listByHaving []PersonAge
    var where []aorm.WhereItem
    where = append(where, aorm.WhereItem{Field: "type", Opt: aorm.Eq, Val: 0})
    aorm.Use(db).Debug(true).
        Table("person").
        Field("age").
        Field("count(age) as age_count").
        Group("age").
        Where(where)
        Select(&listByHaving)
```
then get the sql and params like this
```sql
    SELECT age,count(age) as age_count FROM person WHERE type = ? GROUP BY age                      
    [0]
```
#### Limit and Page
```go
    var list3 []Person
    var where []aorm.WhereItem
    where = append(where, aorm.WhereItem{Field: "type", Opt: aorm.Eq, Val: 0})
    aorm.Use(db).Debug(true).
        Table("person").
        Where(where).
        Limit(50,10)
        Select(&list3)
```
then get the sql and params like this
```sql
    SELECT * FROM person WHERE type = ? LIMIT ?,?                     
    [0, 50, 10]
```
```go
    var list4 []Person
    var where []aorm.WhereItem
    where = append(where, aorm.WhereItem{Field: "type", Opt: aorm.Eq, Val: 0})
    aorm.Use(db).Debug(true).
        Table("person").
        Where(where).
        Page(3,10)
        Select(&list4)
```
then get the sql and params like this
```sql
    SELECT * FROM person WHERE type = ? LIMIT ?,?                     
    [0, 20, 10]
```
