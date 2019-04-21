## 时间戳转换

**获取当前时间戳:** `date +%s`

**将某个时间戳转换成日期时间:** 
`date -d @1544513466`，输出为`Tue Dec 11 15:33:12 CST 2018`

如果想转换成特定的格式可以用`date -d @1544513592 +%Y_%m_%d_%H_%M`

**将某个日期时间转换成时间戳:**
date --date='06/14/2012 07:21:22' +%s，输出为`1339629682`

## 获取当前的年月日时分秒
```text
年：date +%Y
月：date +%m
日：date +%d
时：date +%H
分：date +%M
秒：date +%S
```

## 时间加减

```text
5分钟前时间戳： date -d " - 5 mins" +%s , 如果去掉+%s就是文本日期了
5秒前：date -d " - 5 seconds"
5小时前: date -d " - 5 hours"
5天前：date -d " - 5 days" 
5月前：date -d " - 5 months"
5年前：date -d " - 5 years"
5年5月前：date -d " - 5 years -5months"

5分钟后：date -d " + 5mins"  

如果是循环变量可以写成:
min=`date -d "- $i mins" +%M`
```



## 边边角角

```text
如果想循环获取整5分钟：mod=$(( 10#$min % 5 ))
获取文件的修改时间戳：stat -c %Y truncateAndRemoveLog.sh
获取文件大小：stat -c %s truncateAndRemoveLog.sh
```



