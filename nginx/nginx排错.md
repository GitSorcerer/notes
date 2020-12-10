今天在安装nginx的时候，执行configure命令的时候一切正常，但是在执行make命令的时候就报错了：src/os/unix/ngx_user.c:26:7: error: ‘struct crypt_data’ has no member named ‘current_salt’ 

![安装nginx报错“src/os/unix/ngx_user.c:26:7: error: ‘struct crypt_data’ has no member named ‘current_salt’”](http://cdn.jiweichengzhu.com/upload/image/20200204/w/b6ef5dd1-9137-46b4-b7a3-d74e8f937a14.png)

在网上找了很多资料，都说删除Makefile文件中的Werror，但是测试过后发现并不怎么好使。

nginx/objs/Makefile

![安装nginx报错“src/os/unix/ngx_user.c:26:7: error: ‘struct crypt_data’ has no member named ‘current_salt’”](http://cdn.jiweichengzhu.com/upload/image/20200204/w/3e3f6a23-9195-4b17-bcde-59c4f885f50a.png)

后来又找到了一篇文章，直接按照错误提示中的文件和行数，到指定位置，将current_salt相关的信息给注释掉，简单粗暴，瞬间让我惊为天人！

/nginx/src/os/ngx_user.c

![安装nginx报错“src/os/unix/ngx_user.c:26:7: error: ‘struct crypt_data’ has no member named ‘current_salt’”](http://cdn.jiweichengzhu.com/upload/image/20200204/w/6881e6d6-f361-4567-8bdb-23a78c5023c4.png)

双管齐下，问题得到解决！！！