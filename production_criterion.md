# 程序生产化标准

在项目研究和调试完成后, 将程序整理并落地成正常(如每日)生产流程的过程, 称为生产化.

程序生产化通常需要整理以下几点:
 - git: 提交版本控制
 - env: 程序所需主环境(如cpp, py相关环境)以及其他环境变量
 - dep: 程序特殊依赖(程序包, 中间数据或者其他特殊条件等)
 - entry: 规范函数入口(封装一层统一的函数调用入口)
 - scheduler: 正常情况下的调度程序(如cronjob)


## git

 - 建立生产分支

提倡在项目初创开始就是用版本控制, 在生产化完成后, 生成一个单独的生产分支或者标签(如prod_1.0, prod_YYYYMMDD), 方便在生产机器上部署, 也防止被后续开发所污染

 - 分离参数, 环境变量和程序

版本控制上, 尽量避免上传具体参数, 中间数据或者环境变量, 因为不同机器上的参数可能有变化. (但需要给出环境变量设置说明, 后续env部分会有详细说明)


## env + dep 环境和依赖

- 程序运行时环境

只程序所使用的语言环境的版本, 如 python==3.10, cpp-11
对应的, 也有内部定义的一些主运行时环境, 如py36, gpu38, (cpp)qi-platform-1.4.0

- 程序包

写在项目目录下新建requirement.txt, 如python环境下: 
```python
# 内部包版本
qi.common_tools==0.1.52

# 特殊包/特殊版本要求
pandas>=1.5
```

- 环境变量

参考qi.common_tools的import settings的方法, 将环境变量和参数都写在.env文件中, 并注释每个参数的含义
```env
# common_tools.np_data_loader配置环境变量
QDV2_RUN_MODE=LOCAL
BT_DATA_HOME=${QDV2}

# qi.common_tools.calendar_utils所需环境变量
QI_HOME=${HOME}/qi_conf
```

## entry + scheduler 函数入口 和 时间调度

这是整个生产化的重中之重, 相当于每个生产任务的基本入口.

以下均以T日任务为例

 - sleeper 时间依赖, 任务可以启动(检查数据依赖)的时间, 如:

    1. n0td_1500: 当日收盘后即可运行, 则为(next 0 trading day, hhmm=1500)
    2. n1d_0800:  下一自然日8点: (next 1 natrual day, hhmm=0800)
    3. n1td_0800: 下一交易日8点: (next 1 trading day, hhmm=0800)


 - sensor 数据依赖
    1. 数据项
    2. 数据项的时间范围
    3. 能否(强制)忽略数据缺失

如: dataset_id=Daily/PriceVolume/Close, 所需[T-10, T]日数据

 - output 输出

输出数据文件模式和"数据时间"
如: 

    1. path_xxx/data.$YYYYMMDD.csv, 时间为T日
    2. xxx/data.dat, 时间为T-1日

 - param 入参

    1. from_date, to_date: 任务执行的时间范围
    2. overwrite: 重复执行时, 是否覆盖输出
    3. ignore_miss_datas: 是否(强制)忽略数据缺失


 - cmd 调用脚本

    1. 调用该任务的bash脚本, 若是python脚本, 推荐在函数入口使用click封装, 这样调用(传参)时与bash脚本一致
    2. logs输出, 按任务重定向(输出)运行日志文件, 提倡按任务名和日期区分开来, 便于查错和追溯


这样的调用脚本, 可以放到cronjob, 可以放到集中调用脚本(参考NAS136/public/common_sh/bash_sample.sh), 也具备了放入airflow的要求, 简单流程图如下:

```mermaid
flowchart LR

A[start] --> B[sleeper] --> C[input sensor] --> D[cmd task] --> E[output sensor] --> F[end]

```