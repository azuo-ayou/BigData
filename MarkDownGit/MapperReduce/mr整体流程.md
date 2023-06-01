- 1.在客户端执行submit()方法之前,会先去获取一下待读取文件的信息
  2.将job提交给yarn,这时候会带着三个信息过去(job.split(文件的切片信息),jar.job.xml)
  3.yarn会根据文件的切片信息去计算将要启动的maptask的数量,然后去启动maptask
  4.maptask会调用InPutFormat()方法去HDFS上面读取文件,InPutFormat()方法会再去调用RecordRead()方法,将数据以行首字母的偏移量为key,一行数据为value传给mapper()方法
  5.mapper方法做一些逻辑处理后,将数据传到分区方法中,对数据进行一个分区标注后,发送到环形缓冲区中
  6.环形缓冲区默认的大小是100M,达到80%的阈值将会溢写
  7.在溢写之前会做一个排序的动作,排序的规则是按照key进行字典序排序,排序的手段是快排
  8.溢写会产生出大量的溢写文件,会再次调用merge()方法,使用归并排序,默认10个溢写文件合并成一个大文件,
  9.也可以对溢写文件做一次localReduce也就是combiner的操作,但前提是combiner的结果不能对最终的结果有影响
  10.等待所有的maptask结束之后,会启动一定数量的reducetask
  11.reducetask会发取拉取线程到map端拉取数据,拉取到的数据会先加载到内存中,内存不够会写到磁盘里,等待所有的数据拉取完毕,会将这些输出在进行一次归并排序
  12.归并后的文件会再次进行一次分组的操作,然后将数据以组为单位发送到reduce()方法
  13.reduce方法做一些逻辑判断后,最终调用OutputFormat()方法,Outputformat()会再去调用RecordWrite()方法将数据以KV的形式写出到HDFS上