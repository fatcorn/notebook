# druid源码学习

## druid 功能体系

```
1、数据库连接池 druidDataSource
2、性能监控 statFilter
3、扩展jdbc插件 filter
4、多sql日志记录员 logfilter
5、数据库密码加密 configFilter

 注：使用Druid时，一旦配置filters，则在执行数据库操作时，均将使用代理对象(如ConnectionProxy，StatementProxy，ResultSetProxy等)
```

