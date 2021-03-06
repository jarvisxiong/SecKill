# SecKill 秒杀系统
## 第一阶段 多线程模拟抢票
 1. 数据库设计
  + 商品表
  ```SQL
 drop table GOODS cascade constraints;
/*==============================================================*/
/* Table: GOODS                                                 */
/*==============================================================*/
create table GOODS 
(
   ID                   NUMBER               not null,
   NAME                 VARCHAR2(128)        not null,
   DESCRIPTION          VARCHAR2(512),
   TOTAL_AMOUNT         NUMBER(18)           not null,
   QUANTITY             NUMBER               not null,
   VERSION              NUMBER(18)           default 1,
   GMT_UPDATE           DATE                 not null,
   GMT_CREATE           DATE                 not null,
   constraint PK_GOODS primary key (ID)
);
comment on table GOODS is
'商品表';
comment on column GOODS.ID is
'主键，自增';
comment on column GOODS.NAME is
'商品名称';
comment on column GOODS.DESCRIPTION is
'商品描述';
comment on column GOODS.TOTAL_AMOUNT is
'商品价格';
comment on column GOODS.QUANTITY is
'库存数量';
comment on column GOODS.VERSION is
'版本号,用于实现乐观锁';
comment on column GOODS.GMT_UPDATE is
'修改时间';
comment on column GOODS.GMT_CREATE is
'创建时间';
  ```
  + 订单表
  ```SQL
  drop table ORDERS cascade constraints;
/*==============================================================*/
/* Table: ORDERS                                                */
/*==============================================================*/
create table ORDERS 
(
   ID                   NUMBER               not null,
   ORDER_NO             VARCHAR2(64)         not null,
   GOODS_ID             NUMBER               not null,
   GOODS_NAME           VARCHAR2(128)        not null,
   BUYER_ID             VARCHAR2(64)         not null,
   TRADE_STATUS         VARCHAR2(32)         not null,
   TOTAL_AMOUNT         NUMBER(18)           not null,
   GMT_UPDATE           DATE                 not null,
   GMT_CREATE           DATE,
   constraint PK_ORDERS primary key (ID)
);
comment on table ORDERS is
'订单表';
comment on column ORDERS.ID is
'主键，自增';
comment on column ORDERS.ORDER_NO is
'订单编号';
comment on column ORDERS.GOODS_ID is
'外键，商品编号';
comment on column ORDERS.GOODS_NAME is
'商品名称';
comment on column ORDERS.BUYER_ID is
'购买人ID';
comment on column ORDERS.TRADE_STATUS is
'交易状态。success：交易成功；failed：交易失败；busy：系统繁忙';
comment on column ORDERS.TOTAL_AMOUNT is
'商品总价';
comment on column ORDERS.GMT_UPDATE is
'修改时间';
comment on column ORDERS.GMT_CREATE is
'创建时间';
  ```
 2. 压测结果
 
|序号|并发数|TPS|平均响应时间(ms)|成功笔数|备注|
|----|-----|---|----------------|--------|---|
|1|10|221|41|13260|数据库悲观锁(select for update)|
|2|10|885|10|7418|数据库悲观锁(select for update nowait)|
|3|10|285.7|32|8863|synchronized+ISOLATION.SERIALIZABLE|
|4|10|282.3|32|5107|可重入锁+ISOLATION.SERIALIZABLE|
|5|10|991.8|9|5200|可重入锁(tryLock)+ISOLATION.SERIALIZABLE|
|6|10|578.2|15|7402|ISOLATION.SERIALIZABLE|
|7|10|994.6|9|8219|线程池 + select for update nowait|
|8|10|653.4|13|8040|线程池 + ISOLATION.SERIALIZABLE|
|9|10|204.5|44|13000(异步)|ActiveMQ未优化|
|10|10|268.9|33|16405(异步)|ActiveMQ,多线程发送,单线程处理.有少量慢查询|
|11|10|825|11|8903|乐观锁|

 + 几点结论
   - 尽量使用编程式事务，细粒度控制事务
   - 如果事务隔离级别设置成了ISOLATION.SERIALIZABLE，则不需要select for update或者synchronized加锁，但这种事务隔离级别会导致程序效率较低
   - 锁有两个层面，一个是java中的对象锁，用于线程同步；另一个层面是数据库的锁；如果是分布式的系统，显然只能利用数据库端的锁来控制并发
   - 乐观锁优点在于开销比悲观锁小，加大了系统的吞吐量，但如果并发较多，容易产生冲突
   - 悲观锁优点在于可以对数据独自占有，直到操作完成再释放锁。缺点是执行过程中其它用户必须等待
   
   
