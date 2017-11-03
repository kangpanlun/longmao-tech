## java 进程CPU使用率高排查

### 方法一：

1. jps  获取java进程的PID
2. jstack pid >> java.txt 导出CPU占用高进程的线程栈
3. top -H -p PID 查看对应进程的哪个线程占用CPU过高
4. echo “obase=16;PID” | bc 将线程的pid转换为16进制，大写转为小写。
5. 在第二步导出的java.txt中查找转换成为16进制的线程PID。找到对应的线程栈。
6. 分析负载高的线程栈都是什么业务操作，优化程序处理问题。

### 方法二：

1. 使用top 定位到占用CPU高的进程PID
   top 
   通过ps aux | grep PID命令
2. 获取线程信息，并找到占用CPU高的线程
   ps -mp pid -o THREAD,tid,time | sort -rn 
3. 将需要的线程ID转换为16进制格式
   printf "%x\n” tid
4. 打印线程的堆栈信息
   jstack pid |grep tid -A 30