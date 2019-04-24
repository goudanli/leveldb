- table_builder.cc 实现sstable的构建，通过Add函数，把可key/value 数据写入内部数据接口data_block中，然后适当的时候，把data_block写入sstable文件中，注意：data_block中包含多条k/v数据

- table.c实现了sstable的打开，数据的读取

block_builder.cc 主要实现了Block data的构建过程，Block data内部有很多k，v数据对
block.cc  主要实现了Block data的内部遍历



