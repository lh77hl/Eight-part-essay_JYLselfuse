# MySQL

## 基础篇

### 一、执行一条select语句期间发生了什么？

1 MySQL的基本架构和功能是什么？

答：MySQL的架构共分两层：**Sever层**，负责建立连接、分析以及执行SQL；**存储引擎层**，负责数据的存储和提取。

其中Sever层实现MySQL大部分的核心功能模块，包括**连接器、查询缓存、解析器、预处理器、优化器、执行器**等。所有的**内置函数**(日期、时间、数学、加密函数等)、所有的**跨存储引擎功能**(存储过程、触发器、视图等)也是在这一层实现。

存储引擎层支持不同的存储引擎，他们可以共用一个Sever层。现在InnoDB是默认的存储引擎，默认使用B+树索引类型。

2 **第一步：连接器→连接客户端与MySQL服务器的第一道网关，本身不执行语句，仅仅管控连接**

**功能**-接收客户端请求；建立TCP连接或本地套接字连接；验证用户名密码是否正确；检查用户是否有权限访问数据库

**具体工作流程**-**建立连接**，根据输入的主机地址选择连接方式，使用TCP握手连接；**进行身份认证**，服务端发送随机字符串，客户端加密后返给服务端，服务端进一步验证加密结果是否正确；**校验权限**，登录成功后，由连接器查询数据库的user表判断客户端是否有权限访问或执行特定语句，并保存权限便于此连接后续任何操作都基于这个权限逻辑进行判断。【如果管理员在连接途中修改了用户权限，当前连接的操作是不影响的，只有下一次连接时才会用新的权限设置】

拓展-连接池可以复用连接避免每次连接带来的开销；MySQL支持TCP层的Keep-Alive ，或应用层用连接池实现“长连接”。

【如何查看MySQL服务被多少客户端连接？】

答：执行**show processlist**命令可以查看。

对处于休眠状态的用户，其连接的最大空闲时长由**wait_timeout**参数控制，默认8小时，超过这个时间就会自动断开。也可以通过**kill connection + id**的命令手动断开空闲连接。

其支持的最大连接数，由**max_connections**参数控制，超过时系统会拒绝后面的连接请求。

【MySQL使用长连接时临时使用内存管理连接对象，可能导致占用内存增多而被系统强制kill发生异常重启现象，怎么解决呢？】

答：**定期断开长连接**，释放连接占用的内存资源；或**客户端主动重置连接**，调用mysql_reset_connection函数重置连接，释放内存但不断开连接。

3 **第二步：查询缓存→加速相同SQL重复查询的机制**，发生在解析和执行之前，MySQL8.0已移除Sever层的该特性

**基本原理**-**检查缓存是否启用**，通过判断query_cache_type[=demand就表示关闭这个功能]/size，若未启用则直接跳过这个阶段；**生成查询缓存键**，根据整个SQL语句的文本(包括空格和大小写)计算query cache key；**查询匹配结果**，通过计算好的key去缓存中比对是否有完全一致的记录；如果匹配，且相关表未再发生过写入操作(INSERT/UPDATE/DELETE)，则直接把缓存结果返回，不再走优化器和执行器；反之如果查找失败，或相关表被修改、老旧数据被清除，则正常走优化器、执行器等，并把这次查询和查询结果缓存。

**局限性**-**匹配机制严格**，不适配动态生成的SQL；**缓存失效频繁**，一旦查询涉及的表被修改缓存就失效，对写操作频繁的场景几乎不适用；**并发争用严重**，多线程读写缓存时存在锁竞争。

适用场景-静态数据、重复数据、表更新少查询场景

4 **第三步：解析器→将用户提交的SQL语句进行词法和语法分析，即解析SQL语句转化为数据库可以理解的结构**，是查询处理流程的重要阶段

**工作流程**-**词法分析**，把SQL语句拆分成单个token，包括关键字、字段名、常量以及运算符等；**语法分析**，检查SQL语句是否符合语法规则，并生成语法树。

**输出结果**-**抽象语法树(AST)和内部表示结构**，传递给下面的优化器做进一步处理。

5 **第四步：预处理器、优化器、执行器→验证SQL的语义是否正确并执行SQL**

**①预处理器(prepare)：进行语义分析**。验证语义是否正确，包括要查询的表、字段等是否存在，字段和表之间是否匹配，以及类型是否兼容等。

**②优化器(optimize)：确定SQL查询语句的执行方案**。在多个可行的执行计划中，优化器可以考虑成本决定选择代价最低的。可以通过explain命令查询某语句的执行计划。通过代价模型的估算[根据统计信息，如查询行数、唯一值的个数、索引的选择度等]，确定访问路径[全表or索引]、连接顺序来选择连接算法。【总结来说，如果一个查询可以通过二级索引就直接拿到所有需要的字段，就是所谓的“覆盖索引”，此时优化器就不再需要“先二级索引找到主键，再用主键查整行数据”的**回表**方式进行查询。反之，如果查询字段不在二级索引里，比如过滤条件是name，但是要查询name对应的email，就需要先通过二级索引定位到主键——id，再根据这个id主键索引到整行数据，进而取出email字段对应的value】

【补充知识-在InnoDB存储引擎中，**主键索引B+树的叶子节点存储整行数据，二级索引B+树的叶子节点存储对应行的主键值而不是整行数据**】

**③执行器(execute)：按照优化器给出的“执行计划”一步步执行模块。**执行过程中，执行器和存储引擎交互，以记录为单位。下面是三种不同的执行方式。

**主键索引查询**-访问类型为const，**调用read_first_record函数指向存储引擎<u>索引查询的接口</u>**，令其根据查询条件定位符合的第一条记录→**存储引擎**根据主键索引的B+树结构**定位符合条件的第一条记录**，若记录不存在则报错并结束查询，若记录存在则将其返回给执行器→**执行器**获得记录后**进一步判断是否符合查询条件**，若不符合则跳过这条记录，若符合则返给客户端→基于while循环再次调用read_record函数指向**-1**，令**执行器退出循环**，结束查询。

**全表扫描**-访问类型为all，**调用read_first_record 函数指向存储引擎<u>全扫描的接口</u>**，令其读取表中的第一条记录→**执行器逐一判断所读记录是否符合过滤条件**，若不是则跳过，若是则将记录返给客户端→基于while循环再次调用read_record函数指向**全扫描接口**，**继续扫描下一条记录**→直到存储引擎读完表中所有记录向执行器返回读取完毕的信息→执行器收到信息退出循环停止查询。

**索引下推**[减少二级索引在查询时的回表操作]-Server层调用存储引擎的接口定位满足查询条件的第一条二级索引记录→存储引擎定位到后，先不执行回表，而是先判断该索引中包含的其他列条件是否成立[直接在索引阶段筛掉不符合其他列条件的数据，而不是存储引擎把所有符合第一个条件的记录都找出来用主键回表取出一条条整行数据，再一个个让Server层自己比对其他列条件满不满足]，若不成立则直接跳过该二级索引，若成立再执行回表操作，把完整记录返给Sever层→Server层进一步判断其他查询条件是否成立，若不成立则跳过继续向存储引擎要下一条记录，若成立则返给客户端→直到存储引擎把表中所有记录读完。【但该种方式的前提条件是要查询的字段在二级索引的叶子节点可以拿到，或者和二级索引建立了复合索引】【总的来说，索引下推把原本需要Server层判断的条件提前下放到存储引擎层判断】

【总结一下总流程就是：先根据不同的索引类型调用函数指向存储引擎的接口[索引查询/全扫描]；1>索引查询的话就是存储引擎先定位符合条件的记录再让执行器进一步判断一下2>全表扫描的话就是执行器逐个判断读到的记录符不符合条件；1>索引查询的话直接指向-1就退出循环了2>全表扫描的话，要先指向全接口挨着把所有记录都读完才能退出循环。】



### 二、MySQL一行记录是怎么存储的？

1 MySQL的数据存在哪个文件里？

答：MySQL数据主要存储在磁盘上的文件中，具体文件位置和格式取决于存储引擎以及配置。

一般存储在默认的数据目录中，具体的文件夹路径为“/var/lib/mysql/数据库名”，该路径下一般有db.opt，表名.frm和表名.ibd三个文件。

**db.opt**-存储当前数据库默认的字符集和字符校验规则

**表名.frm**-保存每个表的元数据信息，主要包含表结构的定义

**表名.ibd**-保存每个表的数据信息，包括数据和索引等，也称为独占表空间文件。

【表空间文件的结构是什么样的？】

答：表空间由**段(segment)、区(extent)、页(page)、行(row)**组成。

**行**-数据库表的记录按行进行存放

**页**-InnoDB的数据以页为单位读写，默认大小16KB，同时也是**存储引擎磁盘管理的最小单元**。常见类型：数据页、undo日志页、溢出页。

【补充知识-B+树是什么？它的基本结构是什么样的？】B+ 树是一种**多路搜索树**，非常适合磁盘存储结构。特点是所有值都存放在叶子节点，非叶子节点(根节点)仅存放键值这样的索引信息。类似目录树，非叶子节点即为目录，叶子节点就是实际数据。叶子节点之间通过指针相连形成链表，方便范围查询。

**区**-表中数据量较大时，为某索引分配空间时就以区为单位，默认大小1MB。这样连续的64个页都会划分到一个区，**令链表中相邻页的物理位置也相邻**，便于使用顺序I/O，避免大量随机I/O产生的读取速度慢的问题。

**段**-表空间由各个段组成，一般分为**数据段**[存放叶子节点的区集合]、**索引段**[存放非叶子节点的区集合]和**回滚段**[存放回滚数据{旧版本数据，即事务对数据做修改之前的内容，类似数据修改历史的备份，便于撤销修改或提供历史版本的数据}的区集合]等。

2 InnoDB的行格式有哪些？

答：**共4种行格式，分别是Redundant、Compact、Dynamic和 Compressed 行格式**。

Redundant-已弃用

Compact-5.1版本行格式的默认设置，设计初衷是令一个数据页可以存放更多行记录。

Dynamic 和 Compressed-与Compact类似，5.7版本后默认Dynamic格式。

3 Compact行格式简介

一条完整记录分为**记录的额外信息**和**记录的真实数据**两部分。

**①记录的额外信息**

1>变长字段长度列表-用于管理记录中的变长字段，记录每个变长字段实际使用了多少字节的一个列表，仅用于数据表有变长字段时。排列时**按照列的顺序逆序存放**。若某变长字段值为NULL，其长度信息将不被记录，因为NULL不会存放在行格式中记录的真实数据里，但存放位置仍保留。

【为什么变长字段长度列表的信息逆序存放？】

2>NULL值列表

3>记录头信息

**②记录的真实数据**







