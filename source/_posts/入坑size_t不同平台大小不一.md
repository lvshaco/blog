title: 入坑size_t不同平台大小不一
categories: 语言
tags: [C,平台差异]
---
源码加密打包模块，发现fseek调用失败errno为22，折腾了好久。
```
size_t offset;
fread(&offset, 4, 1, fp);
fseek(fp, offset, SEEK_SET);
fprintf(stderr, "%d", (int)offset);
```
许久不用调试了，直接打印offset并没有超出文件大小，为何会失败。使劲折腾，猛然醒悟，这里offset使用size_t在64位机器为8字节，fread读取4自己，在这里高位为随机值，故offset实际超大。用
```
fprintf(stderr, "%ld", offset);
```
就会发现。
还是使用uint32_t offset为上。
这里记录下，给自己敲下脑袋。
