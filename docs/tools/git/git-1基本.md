# Git图解工作原理

## 基本用法
![git_basic](git_basic.jpg)
 
- git reset [--soft | --mixed | --hard] [HEAD]
	- 默认mixed:
		- `git reset HEAD^` # 回退所有内容到上一个版本
		- `git reset HEAD^ hello.php`  # 回退 hello.php 文件的版本到上一个版本
		- `git reset 052e`  # 回退到指定版本
	- 回退版本：
		- HEAD~0 表示当前版本
		- HEAD~1 上一个版本
		- HEAD^2 上上一个版本
| 选项    | 影响       |              |          ||
| :---    | :---       | :---         | :---     |:---|
|         | HEAD(仓库) | 索引(暂存区) | 工作目录 |备注|
| --soft  | Y          | N            | N        |之前提交过的修改内容没有丢失，只是回到了未提交的状态|
| --mixed | Y          | Y            | N        |将索引（暂存区）变成你刚刚暂存该提交全部变化时的状态，会显示工作目录中有什么修改|
| -hard   | Y          | Y            | Y        |在给定提交后所修改的内容都会丢失(新文件会被删除，不在工作目录中的文件恢复，未清除回收站的前提)|


- 也可以跳过 暂存区 直接从 仓库 取出文件或者提交代码：
	 ![git_checkout_direct](git_checkout_direct.jpg)
	
