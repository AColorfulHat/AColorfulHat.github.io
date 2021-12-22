# Hackathon

## 1. 业务需求

1. 实现一个转账接口，一次交易转入方和转出方需要记录两条交易流水
2. 个人账户需实时更新余额并记录交易流水
3. 热点账户余额及交易流水允许延迟更新，最大延迟更新时间为3分钟
4. tran_hist_tab表中prev_balance字段需要记录当前交易发生前的账户余额
5. 个人账户不允许透支，热点账户允许透支

## 2. 技术需求

1. 防重处理
   - 如果第一次请求已处理完成，应返回第一次请求结果
   - 如果第一次请求还在处理中，应该返回正在处理的返回码

## 3. 数据库设计

### account_tab

| 字段             | 格式          | 备注                        |
| ---------------- | ------------- | --------------------------- |
| id               | bigint        | Auto increment              |
| Account_id       | varchar(64)   | Not null, **unique**        |
| Account_type     | Tinyint(3)    | 0-normal, 1-hot             |
| User_id          | varchar(64)   | reference to table:user_tab |
| account_status   | tinyint(3)    | 0: normal                   |
| balance          | bigint(20)    | current balance             |
| last_eod_balance | bigint(20)    |                             |
| last_tran_date   | char(8)       |                             |
| extra_info       | varchar(1024) |                             |
| row_version      | bigint(20)    |                             |
| create_time      | bigint(20)    |                             |
| update_time      | bigint(20)    |                             |

### fw_tran_info(新增)

| 字段        | 格式          | 备注                                        |
| ----------- | ------------- | ------------------------------------------- |
| id          |               | AUTO_INCREMENT, **primary key**             |
| seq_no      | varchar(64)   | Initial_seq_no from request, **unique key** |
| status      | varchar(64)   | I init,S sucess,F failure                   |
| out_msg     | varchar(1024) | request result,DEFAULT NULL                 |
| create_time | bigint(20)    |                                             |
| update_time | bigint(20)    |                                             |

### hot_tran_hist(新增)

| 字段             | 格式        | 备注                                                         |
| ---------------- | ----------- | ------------------------------------------------------------ |
| id               |             | AUTO_INCREMENT, **primary key**                              |
| seq_no           | varchar(64) | Transation sequence number                                   |
| initial_seq_no   | varchar(64) | Initial_seq_no from request                                  |
| account_id       | varchar(64) | transaction account it, reference to table:account_tab, **key(`status`,`account_id`)** |
| tran_amount      | bigint(20)  |                                                              |
| tran_direction   | tinyint(3)  | sub=0, add=1, add or subtract to/from account_id             |
| status           | varchar(3)  | W wait process,E end process,**key(`status`,`account_id`)**  |
| other_account_id | varchar(64) | couterpart account id, reference to table:account_tab        |
| tran_date        | char(8)     | transaction date, format YYYYmmDD                            |
| row_version      | bigint(20)  |                                                              |
| create_time      | bigint(20)  |                                                              |
| update_time      | bigint(20)  |                                                              |

### sys_conf_tab(无用)

| 字段 | 格式 | 备注                            |
| ---- | ---- | ------------------------------- |
| id   |      | AUTO_INCREMENT, **primary key** |
|      |      |                                 |
|      |      |                                 |

### tran_hist_tab

| 字段             | 格式          | 备注                                                         |
| ---------------- | ------------- | ------------------------------------------------------------ |
| id               |               | AUTO_INCREMENT, **primary key**                              |
| seq_no           | varchar(64)   | system generated sequence no, multiple histories in one transaction should use the same seq_no |
| initial_seq_no   | varchar(64)   | Initial_seq_no from request                                  |
| prev_balance     | bigint(20)    | balance before current transaction                           |
| account_id       | varchar(64)   | transaction account it, reference to table:account_tab       |
| tran_amount      | bigint(20)    |                                                              |
| tran_direction   | tinyint(3)    | sub=0, add=1, add or subtract to/from account_id             |
| other_account_id | varchar(64)   | couterpart account id, reference to table:account_tab        |
| tran_date        | char(8)       | transaction date, format YYYYmmDD                            |
| extra_info       | varchar(1024) |                                                              |
| row_version      | bigint(20)    |                                                              |
| create_time      | bigint(20)    |                                                              |
| update_time      | bigint(20)    |                                                              |

### user_tab

| 字段        | 格式          | 备注                            |
| ----------- | ------------- | ------------------------------- |
| id          |               | AUTO_INCREMENT, **primary key** |
| user_id     | varchar(64)   |                                 |
| user_type   | tinyint(3)    | 0: normal, 1: hot               |
| user_name   | varchar(64)   |                                 |
| user_status | tinyint(3)    | 0: normal                       |
| extra_info  | varchar(1024) |                                 |
| row_version | bigint(20)    |                                 |
| create_time | bigint(20)    |                                 |
| update_time | bigint(20)    |                                 |

## 4. 请求体

```json
{
  "from_account": "A1000200030001",
  "to_account": "A1000200030002",
  "amount": "1000",
  "initial_seq_no": "1638418955701__74c44fc8-3d17-476c-a9e4-500199e367f3"
}
	
```

## 5. 代码逻辑

### 1. 防重放

```java
public String preTransfer(String fromAcct, String toAcct, Long amount, String initSeqNo) {
    isEmpty = false;
    开启事务;// 需要开启事务, 否则加锁无效
    try{
      	record = 根据initSeqNo在fw_tran_info表中获取记录for update;
      	if(record == null){
            isEmpty = true;
            新增一条fw_tran_info的记录, status=I;
        }
      	commit;
    }catch(Exception e){
      	rollback;
      	return 异常;
    }
  	
    record = 根据initSeqNo在fw_tran_info表中获取记录;
    开启事务; //涉及多张表，需要事务
    try{
      	if(isEmpty){
            newSeqNo = transfer(fromAcct, toAcct, amount, seqNo);// refer to 2.转账
            record.setOutMsg(newSeqNo)且Status=S,并更新到fw_tran_info;
            commits;
            return newSeqNo;
        }else{
            outMsg = record.getOutMsg();
            if(outMsg != null){
              	return outMsg;
            }else{
              	return 正在处理中的异常;
            }
        }
    }catch(Exception e){
      	rollback;
      	return 异常;
    }
}
```

1. 根据initSeqNo在fw_tran_info表中获取记录for update;

   这里加for update是因为避免同时两个相同的请求进行，都开启了事务，在insert处冲突报错

   

### 2. 转账

```java
public String transfer(String fromAcct, String toAcct, Long amount, String initSeqNo){

    在account_tab中获取fromAcct/toAcct记录,若不为热点户,需要对记录加锁for update;// 热点户先从cache中获取
    seq_no = initSeqNo + "EBK";
    
    // handle debit
    if(fromAcct is 热点户){
      	记录到hot_tran_hist表中,status=W,tranDirection=0;
    }else{
	记录更新到tran_hist_tab表中,tranDirection=0, pre_balance为当前的balance;
	balance = balance - amount;
	if(balance < 0){
	    抛异常;
	}
	更新account_tab表的balance;
	}
  	
    // handle credit
    if(toAcct is 热点户){
      	记录到hot_tran_hist表中,status=W, tranDirection=1;
    }else{
      	记录更新到tran_hist_tab表中,tranDirection=1, pre_balance为当前的balance;
      	balance = balance + amount;
      	if(balance > Long.MAX_VALUE){
            抛异常;
        }
      	更新account_tab表的balance;
    }
    return seq_no;
}
```

优化：热点户存在cache中

- 数量小
- 不易变动

### 3. 热点户定时批量处理

```java
@Scheduled(cron = "0 */1 * * * ?") //一分钟执行一次
public void refreshHotBalance() {
    do{
      	开启事务;
      	try{
            新建cache;
            recordList = 批量在hot_tran_hist获取status=W的记录limit 1000; // order by account_id
	    for(record: recordList){
		在account_tab中获取record.getAccountId的记录 for update; //存入cache
		根据record的记录新增tran_hist_tab的记录;
		if(record.tranDirection == 0){
		    balance = account.balance - amount;
		}else{
		    balance = account.balance + amount;
		}
		更新account的余额;
		更新hot_tran_hist这条record的status=E;
	    }
	    commits;
	}catch(Exception e){
	    rollback;
	    抛异常;
	}
    }while(批量在hot_tran_hist获取记录的数量<1000)
}
```

优化：

1. 在select的时候进行order by account_id

   批量可能数据随机分布，order by可以让尽可能相同的账户在一次批量中

2. 使用cache存储一次批量中account的信息

   prev_balance可以不连续

## 6. 性能测试
- one2many

  热点户向多个普通账户转账

  1. 插⼊1个热点户，账户序号是100000，初始余额为1,000,000,000
  2. 插⼊10000个普通账户，账户序号范围是[110000,120000)，初始余额为0
  3. 准备测试⽤例，金额范围是[100,200]，用例条数是10000，重复率是0

  

- one2one

  普通账户之间互相转账

  1. 插⼊2个普通账户，账户序号分别是200001和200002，初始余额为1,000,000
  2. 准备测试⽤例，金额范围是[100,200]，用例条数是10000，重复率是0

  

- many2many

  多个普通账户向多个普通账户转账

  一定概率重复出现（防重）

  1. 插⼊10000个普通账户，账户序号范围是[310000,320000)，初始余额为1,000,000
  2. 准备测试⽤例，金额范围是[100,200]，独立用例条数是10000，重复率是20%（总用例约12000）

  

- many2one

  多个普通账户向一个普通账户转账

  1. 插⼊1个普通账户，账户序号分别是400000，初始余额为0
  2. 插⼊10000个普通账户，账户序号范围是[410000,420000)，初始余额为1,000,000
  3. 准备测试⽤例，金额范围是[100,200]，用例条数是10000，重复率是0



总结：

| **用例**      | **Requests per second****[#/sec] (mean)** | **Time per request [ms] (mean)** | **Time per request [ms] (mean, across all concurrent requests)** |
| --------- | ----------------------------------------- | -------------------------------- | ------------------------------------------------------------ |
| **one2many**  | **172.41**                                | **57.93**                        | **5.80**                                                     |
| **one2one**  | 54.27                                     | 184.11                           | 18.43                                                        |
| **many2many** | 137.86                                    | 72.46                            | 7.25                                                         |
| **many2one**  | 55.14                                     | 181.24                           | 18.14                                                        |



## 7. 总结

- 包名需全部小写
- 什么样的数据可以存到内存中
  - 数量少/变动少
- 何时开启事务
  - 涉及多张表
- 何时加锁
  - 并发请求
- 一般如何做防重
  - 数据库
  - redis？


## 8. 补充

- 若热点户不可透支应如何处理？
  - 获取当前热点户balance（不加锁）
  - 插入当前流水到hot_tran_hist中，提交事务（为了让别的看到）
  - 计算balance+sum（hot_tran_hist当前账户的所有status=W的记录的amount)
    - 若 > 下限， ok，退出
    - 若 < 上限，ok，退出
  - 将刚插入的流水的状态status=E，后续不做任何处理


