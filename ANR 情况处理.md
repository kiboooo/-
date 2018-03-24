ANR 情况处理
---
> Android规定，Activity如果5秒钟之内无法响应屏幕触摸事件或者键盘输入事件就会出现ANR，
> 而BroadcastReceiver如果10秒钟之内还未执行完操作也会出现ANR
> Service 各个生命周期20s内没有处理完毕；

#### 导致ANR常见情形：
1. 耗时的网络访问；
2. 大量的读写操作；
3. Service Binder 数量达到上限；
4. service 忙导致超时无响应；
5. 其他线程持有锁，主线程等待超时；
6. 数据库操作；
7. 在UI线程中调用 sleep（）超过5秒以上；

#### 通过查看 trace.txt 文件
$adb pull data/anr/traces.txt 