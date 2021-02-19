# MySQL常用

# 查询并修改某一字段内容
- 例如：将province字段的‘北京’都改为‘山东’
	`UPDATE tmp_table SET province = REPLACE(province, '北京', '山东') WHERE  province='北京';`
	

