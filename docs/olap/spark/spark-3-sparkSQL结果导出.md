# Spark-SQL 导出查询结果
## 使用 -e 参数
	`spark-sql --database ods --num-executors 100 -e "select * from table" > results.txt`

## 使用 -f 参数
	`spark-sql --database mydb --num-executors 100 -f sdk.sql > results.txt`   //执行sql文件

## 参数替换
	```
	export year=2018
	spark-sql --num-executors 100 -v -e "select * from mydb.table where year=${year} and month='09' and day='03' limit 10;" > results.txt
	```
	//-v: 打印Spark每步的执行信息

如果需要进入Spark Sql Cli中执行，这时就可以使用“-d”定义参数，如下:
	```
	spark-sql --num-executors 100 -d year=2018
	#进入Spark Sql Cli，这时可以进行变量替换
	select * from sec_ods.sdk where year=${year} and month='09' and day='03' limit 10;
	```

## 其他
- 设置更大的任务实例，可提高运行速度
	`spark-sql --conf "spark.dynamicAllocation.maxExecutors=200"`
- 增大内存数量来处理巨量数据
	```
	#方式1
	spark-sql --driver-memory 8g -e "select * from table"
	#方式2
	spark-sql --conf "spark.driver.maxResultSize=8g" -e "select * from table"
	#方式3
	spark-sql -e "set spark.driver.maxResultSize=8g;select * from table;"
	```
