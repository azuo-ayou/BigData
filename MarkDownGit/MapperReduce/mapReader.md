不设定压缩格式，直接读文本文件，默认走 UncompressedSplitlineReader这个类，记录切片的大小（字节为单位，也就是文件包含的字符），每次读取4096byte到缓存（这里会有个辅助数组：byte[] buffer, 默认大小是4096）65536byte（64kb）

类继承关系：UncompressedSplitlineReader--->SplitLineReader--->LineReader

1. 每次读下一行数据的时候，其实是读下一个<key,value>，key的值就是数组的上一次读取的位置pos，value就是从pos开始便利buffer，直到遇到了换行符，或者回车符号

123456789123456789123

'123456789123456789120'

select 111111111111111111 = '111111111111111110'

