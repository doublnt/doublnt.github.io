Linux 常用命令记录

1. 重命名或者移动文件

 `mv`  



2. 在Vim 中查找

> 在normal模式下按下`/`即可进入查找模式，输入要查找的字符串并按下回车。 Vim会跳转到第一个匹配。按下`n`查找下一个，按下`N`查找上一个。Vim查找支持正则表达式，例如`/vim$`匹配行尾的`"vim"`。 需要查找特殊字符需要转义，例如`/vim\$`匹配`"vim$"`
>
> http://harttle.com/2016/08/08/vim-search-in-file.html

重命名文件夹 mv Repository Repository.Interface

3. 重启服务器

```shell
sudo ssh root@dev_name
reboot
sudo reboot
```



local 命令查看本地变量