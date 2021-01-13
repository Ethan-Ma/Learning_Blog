#LZ4
LZ4就是一个用 16K大小的哈希表 存储字典并简化检索的LZ77。

##压缩原理
举个例子：
输入：abcde_bcdefgh_abcdefghxxxxx
输出：abcde_(5,4)fgh_(14,5)fghxxxxx
//(5,4)代表向前5个byte，匹配到的内容长度为4.

1. 压缩格式
![LZ4 Sequence]()

输入：abcde_bcdefgh_abcdefghxxxxxxx
输出：tokenabcde_(5,4)fgh_(14,5)fghxxxxxxx
格式：[token]literals(offset,match length)[token]literals(offset,match length)....

其他情况也可能有连续的匹配：

输入：fghabcde_bcdefgh_abcdefghxxxxxxx
输出：fghabcde_(5,4)(13,3)_(14,5)fghxxxxxxx
格式：[token]literals(offset,match length)[token](offset,match length)....
这里(13,3)长度3其实并不对，match length匹配的长度默认是4

Literals指没有重复、首次出现的字节流，即不可压缩的部分
Match指重复项，可以压缩的部分
Token记录literal长度，match长度。作为解压时候memcpy的参数
//重复项越多越长，压缩率就越高；

2. 压缩算法实现
大致流程，压缩过程以至少4个字节为扫描窗口查找匹配，每次移动1byte进行扫描，遇到重复的就进行压缩；
由于offset用2个字节表示，所以只能扫描到2^16(64KB)距离的元素，对于4KB的内核页，只需要用到12位；
扫描步长1字节是可以调整的，即对应LZ4_compress_fast机制，步长变长可以提高压缩解压速度，减少压缩率。

![LZ算法实现]()

3. 解压算法
根据解压前的数据流，取出token内的length，literals直接复制到输出，即memcpy(src,dst,length)
遇到match，在从前面已经拷贝的literals复制到后面即可.
