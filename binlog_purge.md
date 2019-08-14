```
按binlog保留个数持续清理
#!/bin/bash

# binlog index文件路径和名称
binlog_index_name=/var/lib/mysql/archive/mysql-bin.index

# 需要保留的binlog个数
reserve_num=10

# 执行检查间隔，循环检查，发现在该参数指定的时间间隔内有超过reserve_num参数指定的binlog个数，即执行删除，否则等待下一次检查
check_interv=60

# 单个文件删除间隔，在reserve_num参数指定的间隔内发现binlog数量超过reserve_num参数指定的数量就会执行删除，在有多个文件需要删除时，按照rm_interv参数指定的时间间隔逐个删除文件
rm_interv=0

# 执行日志
log_file=/tmp/`basename $0`.log

rm -f $log_file

# 执行清理binlog函数
exec_purge_binlog() {
    total_num=`sed -n '$=' $binlog_index_name`
    let purge_num=${total_num}-${reserve_num}
    if [ "$purge_num" -gt "0" ];then
        purge_files=(`sed -n "1,${purge_num}p" $binlog_index_name`)
        start_file=$(basename $(sed -n '1p' $binlog_index_name))
        end_file=$(basename $(sed -n "${purge_num}p" $binlog_index_name))
        echo
        echo '=================================================================================================================================='
        echo "发现有需要清理的binlog：binlog文件总个数：${total_num}，需要保留文件个数：${reserve_num}，需要清理的文件个数：${purge_num}，将执行清理的文件名称范围：${start_file}-${end_file}"
        echo "---------------------------`date`----------------------------------正在执行binlog的清理"
        
        i=0
        for purge_file in "${purge_files[@]}"
        do
            echo
            echo "正在清理binlog文件：${purge_file} ----------------------`date`-----------------"
            \rm -f "$purge_file"
            sleep "$rm_interv"
            let i++
            echo "完成清理binlog文件：${purge_file} 时间：`date`，标文件名：${end_file}，目标文件数：${purge_num}，已清理文件个数：${i}"
        done
        
        sed -i "1,${purge_num}d" $binlog_index_name
        echo "本次清理binlog执行完成------------------`date`--------------------"
    else
        echo
        echo '=================================================================================================================================='
        echo "本次未检测到需要清理的binlog，等待下次执行检查中......"
    fi
}


# 执行函数调用
if [ ! -f "$binlog_index_name" ];then
    echo "警告，指定的binlog index文件不存在，脚本退出！！" |tee -a $log_file
    exit 1
else
    while :
    do
        exec_purge_binlog |tee -a $log_file
        sleep "$check_interv"
    done
fi
```
