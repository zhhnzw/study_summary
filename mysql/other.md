## 其他

**在事务中混合使用存储引擎(如InnoDB和MyISAM表)**

如果该事务需要回滚，非事务型的表上的变更就无法撤销

解决方案：在表设计的阶段就要选好合适的存储引擎

