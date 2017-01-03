# QTL
QTL是一个访问SQL数据库的C++库，目前支持MySQL和SQLite。QTL是一个轻量级的库，只由头文件组成，不需要单独编译安装。QTL是对数据库原生客户端接口的薄封装，能提供友好使用方式的同时拥有接近于使用原生接口的性能。
使用QTL需要支持C++11的编译器。

## 使用方式

### 打开数据库

```C++
qtl::mysql::database db;
db.open("localhost", "root", "", "test");
```

### 执行查询


#### 1. 插入记录

```C++
uint64_t id=db.insert("insert into test(Name, CreateTime) values(?, now())", "test_user");
```

#### 2. 更新记录

```C++
db.execute_direct("update test set Name=? WHERE ID=?",  NULL, "other_user", id);
```
#### 3. 更新多条记录：

```C++
uint64_t affected=0;
auto stmt=db.open_command("insert into test(Name, CreateTime) values(?, now())");
qtl::execute(stmt, &affected, "second_user", "third_user");

```

#### 4. 查询数据，以回调函数方式处理数据
当回调函数返回false时，中止遍历数据。

```C++
db.query("select * from test where id=?",
	id, std::tuple<uint32_t, std::string, qtl::mysql::time>(),
	[](uint32_t id, const std::string& name, qtl::mysql::time& create_time) {
		printf("ID=\"%d\", Name=\"%s\"\n", id, name.data());
		return true;
});
```

#### 5. 也可以把数据绑定到结构上

```C++
struct TestMysqlRecord
{
	uint32_t id;
	char name[33];
	qtl::mysql::time create_time;

	TestMysqlRecord()
	{
		memset(this, 0, sizeof(TestMysqlRecord));
	}
};

namespace qtl
{
	template<>
	inline void bind_record<qtl::mysql::statement, TestMysqlRecord>(qtl::mysql::statement& command, TestMysqlRecord&& v)
	{
		qtl::bind_field(command, 0, v.id);
		qtl::bind_field(command, 1, v.name);
		qtl::bind_field(command, 2, v.create_time);
	}
}

db.query("select * from test where id=?",
	id, TestMysqlRecord(),
	[](TestMysqlRecord& record) {
		printf("ID=\"%d\", Name=\"%s\"\n", record.id, record.name);
		return true;
});
```
#### 6. 以迭代器方式访问数据

```C++
for(auto& record : db.result<TestMysqlRecord>("select * from test"))
{
	printf("ID=\"%d\", Name=\"%s\"\n", record.id, record.name);
}
```

## 有关MySQL的说明

访问MySQL时，包含头文件qtl_mysql.hpp。

### MySQL的参数数据绑定

| 参数类型 | C++类型 |
| ------- | ------ |
| tinyint | int8_t<br/>uint8_t |
| smallint | int16_t<br/>uint16_t |
| int | int32_t<br/>uint32_t |
| bigint | int64_t<br/>uint64_t |
| float | float |
| double | double |
| char<br>varchar | const char*<br>std::string |
| blob<br>binary<br>text | qtl::const_blob_data<br>std::istream |
| date<br>time<br>datetime<br/>timestamp | qtl::mysql::time |

### MySQL的字段数据绑定

| 字段类型 | C++类型 |
| ------- | ------ |
| tinyint | int8_t<br/>uint8_t |
| smallint | int16_t<br/>uint16_t |
| int | int32_t<br/>uint32_t |
| bigint | int64_t<br/>uint64_t |
| float | float |
| double | double |
| char<br>varchar | char[N]<br>std::array<char, N><br>std::string |
| blob<br>binary<br>text | qtl::blob_data<br>std::ostream |
| date<br>time<br>datetime<br>timestamp | qtl::mysql::time |

### MySQL相关的C++类
- qtl::mysql::database
表示一个MySQL的数据库连接，程序主要通过这个类操纵数据库。
- qtl::mysql::statement
表示一个MySQL的查询语句，实现查询相关操作。
- qtl::mysql::error
表示一个MySQL的错误，当操作出错时，抛出该类型的异常，包含错误信息。
- qtl::mysql::transaction
表示一个MySQL的事务操作。
- qtl::mysql::query_result
表示一个MySQL的查询结果集，用于以迭代器方式遍历查询结果。

## 有关SQLite的说明

访问SQLite时，包含头文件qtl_sqlite.hpp。

### SQLite的参数数据绑定

| 参数类型 | C++类型 |
| ------- | ------ |
| integer | int</br>int64_t |
| real | double |
| text | const char*<br>std::string<br>std::wstring |
| blob | qtl::const_blob_data |


### SQLite的字段数据绑定

| 字段类型 | C++类型 |
| ------- | ------ |
| integer | int</br>int64_t |
| real | double |
| text | char[N]<br>std::array<char, N><br>std::string<br>std::wstring |
| blob | qtl::const_blob_data<br>qtl::blob_data<br>ios::ostream |

当以qtl::const_blob_data接收blob数据时，直接返回SQLite给出的数据地址；当以qtl::blob_data接收blob数据时，数据被复制到qtl::blob_data指定的地址。

### SQLite相关的C++类
- qtl::sqlite::database
表示一个SQLite的数据库连接，程序主要通过这个类操纵数据库。
- qtl::sqlite::statement
表示一个SQLite的查询语句，实现查询相关操作。
- qtl::sqlite::error
表示一个SQLite的错误，当操作出错时，抛出该类型的异常，包含错误信息。
- qtl::sqlite::transaction
表示一个SQLite的事务操作。
- qtl::sqlite::query_result
表示一个SQLite的查询结果集，用于以迭代器方式遍历查询结果。

## 关于测试

编译测试用例的第三方库需要另外下载。除了数据库相关的库外，测试用例用到了测试框架[CppTest](https://sourceforge.net/projects/cpptest/ "CppTest")。

测试用例所用的MySQL数据库如下：
```SQL
CREATE TABLE test (
  ID int NOT NULL AUTO_INCREMENT,
  Name varchar(32) NOT NULL,
  CreateTime timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (ID)
);

CREATE TABLE test_blob (
  ID int unsigned NOT NULL AUTO_INCREMENT,
  Filename varchar(255) NOT NULL,
  Content longblob,
  MD5 binary(16) DEFAULT NULL,
  PRIMARY KEY (ID)
);
```

测试用例在 Visual Studio 2013 和 GCC 4.8 下测试通过。
