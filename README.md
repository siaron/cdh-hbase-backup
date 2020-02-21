## CDH6.3.2 hbase 增量备份
在hbase中快照无法增量备份. 如果使用Copy/export 方式则需要scan表. 社区已有关于增量备份的方式应该在3.0会发布正式版. 看HDP中的文件已经支持. CDH中还不支持. 所以进行了移植. 主要是参照hdp3的方式实现.

## 新建hbase增量备份工程.
- 参考`https://github.com/hortonworks/hbase-release/tree/HDP-3.1.5.6-1-tag/hbase-backup`

## 打包
```
maven clean install -Dskip.test=true
```
## 导入hbase-backup.jar到cdh中
将 打包好的jar包上传到`/opt/cloudera/parcels/CDH/jars`包下. 在`hbase/lib`下创建软连接`ln -s hbase-backup-2.1.0-cdh6.3.2.jar /opt/cloudera/parcels/CDH/jars/hbase-backup-2.1.0-cdh6.3.2.jar`

## 修改hbase的`bash`脚本
修改文件`/opt/cloudera/parcels/CDH/lib/hbase/bin/hbase`

- 增加help
```
129   echo "  backup           Backup tables for recovery"
130   echo "  restore          Restore tables from existing backup image"
131   echo "  completebulkload Run LoadIncrementalHFiles tool"
```
- completebulkload
```
670 elif [ "$COMMAND" = "completebulkload" ] ; then
671   CLASS='org.apache.hadoop.hbase.tool.LoadIncrementalHFiles'
```
- `backup`与`restore`
```
519 elif [ "$COMMAND" = "backup" ] ; then
520   CLASS='org.apache.hadoop.hbase.backup.BackupDriver'
521   if [ -n "${shaded_jar}" ] ; then
522     for f in "${HBASE_HOME}"/lib/hbase-backup*.jar; do
523       if [ -f "${f}" ]; then
524         CLASSPATH="${CLASSPATH}:${f}"
525         break
526       fi
527     done
528     for f in "${HBASE_HOME}"/lib/phoenix-serv*.jar; do
529       if [ -f "${f}" ]; then
530         CLASSPATH="${CLASSPATH}:${f}"
531         break
532       fi
533     done
534   fi
535 elif [ "$COMMAND" = "restore" ] ; then
536   CLASS='org.apache.hadoop.hbase.backup.RestoreDriver'
537   if [ -n "${shaded_jar}" ] ; then
538     for f in "${HBASE_HOME}"/lib/hbase-backup*.jar; do
539       if [ -f "${f}" ]; then
540         CLASSPATH="${CLASSPATH}:${f}"
541         break
542       fi
543     done
544     for f in "${HBASE_HOME}"/lib/phoenix-serv*.jar; do
545       if [ -f "${f}" ]; then
546         CLASSPATH="${CLASSPATH}:${f}"
547         break
548       fi
549     done
550     for f in "${HBASE_HOME}"/lib/commons-lang3*.jar; do
551       if [ -f "${f}" ]; then
552         CLASSPATH="${CLASSPATH}:${f}"
553         break
554       fi
555     done
556   fi
```
- 主要参考了 `https://github.com/hortonworks/hbase-release/blob/HDP-3.1.5.6-1-tag/bin/hbase` 实现

### 备份文档
- `https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.5/hbase-data-access/content/hbase_bar.html`

### example
- Created a "test_backup" table in HBase to test_backup incremetal backup feature on HDP.
  ```
  hbase(main):002:0> create 'test_backup', 'cf'
  0 row(s) in 1.4690 seconds
  
  hbase(main):003:0> put 'test_backup', 'row1', 'cf:a', 'value1'
  0 row(s) in 0.1480 seconds
  
  hbase(main):004:0> put 'test_backup', 'row2', 'cf:b', 'value2'
  0 row(s) in 0.0070 seconds
  
  hbase(main):005:0> put 'test_backup', 'row3', 'cf:c', 'value3'
  0 row(s) in 0.0120 seconds
  
  hbase(main):006:0> put 'test_backup', 'row3', 'cf:c', 'value4'
  0 row(s) in 0.0070 seconds
  
  hbase(main):010:0> scan 'test_backup'       
  ROW                   COLUMN+CELL                                               
  row1                 column=cf:a, timestamp=1317945279379, value=value1        
  row2                 column=cf:b, timestamp=1317945285731, value=value2        
  row3                 column=cf:c, timestamp=1317945301466, value=value4        
  3 row(s) in 0.0250 seconds
  ```
- Now i have taken a full backup using below in it's success
  ```
  sudo -u hdfs hdfs dfs -mkdir /hbase_backup
  
  sudo -u hdfs hdfs dfs -chown -R hbase:hbase /hbase_backup
  
  sudo -u hdfs hdfs dfs -touchz /hbase_backup/test
  
  sudo -u hdfs hbase backup create full hdfs://cm.hadoop:8020/hbase_backup -t test_backup -w 3

  ```
- Now I want to test_backup the "incremetal" backup on the above "test_backup" table. So what I did :
  ```
  put 'test_backup', 'row123', 'cf:a', 'newValue'
  ```
- Now when I am doing the below, it' getting falied
  ```
  hbase backup create incremental hdfs://cm.hadoop:8020/tmp/full
  ```

## hbase-site.xml配置项
  ```
  <property>
    <name>hbase.backup.enable</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.master.logcleaner.plugins</name>
    <value>YOUR_PLUGINS,org.apache.hadoop.hbase.backup.master.BackupLogCleaner
    </value>
  </property>
  <property>
    <name>hbase.procedure.master.classes</name>
    <value>YOUR_CLASSES,org.apache.hadoop.hbase.backup.master.
  LogRollMasterProcedureManager</value>
  </property>
  <property>
    <name>hbase.procedure.regionserver.classes</name>
    <value>YOUR_CLASSES,org.apache.hadoop.hbase.backup.regionserver.
  LogRollRegionServerProcedureManager</value>
  </property>
  <property>
    <name>hbase.coprocessor.region.classes</name>
    <value>YOUR_CLASSES,org.apache.hadoop.hbase.backup.BackupObserver</value>
  </property>
  ```
