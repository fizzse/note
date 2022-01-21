## grant/flush privileges 修改权限命令

## grant 命令做了什么
- grant会修改mysql.user中的数据(权限相关)
- 修改内存中access_user中的信息
- 不会对已经建立的链接产生影响，只会对新建的链接产生影响
## flush privileges 做了什么
- 会清空内存中的access_user。根据mysql.user表中的数据重新构建

## 总结
- grant/revoke 之后不需要执行flush privileges，因为数据总是一致的
- 只有在磁盘与内存不一致时，使用
- 直接操作mysql.user表，如delete，update操作。会导致数据的不一致。需要执行flush privileges