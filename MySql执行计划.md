# MySql执行计划

### 1.如何查看执行计划

在查询的参数前加上 ***explain***关键字

例如:

```sql
EXPLAIN SELECT * FROM tableName 
```

### 2.查询计划的参数解读 

![](img\1.png)

|     名称      | 描述                                                         |
| :-----------: | :----------------------------------------------------------- |
|      id       | 包含一组数字，表示查询中执行select子句或操作表的顺序（执行顺序从大到小执行；   当id值一样的时候，执行顺序由上往下。 |
|  Select_type  | 表示查询中每个select子句的类型（简单OR复杂）;<br />1. SIMPLE（简单的）：查询中不包含子查询或者UNION（联合）。<br />2. PRIMARY（基本的，首要的）：查询中若包含任何复杂的子部分，最外层查询则被标记为PRIMARY。<br />3. SUBQUERY（子查询）：在SELECT或WHERE列表中包含了子查询，该子查询被标记为SUBQUERY。 　<br />4. DERIVED（衍生）：在FROM列表中包含的子查询被标记为DERIVED（衍生）。 <br />5. 若第二个SELECT出现在UNION之后，则被标记为UNION。 <br />6. 若UNION包含在FROM子句的子查询中，外层SELECT将被标记为：DERIVED。<br /> 7. 从UNION表获取结果的SELECT被标记为：UNION RESULT（结合的结果）。 |
|     table     | 表名称                                                       |
|     Type      | 表示MySQL在表中找到所需行的方式，又称“访问类型”，常见有以下几种：<br />1. ALL：Full Table Scan， MySQL将进行全表扫描。<br />2. index：Full Index Scan，index与ALL区别为index类型只遍历索引树。<br />3. range：range Index Scan，对索引的扫描开始于某一点，返回匹配值域的行，常见于between、<、>等的查询。<br />4. ref：非唯一性索引扫描，返回匹配某个单独值的所有行。常见于使用非唯一索引或唯一索引的非唯一前缀进行的查找。<br />5. eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一索引扫描。<br /><br />6. const、system：当MySQL对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，MySQL就能将该查询转换为一个常量。<br />7. NULL：MySQL在优化过程中分解语句，执行时甚至不用访问表或索引。 |
| possible_keys | 指出MySQL能使用哪个索引在表中找到行，查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询使用。 |
|      key      | 显示MySQL在查询中实际使用的索引，若没有使用索引，显示为NULL。当查询中若使用了覆盖索引，则该索引仅出现在key列表中。 |
|    key_len    | 表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。 |
|      ref      | 表示上述表的连接匹配条件，即那些列或常量被用于查找索引列上的值。 |
|     rows      | 表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数。 |
|     Extra     | 包含不适合在其他列中显示但十分重要的额外信息。<br />1. Using where：表示MySQL服务器在存储引擎受到记录后进行“后过滤”（Post-filter）,如果查询未能使用索引，Using where的作用只是提醒我们MySQL将用where子句来过滤结果集。<br />2. Using  ：表示MySQL需要使用临时表来存储结果集，常见于排序和分组查询。<br />3. Using filesort：MySQL中无法利用索引完成的排序操作称为“文件排序”。 |

###  3.执行计划的局限性

- **EXPLAIN不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况；**
- **EXPLAIN不考虑各种Cache（通常人们所说的Cache就是指缓存SRAM。 SRAM叫静态内存，“静态”指的是当我们将一笔数据写入SRAM后，除非重新写入[新数据](https://www.baidu.com/s?wd=新数据&tn=44039180_cpr&fenlei=mv6quAkxTZn0IZRqIHckPjm4nH00T1YLPHP-mhPbmW6vmHFbmvfk0ZwV5Hcvrjm3rH6sPfKWUMw85HfYnjn4nH6sgvPsT6KdThsqpZwYTjCEQLGCpyw9Uz4Bmy-bIi4WUvYETgN-TLwGUv3EnWT3PWR3PHR)或关闭电源，否则写入的数据保持不变）；**
- **EXPLAIN不能显示MySQL在执行查询时所作的优化工作；**
- **EXPALIN只能解释SELECT操作，其他操作要重写为SELECT后查看执行计划。（mysql5.6的版本已经支持直接查看）**
- **部分统计信息是估算的，并非精确值；**

