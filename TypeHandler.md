### TypeHandler

Mybatis的实现流程里需要把java世界里的字段类型和关系数据库世界里字段类型进行关联起来，如此，在sql组装参数和sql返回内容转换成java对象时，就可以依照这个关联关系进行转换。