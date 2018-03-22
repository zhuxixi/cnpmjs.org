# 内网搭建基于CNPM+Mysql的NPM包私服
## 前言
目前8.10LTS版本的node和5.6版本的NPM环境，在使用`npm adduser`命令注册CNPM私服用户时存在bug，显示注册成功，其实是失败了，解决方案是使用9.2.0版本node与5.5.1的npm环境。  

CNPM源码中默认写好的数据库是Sqlite3，但是这个数据库我没用过，而且纯内网环境使用很麻烦，网上教程也少，所以数据库选型为Mysql 5.7。  

具体操作流程请往下看。


## 准备工作 
### 在本机安装node、npm
1. 参照你本机的操作系统版本在本机安装[node、npm环境]
2. 在终端或bash中输入`npm install n -g`安装node环境切换工具[n模块]
3. 在终端或bash中输入`n -V`查看n模块版本，如果成功显示版本号则安装成功
4. 在终端或bash中输入`n 9.2.0`安装 9.2.0 node环境
5. 在终端或bash中输入`n` 显示node环境列表，通过↑↓按键选择刚刚安装的9.2.0版本
6. 在终端或bash中输入`node -v`查看版本是否切换成功
7. 在终端或bash中输入以下命令重新安装4.5.1版本的cnpm工具  
	* `npm uninstall -g cnpm`
	* `npm install -g cnpm@4.5.1`  
8. 以上，node环境安装完毕  
 
### 在服务器安装mysql数据库
1. 参照服务器的操作系统版本安装[Mysql数据库]
2. 在终端或bash中输入 `vi /etc/my.cnf`打开Mysql配置文件  
3. 在配置文件的mysqld配置项下面增加`skip-grant-tables`，否则你要去日志中寻找首次登陆的默认root密码
4. 在终端或bash中输入`systemctl restart mysqld.service`重启mysql服务
5. 在终端或bash中输入`mysql`无密码登陆
6. 在Mysql中输入以下命令修改root用户密码(首字母大写，必须包含一个特殊字符)
  * `update user set authentication_string=password('Chinalife001!')where user='root';`
  * `flush privileges;` 
7. 回到终端输入`systemctl restart mysqld.service`重启
8.  在终端或bash中输入`mysql -uroot -p`，会提示你输入密码，然后登录mysql。
9. 在Mysql中输入以下命令配置mysql远程连接权限
	* `use mysql;`
	* `update user set host = '%' where user = 'root';`
	* `FLUSH RIVILEGES;`
10. 如果远程访问还是不成功，查看以下my.cnf中是否写了bind-address如果有，就注释掉或改成`bind-address=0.0.0.0`，一般mysql 5.7以后的版本都没配置bind-address了。如果还是不行，就查看selinux状态和iptables是不是有规则(一般centos/redhat 7+的版本iptables都默认不安装了)
11. 如果还是不行，就报警吧
12. 以上，mysql安装完毕  

### 在本机下载NPM依赖
1. 在[淘宝cnpm项目github主页]下载Master分支代码
2. 终端进入本地cnpm项目目录，执行`npm install`下载项目所需npm包
3. 在项目目录中创建logs、nfs文件夹
	* 在项目目录中`mkdir logs`
	* 在项目目录中`mkdir nfs`
4. 终端执行`npm start`启动项目
5. 终端执行`curl 127.0.0.1:7002`看看启动是否成功
6. 如果启动成功，将项目目录上传至内网服务器

## 部署CNPM

### 服务器配置CNPM并启动
1. 上传项目至服务器并终端进入项目所在目录
2. 终端输入`pwd`查看当前路径
3. 终端输入`vi config\index.js`编辑cnpm配置文件，配置文件修改视需求而定，如果要搭建一个使用群体庞大的内部私服，还是要仔细研读一下。如果这个私服只是小团队内部使用，可以参考以下配置
4. 修改配置文件中 `bindingHost: '0.0.0.0'`
5. 修改配置文件中 ` logdir: '/你的CNPM项目文件夹在服务器中的路径/log'`  
6. 修改配置文件中 database相关配置项，如下所示
  	
	```
	database: {
    db: 'cnpm',
    username: 'root',
    password: 'Chinalife001!',
    dialect: 'mysql',
    host: '127.0.0.1',
    port: 3306,
    pool: {
      maxConnections: 10,
      minConnections: 0,
      maxIdleTime: 30000
    },
    storage: path.join(dataDir, 'data.sqlite'),
    logging: !!process.env.SQL_DEBUG,
  }  
  ```
  
7. 修改配置文件中npm包存储路径，如下所示

	```
	nfs: require('fs-cnpm')({
    	dir: '/你的CNPM项目文件夹在服务器中的路径/nfs'
  	}),
	```
8. 修改配置文件中`registryHost: '服务器IP地址:7001'`  
9. 配置文件修改完毕之后在在终端输入`npm start`和`curl 服务器IP地址:7002`查看CNPM是否启动成功
10. 浏览器中输入`服务器IP地址:7002`看看有没有CNPM页面
11. 进入项目所在目录/docs文件夹找到db.sql文件，此文件为mysql建库脚本
12. 在终端中执行(不要登录mysql执行)以下代码
	
	```
	mysql -uroot -pChinalife001! -e 'DROP DATABASE IF EXISTS cnpm;' &&\
	mysql -uroot -pChinalife001! -e 'CREATE DATABASE cnpm;' &&\
	mysql -uroot -pChinalife001! 'cnpm' < /你的CNPM项目文件夹在服务器中的路径/docs/db.sql &&\
	mysql -uroot -pChinalife001! 'cnpm' -e 'show tables;'
	```
13. 继续用上面的命令重启mysql
14. 在CNPM项目目录下执行`npm restart`重启CNPM私服

### 配置CNPM源

1. 打开你自己的node项目目录(有package.json文件的目录)输入以下命令来切换CNPM源为我们刚刚搭建的私服
	* `cnpm config set registry="http://服务器ID地址:7001"`   
	
### 创建私服用户
  
1. 在终端中输入以下命令创建私服用户

	```
	cnpm adduser
	输入用户名:test001
	输入密码:test001
	输入邮箱:123@qq.com
	终端显示登录成功
	cnpm whoami
	显示test001
	cnpm logout 登出
	cnpm login 登录
	输入用户名:test001
	输入密码:test001
	输入邮箱:123@qq.com
	终端显示登录成功
	```
2. 登录mysql执行以下代码查看用户是否创建成功
	
	```
	use cnpm
	select * from user;
	```
	> 如果你的前面的操作都很顺利，CNPM.USER表中应该有test001的记录
	
### 上传和安装私包

1. 如果用户注册成功，继续在你的node项目目录中执行`cnpm publish`上传你的私包到私服
2. 可以在其他目录试试`cnpm install 你刚上传的包名` 来安装你刚刚下载的私包

>祝顺利



[淘宝cnpm项目github主页]:https://github.com/cnpm/cnpmjs.org
[node、npm环境]:https://nodejs.org/en/download/
[n模块]:https://www.npmjs.com/package/n
[Mysql数据库]:https://dev.mysql.com/downloads/mysql/
