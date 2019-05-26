Linux命令

部署在Linux下的程序，日志很多，而且实时滚动，可以通过以下方式快速查找自己自己想要的内容：

cat log.txt | grep 'ERROR' -A 5

意思是，在log.txt文件中，查找ERROR字符，并显示ERROR所在行的之后5行

cat log.txt | grep 'ERROR' -B 5  之前5行

cat log.txt | grep 'ERROR' -C 5 前后5行

cat log.txt | grep -v 'ERROR' 排除ERROR所在的行

