


《 MySQL性能优化－－数据类型的选择》首发[橙寂博客](http://www.luckyhe.com/post/66.html)转发请加此提示

# MySQL性能优化－－数据类型的选择

## 数值类型

#### 整型

| 类型        | 类型说明    |
| --------- | ------- |
| tinyint   | 非常小的整数  |
| smallint  | 较小整数    |
| mediumint | 中等大小整数  |
| int       | 标准整数    |
| bigint    | 较大整数    |



 **mysql**提供了五种整型： tinyint、smallint、mediumint、int和bigint。int为integer的缩写。这些类型在可表示的取值范围上是不同的。整数列可定义为unsigned从而禁用负值；这使列的取值范围为0以上。各种类型的存储量需求也是不同的。取值范围较大的类型所需的存储量较大。<br>

#### 浮点数
| 类型        | 类型说明    |
| --------- | ------- |
| float     | 单精度浮点数  |
| double    | 双精度浮点数  |
| decimal   | 一个串的浮点数 |

**mysql** 提供三种浮点类型： float、double和decimal。与整型不同，浮点类型不能是unsigned的，其取值范围也与整型不同，这种不同不仅在于这些类型有最大值，而且还有最小非零值。最小值提供了相应类型精度的一种度量，这对于记录科学数据来说是非常重要的（当然，也有负的最大和最小值）。</br>

在浮点数的选择中除了考虑数值范围，必须考虑的是您数据需要的精度，'float'跟'double'都会有精度损失，所以在钱等需要计算的，对精度有要求的一定要选择'decimal'。</br>

#### 类型范围


| **类型说明**     | **存储需求**  | **取值范围**                                 |
| -------------- | ----------------------------------------| ---------------------------------------- |
| tinyint[(m)]  | 1字节   | 有符号值：-128 到127（- 27 到27 - 1）                                                                                                                                                                      无符号值：0到255（0 到28 - 1） |
| smallint[(m)] | 2字节     | 有符号值：-32768 到32767（- 215 到215 - 1）                                                                                                                                                 无符号值：0到65535（0 到21 6 - 1） |
| mediumint[(m)]| 3字节     | 有符号值：-8388608 到8388607（- 22 3 到22 3 - 1 ）                                                                                                                             无符号值：0到16777215（0 到22 4 - 1） |
| int[(m)]       | 4字节 | 有符号值：-2147683648 到2147683647（- 231 到231- 1）                                                                                                                                    无符号值：0到4294967295（0 到232 - 1） |
| bigint[(m)]   | 8字节       | 有符号值：-9223372036854775808 到9223373036854775807（- 263到263-1）                                                                                          无符号值：0到18446744073709551615（0到264 – 1） |
| float[(m, d)] | 4字节    | 最小非零值：±1.175494351e - 38                 |
| double[(m,d)]  | 8字节 | 最小非零值：±2.2250738585072014e - 308         |
| decimal (m, d)| m字节（**mysql** < 3.23），m+2字节（**mysql** > 3.23 ） | 可变；其值的范围依赖于m 和d                          |



## 字符类型

|  **类型名**   |      **说明**      |
| :--------: | :--------------: |
|    char    |      定长字符串       |
|  varchar   |      可变长字符串      |
|  tinyblob  | 非常小的blob（二进制大对象） |
|    blob    |      小blob       |
| mediumblob |     中等的blob      |
|  longblob  |      大blob       |
|  tinytext  |     非常小的文本串      |
|    text    |       小文本串       |
| mediumtext |      中等文本串       |
|  longtext  |     **大文本串**     |
|    enum    |  枚举；列可赋予某个枚举成员   |
|    set     |  集合；列可赋予多个集合成员   |

以下为每个类型的取值范围和占用字节数
| **类型说明**                      | **最大尺寸**      | **存储需求**     |
| ----------------------------- | ------------- | ------------ |
| char( m)                      | m 字节          | m 字节         |
| varchar(m)                    | m 字节          | l + 1字节      |
| tinyblob, tinytext            | 2的8次方- 1字节    | l + 1字节      |
| blob, text                    | 2的16次方 - 1 字节 | l + 2字节      |
| mediumblob, mediumtext        | 2的24次方- 1字节   | l + 3字节      |
| longblob, longtext            | 2的32次方- 1字节   | l + 4字节      |
| enum(“value1”, “value2”, ...) | 65535 个成员     | 1 或2字节       |
| set (“value1”, “value2”, ...) | 64个成员         | 1、2、3、4 或8字节 |

'l' 为所需的额外字节为存放该值的长度所需的字节数。

在短文本的取值上使用'char'跟'varchar'这两个有个区别就是'char'不会去空格，如果长度不够还会补空格所以它是固定长度查询起来比较快，缺点是浪费存储空间对于某些定长的数据可以使用'char'，'varchar'长度可变，存储空间要小。在'innoDB'中推荐使用'varchar'，因为节省了空间。</br>

在长文本的选择上使用 'blob'跟'text'。'blob'是可以用来保存二进制数据的，'text'只能存文本。这两种类型会引起一些性能问题，执行'删除操作'的时候会留下'空洞'。相当于只是在表中删除了，但是t.MYD的实际存储确没有少。所以要使用'OPTIMIZE'去优化
```
OPTIMIZE TABLE 表名;
```

## 日期类型

| **类型名**   | **说明**                    |
| --------- | ------------------------- |
| date      | “yyyy-mm-dd”格式表示的日期值      |
| time      | “hh:mm:ss”格式表示的时间值        |
| datetime  | “yyyy-mm-dd hh:mm:ss”格式   |
| timestamp | “yyyymmddhhmmss”格式表示的时间戳值 |
| year      | “yyyy”格式的年份值表             |
日期时间列类型

| **类型名**   | **取值范围**                                 | **存储需求** |
| --------- | ---------------------------------------- | -------- |
| date      | “1000-01-01”到“9999-12-31”                | 3字节      |
| time      | “-838:59:59”到“838:59:59”                 | 3字节      |
| datetime  | “1000-01-01 00:00:00” 到“9999-12-31 23:59:59” | 8字节      |
| timestamp | 19700101000000 到2037 年的某个时刻              | 4字节      |
| year      | 1901 到2155                               | 1字节      |

 日前时间列类型的取值范围和存储需求  

在日期的选择上其实看我以上两个表格应该就很清楚了。如果你只需要年那么用'year'就好了，只需要日期选择'date' 啥都要那就选择'dateTime' ,'timestamp'因为取值范围比较小，谨慎使用，但是这个有个优点就是能和实际'时区'想对应。</br>