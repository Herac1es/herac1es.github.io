# 使用GORM IN查询的一个问题


## 问题背景

这是一个我在开发过程中遇到的小问题，为了简单，现假设有一张 `user` 表，结构如下：

```sql
CREATE TABLE `user` (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(20) NOT NULL DEFAULT '' COMMENT '用户名',
  `age` tinyint(4) unsigned NOT NULL DEFAULT '0' COMMENT '年龄',
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb4;
```

对应的 GORM Model：

```go
type User struct {
	ID   int    `gorm:"primaryKey;column:id"`
	Name string `gorm:"column:name;varchar(20);not null;default"`
	Age  uint8  `gorm:"column:age;tinyint unsigned;not null;default"`
}

func (u User) TableName() string {
	return "user"
}
```

在 MySQL 中年龄 `age` 被定义为一个 `tinyint unsigned` 类型，范围是 0-255 ，对应 Go 代码中定义的类型为 `uint8` 。

插入一条测试数据:

```sql
insert into `user` (`name`,`age`) values('heracles','23');

select * from `user` where `age` IN ('23')\G;

```

得到现在表中的实际数据：

```shell
*************************** 1. row ***************************
  id: 4
name: 马力神
 age: 23
*************************** 2. row ***************************
  id: 5
name: heracles
 age: 23
2 rows in set (0.01 sec)
```

然后使用 GORM (V1) 构建查询语句，本意是使用 IN 语句并指定一个 User.Age 类型的集合来查询年龄符合的行:

```go
func main() {
  // 连接数据库
	db, err := gorm.Open("mysql", "root:520168@/test?charset=utf8mb4&parseTime=True&loc=Local")
	if err != nil {
		panic(err)
	}
	defer db.Close()
  
	user := make([]User, 0)
  // 开启 debug 日志
	if err := db.Debug().Where("`age` IN (?)", []uint8{23}).Find(&user).Error; err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
	fmt.Printf("%+v", user)
}
```

得到的结果是这样的:

```shell
[2021-04-19 15:38:42]  [1.54ms]  SELECT * FROM `user`  WHERE (`age`   ('<binary>'))  
[0 rows affected or returned ] 
[]
```

可以明显看到，首先生成的 SQL 语句是错乱的，自然拿不到预期结果。

然后试了下，将  `db.Debug().Where("\`age\` IN (?)", []uint8{23})` 中的切片改成 `[ ]uint`、`[ ]int` 等其他整型后，结果：

```shell
[2021-04-19 15:53:47]  [8.71ms]  SELECT * FROM `user`  WHERE (`age` IN (23))  
[2 rows affected or returned ] 
[{ID:4 Name:马力神 Age:23} {ID:5 Name:heracles Age:23}]
```

可以看到输出符合预期了，

## 为什么

当时在测试时看到这个情况，第一反应就是 GORM 构建 SQL 语句的某个部分，通过类型断言处理的逻辑和自己想象的不一样，于是开始 Debug , 追踪源码:

`Where()`中定义的查询条件会被保存在 `gorm/search.whereConditions`的一个 map 中：

```go

func (s *DB) Where(query interface{}, args ...interface{}) *DB {
   return s.clone().search.Where(query, args...).db
}

func (s *search) Where(query interface{}, values ...interface{}) *search {
	s.whereConditions = append(s.whereConditions, map[string]interface{}{"query": query, "args": values})
	return s
}

```

然后在 `Find()` 中会在 `queryCallback()` 查询回调中通过 `prepareQuerySQL()` 对查询条件进行构建：

```go
// queryCallback used to query data from database
func queryCallback(scope *Scope) {
  ...
  
  scope.prepareQuerySQL()
  
  ...
}
```

最终在 `CombinedConditionSql()` 中的 `whereSQL()` 对 whereConditions 中的键值对进行处理:

```go
func (scope *Scope) whereSQL() (sql string) {
	...
  
	for _, clause := range scope.Search.whereConditions {
		if sql := scope.buildCondition(clause, true); sql != "" {
			andConditions = append(andConditions, sql)
		}
	}

	...
}


func (scope *Scope) buildCondition(clause map[string]interface{}, include bool) (str string) {
	...

	replacements := []string{}
	args := clause["args"].([]interface{})
	for _, arg := range args {
		var err error
		switch reflect.ValueOf(arg).Kind() {
		case reflect.Slice: // For where("id in (?)", []int64{1,2})
			if scanner, ok := interface{}(arg).(driver.Valuer); ok {
				arg, err = scanner.Value()
				replacements = append(replacements, scope.AddToVars(arg))
			} else if b, ok := arg.([]byte); ok {
				replacements = append(replacements, scope.AddToVars(b))
			} else if as, ok := arg.([][]interface{}); ok {
						...
			} else if values := reflect.ValueOf(arg); values.Len() > 0 {
				var tempMarks []string
				for i := 0; i < values.Len(); i++ {
					tempMarks = append(tempMarks, scope.AddToVars(values.Index(i).Interface()))
				}
				replacements = append(replacements, strings.Join(tempMarks, ","))
			} else {
				replacements = append(replacements, scope.AddToVars(Expr("NULL")))
			}
        
	...
        
			}
		default:
			
	}

	return
}
```

在这里，程序按预期的进入了 `case reflect.Slice` 分支，由于 [ ]uint8 没有实现标准数据库驱动库的 `driver.Valuer` 接口，接着在下一步 `else if b, ok := arg.([]byte); ok` 这里命中，这个 [ ]uint8 的参数会被当成一个 `replacement`元素，后续在 `where(query, args...)` 中的 query 中对应的位置填充，可以说 uint8 类型的切片被当成一个字符串而不是多个 uint8 的元素集合。

而如果是非 `uint8` 类型的切片，会进入 `else if values := reflect.ValueOf(arg); values.Len() > 0 ` 分支，相应的处理是将切片的每一个元素当成一个属性值，用 `,`拼接起来再填充到 query 中的相应位置。

所以对于 `Where(\`age\` IN (?),[]int{23,26})` ，转换后的结果

```sql
select * from `user` where `age` IN ('23','26');
```

而对于 `Where(\`age\` IN (?),[]uint8{23,26})` ，转换后的结果

```sql
select * from `user` where `age` IN ('TBSUB');
```

[]uint8 对应的是 ascii 码，所以23和26分别对应不可见字符 'TB' 和 'sub'，可以用0-32以外的可见字符试一下，比如98和99，对应的结果sql语句就为:

```go
[2021-04-19 17:30:31]  [2.08ms]  SELECT * FROM `user`  WHERE (`age` IN ('bc')) 
```
至于 [ ]uint8 和 [ ]byte 等同，是因为在 Go 语言中的 `byte` 类型就是 `uint8` 的一个别名，见内建类型包 builtin :
```go
// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8
```

## 总结

1. 由于在 Go 语言中，uint8 和 byte 本质上完全等同，所以使用 [ ]uint8 命中类型断言后的 [ ]byte 类型。
2. 在 GORM 和 Go 标准 database/driver 库中，对于 [ ]byte 的处理其实和 string 类似，并不会按文中设想的将每一个 uint8 元素当成 IN 的匹配元素 (然后用英文 `,` 分隔)，而是将整个 [ ]byte 当成一个匹配元素。 
3. 为了节省内存而定义一些 uint8 类型的实体 model 属性，在使用 GORM 的 `Where()` 构建 IN 查询时，一定要转换成其他的整型切片类型。

