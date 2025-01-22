# [一] 安装oceanbase和相关工具
### 1. 选择合适的版本
目前来看rockylinux-8是比较好的选择

### 2. 下载安装ob-ce
```shell
bash -c "$(curl -s https://obbusiness-private.oss-cn-shanghai.aliyuncs.com/download-center/opensource/oceanbase-all-in-one/installer.sh)"
source ~/.oceanbase-all-in-one/bin/env.sh
```

### 3. 安装工具
```shell
yum-config-manager --add-repo https://mirrors.aliyun.com/oceanbase/OceanBase.repo
yum install oceanbase-ce-utils
```

# [二] 调整系统参数
### 1. 创建用户，一般数据库运行在相对固定的账户下比较好
### 2. 修正系统参数

```shell
# 修改系统参数
# /etc/sysctl.conf
fs.aio-max-nr=1048576
fs.file-max=6573688
vm.max_map_count=655360
```
```shell
# 刷新系统参数
sudo sysctl -p
```

```shell
# /etc/security/limits.conf
* soft nofile 20000
* hard nofile 40000
* soft nproc 120000
* hard nproc 120000
* soft core unlimited 
* hard core unlimited
* soft stack unlimited 
* hard stack unlimited
```
```shell
# 刷新系统限制
重新登录账户
```

# [三] 安装oceanbase-ce
```shell
obd demo -c oceanbase-ce
```



# [四] 创建租户
oceanbase希望运行在租户下

#### 1. 创建资源约束
```sql
CREATE RESOURCE UNIT S1_unit_config
                MEMORY_SIZE = '5G',
                MAX_CPU = 1, MIN_CPU = 1,
                LOG_DISK_SIZE = '6G',
                MAX_IOPS = 10000, MIN_IOPS = 10000, IOPS_WEIGHT=1;
DROP RESOURCE UNIT S1_unit_config;
SELECT * FROM oceanbase.DBA_OB_UNIT_CONFIGS WHERE NAME = 'S1_unit_config';
```

#### 2. 创建资源池
```sql
CREATE RESOURCE POOL mq_pool_01 
                UNIT='S1_unit_config', 
                UNIT_NUM=1, 
                ZONE_LIST=('zone1');
DROP RESOURCE POOL mq_pool_01;
SELECT * FROM DBA_OB_RESOURCE_POOLS WHERE NAME = 'mq_pool_01'; 
```

#### 3. 创建租户
```sql
SELECT * FROM oceanbase.DBA_OB_TENANTS;
CREATE TENANT IF NOT EXISTS mysql_t1 
                PRIMARY_ZONE='zone1', 
                RESOURCE_POOL_LIST=('mq_pool_01')
                set OB_TCP_INVITED_NODES='%';
```

# [五] 准备oss的access-key, access-secret并测试

```
# {bucket}是桶名称，例如test-bucket
# oss的bucket路径是{path},例如ob/backup
# oss的连接路径是{endpoint},例如http://oss-cn-hangzhou.aliyuncs.com
# oss的ak是{ak}, oss的sk是{sk}
ob_admin test_io_device -d'oss://{bucket}/{path}' -s'host={endpoint}&access_id={ak}&access_key={sk}'
```

# [六] 开启日志归档
```sql
ALTER SYSTEM SET LOG_ARCHIVE_DEST='LOCATION=oss://{bucket}.{endpoint}/{log-path}?host={endpoint}&access_id={ak}&access_key={sk}&delete_mode=delete' TENANT = mysql_tenant;
ALTER SYSTEM ARCHIVELOG TENANT = mysql_t1;
```

# [七] 开启备份
```sql
ALTER SYSTEM SET DATA_BACKUP_DEST='oss://{bucket}/{backup-path}?host={endpoint}&access_id={ak}&access_key={sk}&delete_mode=delete' tenant=mysql_t1;
ALTER SYSTEM BACKUP DATABASE TENANT = mysql_t1;
```


# [八] 尝试从oss恢复租户

```sql
# 创建资源约束
CREATE RESOURCE UNIT S1_unit_config
                MEMORY_SIZE = '5G',
                MAX_CPU = 1, MIN_CPU = 1,
                LOG_DISK_SIZE = '6G',
                MAX_IOPS = 10000, MIN_IOPS = 10000, IOPS_WEIGHT=1;
# 创建资源池
CREATE RESOURCE POOL restore_pool 
                UNIT='S1_unit_config', 
                UNIT_NUM=1, 
                ZONE_LIST=('zone1');

# 恢复数据
ALTER SYSTEM RESTORE mysql_t1 FROM 'oss://{bucket}/{backup-path}?host={endpoint}&access_id={ak}&access_key={sk},oss://{bucket}/{log-path}/?host={endpoint}&access_id={ak}&access_key={sk}' WITH 'pool_list=restore_pool&concurrency=50';
SELECT * FROM oceanbase.CDB_OB_RESTORE_PROGRESS
```