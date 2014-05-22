# 基于vim配置scala开发环境

因为习惯了vim, 所以急需一个基于vim的scala开发环境. 相对高大上的emacs来说, vim对scala支持的不算好, 和专用的一些IDE更是不能比了...不过通过一番适配, 至少可以支持语法高亮和ctags跳转了, 这样也算是勉强能用了. 这里假定读者已经安装好了较高版本的vim(笔者的vim版本7.2), ctags工具和taglist插件.

## 1. 语法高亮
首先到[scala-dist](https://github.com/scala/scala-dist/tree/master/tool-support/src/vim)下载需要的插件, 放到本地的$HOME/.vim文件夹里.

## 2. ctags跳转
创建$HOME/.ctags文件, 把下面的代码拷贝进去
	
	--langdef=scala
	--langmap=scala:.scala
	--regex-scala=/^[ \t]*class[ \t]+([a-zA-Z0-9_]+)/\1/c,classes/
	--regex-scala=/^[ \t]*trait[ \t]+([a-zA-Z0-9_]+)/\1/t,traits/
	--regex-scala=/^[ \t]*type[ \t]+([a-zA-Z0-9_]+)/\1/T,types/
	--regex-scala=/^[ \t]*def[ \t]+([a-zA-Z0-9_\?]+)/\1/m,methods/
	--regex-scala=/^[ \t]*val[ \t]+([a-zA-Z0-9_]+)/\1/C,constants/
	--regex-scala=/^[ \t]*var[ \t]+([a-zA-Z0-9_]+)/\1/l,local variables/
	--regex-scala=/^[ \t]*package[ \t]+([a-zA-Z0-9_.]+)/\1/p,packages/
	--regex-scala=/^[ \t]*case class[ \t]+([a-zA-Z0-9_]+)/\1/c,case classes/
	--regex-scala=/^[ \t]*final case class[ \t]+([a-zA-Z0-9_]+)/\1/c,case classes/
	--regex-scala=/^[ \t]*object[ \t]+([a-zA-Z0-9_]+)/\1/o,objects/
	--regex-scala=/^[ \t]*private def[ \t]+([a-zA-Z0-9_]+)/\1/pd,defs/

这里其实并不是特别严格的语法匹配, 仅仅是关键字匹配. 在使用时发现如果代码里用了override def这种该匹配规则匹配不到, 可以在def下面再加一个override def. 这样你的ctags就可以支持scala程序了. 可以在项目目录下执行

	ctags -h ".scala" -R

接下来就可以在代码里面用ctrl + ']'跳转了.

## 3. TagList设置
默认的taglist不识别scala语法, 不过在使用ctags匹配规则以后我们已经可以配置taglist支持scala了. 具体只要在$HOME/.vim/plugin/taglist.vim里找到s:tlist_def_yacc_settings, 在这一行下面加入

	let s:tlist_def_scala_settings = 'scala;t:trait;c:class;T:type;' .
                      \ 'm:method;C:constant;l:local;p:package;o:object'

OK, done.

	