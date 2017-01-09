---
title: gulp重命名和替换js文件
date: 2017-01-09 22:23:22
tags: [Javascript]
category: 前端
---
内部开发的平台出现问题，服务端修改了JS文件，但因为用户浏览器有缓存，导致使用时出现问题，有时候强刷也没有用。而页面引用的js文件比较多，也不能全文件都不缓存。
比较常用的解决方案是：
> 1. 在开发环境修改js文件src.js
> 2. 开发完成后根据文件的md5生成src-{md5}.js
> 2. 开发完成后发布到生产环境，将页面模板中的js链接替换成src-{md5}.js

使用gulp可以很简单的实现上诉步骤。

##安装gulp相关插件

```sh
#安装gulp
npm install gulp 

#生成m5d命名文件的插件
npm install gulp-rev

#替换页面引用url的插件
npm install gulp-rev-collector
```

##在根目录下新建gulp脚本gulpfile.js

```js
var gulp         = require('gulp');
var rev          = require('gulp-rev');//生成md5命名文件
var revCollector = require('gulp-rev-collector');//替换页面引用链接
gulp.task('jsrename',function(){
	return gulp.src(
		//需要重命名的文件列表
	    [
		"./public/js/show-create.js", 
		"./public/js/base.js",  
		"./public/js/main.js",
		"./public/css/strategy-solution.css",
		"./public/css/style.css",
		], 
		{base : "public"})	               //基础目录，保留源文件相对这个目录的路径
	  .pipe(rev())	                     //文件重命名
	  .pipe(gulp.dest("./public/dist"))  //重命名文件保存的目录
	  .pipe(rev.manifest())				//生成转换列表的json文件
	  .pipe(gulp.dest("./public"));		//json文件保存目录
	
 });

//第二个参数表示task需要依赖指定task执行完成后执行
gulp.task('rev', ['jsrename'], function(){
	return gulp.src(
			[
			 './public/*.json',                  //rev-manifest.json的路径
			 './resources/views/**/*.blade.php', //需要替换链接的文件
			])
			//读取 rev-manifest.json 文件以及需要进行css名替换的文件
			.pipe(revCollector({
				replaceReved: true,
				dirReplacements : {
					'/': '/dist',	//路径替换配置
				}
			}))
			.pipe(gulp.dest('./resources/dist_view'));	//替换后的文件输出的目录
});

gulp.task('default', ['jsrename', 'rev']);//执行任务
```

执行gulp后在public目录下生成**rev-manifest.json**文件:

```json
{
  "css/strategy-solution.css": "css/strategy-solution-c51ee853ba.css",
  "css/style.css": "css/style-551a462e31.css",
  "js/base.js": "js/base-f4a7b9200a.js",
  "js/main.js": "js/main-05828bf4dc.js",
  "js/show-create.js": "js/show-create-10239cd469.js"
}
```

代码目录生成的文件列表如下：

```sh
public/dist/
├── css
│   ├── strategy-solution-c51ee853ba.css
│   └── style-551a462e31.css
└── js
    ├── base-f4a7b9200a.js
    ├── main-05828bf4dc.js
    └── show-create-10239cd469.js
```

页面代码中的JS链接替换如下，只会替换脚本中列表的文件：

```
<script type="text/javascript" src="/dist/js/show-create-10239cd469.js"></script>
```

##注意事项：
###1.gulp的任务执行顺序
修改了js文件，执行gulp后发现，`rev-manifest.json`生成了，但是页面中替换的结果还是上一次的。
这个问题是因为**gulp的task执行是异步的**，在执行rev这个任务时，jsrename这个任务还没完成，还没有更新json文件，导致rev任务执行完成后结果没有变化。
解决方案就是加入**rev任务对jsrename的依赖**（gulp.task方法的第二个参数），这样这两个任务就会顺序执行。

###2.文件路径的替换
在renamejs这个task中，因为加了`{base : "public"}`这个配置，在生成json文件时，文件名都是带着路径的，如`js/main.js`。
此时要注意，如果你页面中的js引用url是`/js/main.js`，而你要替换成`/dist/js/main.sh`。在revCollector中dirReplacements应该配置成`'/': '/dist',`(如上gulpfile.js)。这里猜测是因为revCollector中将json中的key整体作为文件名，页面中应用的src去掉这部分文件名后其余部分作为路径。


