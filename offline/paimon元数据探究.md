paimon能否只修复个别bucket文件
1.查看manifest引用的bucket文件，如果manifest文件未引用，是


1. paimon 元数据是怎么修复的
2. paimon系统表在哪，和catalog类型有关吗
3. export-manifest 脚本
4. call statements 原理，怎么添加一个call statements
CALL system.rebuild_snapshots('my_table'); 是什么语法，为什么加了paimon的jar包就可以解析


action包是怎么加载的

查看bucket文件到snapshot的映射
为什么要查看
bucket文件损坏后，如果能知道对应哪个snapshot，就可以恢复到对应的snapshot






# orphan files检测，怎么清除orphan files




CALL sys.remove_orphan_files(table => 'dwd.dwd_clt_db_asgcontent_b_book_pm', dry_run => true)

org.apache.paimon.operation.OrphanFilesClean





id,lastopendt,vipEt,lastChid,lastType 

attrList

changeLog消费

然后做merge，写到final paimon ret中

当天的attribute list


paimon保障，做一次演练


CREATE TABLE dwd.dwd_clt_en_hwycstatistics_sys_common_config_pm (
  id BIGINT NOT NULL COMMENT '主键，自增ID',
  business_line STRING COMMENT '业务线标识，例如: db',
  config_type STRING COMMENT '配置类型，例如: code、deeplink、all',
  config_name STRING COMMENT '配置项名称，例如: code_search_window',
  config_desc STRING COMMENT '配置项解释说明，例如:有效搜索时间窗',
  config_value STRING COMMENT '配置值，以字符串形式存储，例如: 30',
  effective_date DATE COMMENT '配置项生效日期 yyyy-MM-dd',
  history_version INT COMMENT '配置项的历史版本号，递增，用于版本管理',
  operator STRING COMMENT '操作者的标识，记录最后修改该配置的人员',
  delete_status TINYINT COMMENT '删除状态',
  ctime TIMESTAMP COMMENT '配置项创建时间，精确到秒',
  utime TIMESTAMP COMMENT '配置项最后更新时间，精确到秒',
  tenant_id BIGINT COMMENT '代理ID')
USING paimon
COMMENT '配置项表'
LOCATION 'hdfs://nameservice1/user/hive/warehouse/dwd.db/dwd_clt_en_hwycstatistics_sys_common_config_pm'
TBLPROPERTIES (
  'bucket' = '-1',
  'changelog-producer' = 'input',
  'deletion-vectors.enabled' = 'true',
  'path' = 'hdfs://nameservice1/user/hive/warehouse/dwd.db/dwd_clt_en_hwycstatistics_sys_common_config_pm',
  'primary-key' = 'id',
  'sink.parallelism' = '4')




select *
from dwd.dwd_clt_pm_bak
where tb_name='dwd_clt_en_hwycstatistics_sys_common_config_pm' limit 100

insert into yfb.dwd_clt_en_hwycstatistics_sys_common_config_fault_drill
select get_json_object(struct_json,'$.id') as id,get_json_object(struct_json,'$.business_line') as business_line,get_json_object(struct_json,'$.business_line') as business_line,
get_json_object(struct_json,'$.config_type') as config_type,get_json_object(struct_json,'$.config_name') as config_name,get_json_object(struct_json,'$.config_desc') as config_desc,
get_json_object(struct_json,'$.config_value') as config_value,get_json_object(struct_json,'$.effective_date') as effective_date,get_json_object(struct_json,'$.history_version') as history_version,
get_json_object(struct_json,'$.delete_status') as delete_status,get_json_object(struct_json,'$.ctime') as ctime,get_json_object(struct_json,'$.utime') as utime,
get_json_object(struct_json,'$.tenant_id') as tenant_id
from dwd.dwd_clt_pm_bak
where tb_name='dwd_clt_en_hwycstatistics_sys_common_config_pm'
and snapshot_date='20251027'


sudo -u hdfs hdfs dfs -setrep -w 0 /user/hive/warehouse/yfb.db/dwd_clt_en_hwycstatistics_sys_common_config_fault_drill/bucket-80/data-5ea0c5d2-212b-4025-ae08-1f670e91f7f0-1.parquet


sudo -u hdfs /srv/bd/flink-1.19.1/bin/flink run -yD execution.checkpointing.interval=180s \
  -yD state.checkpoints.dir='hdfs://nameservice1/flink/paimon/checkpoints/sys_common_config_fault_drill' \
  -yD taskmanager.memory.managed.size=1m \
  -yD taskmanager.memory.network.fraction=0.05 \
  -yD execution.checkpointing.max-concurrent-checkpoints=3 \
  --yarnjobManagerMemory 1g --yarntaskManagerMemory  2g \
  -m yarn-cluster -d --yarnname 'paimon_sys_common_config_fault_drill' --yarnqueue root.yarn.hdfs -p 4 -ys 4 \
  --allowNonRestoredStat \
  /srv/bd/flink-1.19.1/apps/paimon-flink-action-1.0.0.jar \
  mysql-sync-database \
  --warehouse hdfs://nameservice1/user/hive/warehouse \
  --database yfb \
  --table_prefix dwd_clt_en_hwycstatistics_ \
  --table_suffix _fault_drill \
  --type_mapping 'tinyint1-not-bool,to-nullable,char-to-string,timestamp-with-local-time-zone' \
  --mysql_conf hostname='hwyc-aurora-01-cluster.cluster-ciyshszi7xv9.ap-southeast-1.rds.amazonaws.com' \
  --mysql_conf username=hd_repo_user \
  --mysql_conf password='hd_repo_userjP3PE6a7JfVlZTebzJB' \
  --mysql_conf database-name=hwyc_statistics \
  --mysql_conf server-id=9600002-9800002  \
  --mysql_conf serverTimezone='Asia/Shanghai' \
  --catalog_conf metastore=hive \
  --catalog_conf uri=thrift://hw-bg-03:9083 \
  --table_conf bucket=-1 \
  --table_conf changelog-producer=input \
  --table_conf sink.parallelism=4 \
  --table_conf deletion-vectors.enabled=true \
  --including_tables 'sys_common_config' \
  --debezium.option.startup.timestamp.millis 1761494400000 \
  


