# web_log_analyse
This tool aim at trouble shooting and performance optimization based on web logs, it's not a generally said log analyse/statistics solution. It preprocess logs on all web server with a specified period and save the intermediate results into mongodb for finally use(with `log_show.py`)


日志分析在web系统中故障排查、性能分析方面有着非常重要的作用。该工具的侧重点不是通常的PV，UV等展示，而是在指定时间段内提供细粒度（最小分钟级别，即一分钟内的日志做**抽象**和**汇总**）的异常定位和性能分析。

**先明确几个术语**：  
`uri`指请求中不包含参数的部分；`request_uri`指原始的请求，包含参数或者无参数；`args`指请求中的参数部分。（参照[nginx](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)中的定义）  
`uri_abs`和`args_abs`是指对uri和args进行抽象处理后的字符串（以便分类）。例如`"/sub/0/100414/4070?channel=ios&version=1.4.5"`经抽象处理转换为`uri_abs: /sub/?/?/?  args_abs: channel=?&version=?`

### 特点
1. 提供一个日志分析的总入口：经由此入口，可查看某站点所有 server 产生日志的汇总分析；亦可根据`时间段`和`server`两个维度进行过滤
2. 侧重于以**某一类** uri 或其对应的**各类** args 为维度进行分析（参照3）
3. 对 request_uri 进行抽象处理，分为uri_abs和args_abs两部分，以实现对uri和uri中包含的args进行归类分析
4. 展示界面对归类的 uri 或 args 从`请求数`、`响应大小`、`响应时间`三个维度进行展示，哪些请求数量多、那些请求耗时多、哪些请求耗流量一目了然
5. 引入了4分位数的概念以实现对`响应时间`和`响应大小`更准确的描述，因为对于日志中的响应时间，算数平均值的参考意义不大
 
### 实现思路：
分析脚本（`log_analyse.py`）部署到各台 web server，并通过crontab设置定时运行。`log_analyse.py`利用python的re模块通过正则表达式对日志进行分析处理，取得`uri`、`args`、`时间当前`、`状态码`、`响应大小`、`响应时间`、`server name` 等信息存储进MongoDB。查看脚本（`log_show.py`）作为一个总入口即可对所有web server的日志进行分析查看，至于实时性，取决于web server上`log_analyse.py`脚本的执行频率。

#### 前提规范：
 - 各台server的日志文件按统一路径存放
 - 日志格式、日志命名规则保持一致(代码中规定格式为xxx.access.log)
 - 每天的0点日志切割
 
日志格式决定了代码中的正则表达式，是可根据自己情况参考`analyse_config.py`中的正则定义进行定制的)。项目中预定义的日志格式对应如下：
```
log_format  access  '$remote_addr - [$time_local] "$request" '
             '$status $body_bytes_sent $request_time "$http_referer" '
             '"$http_user_agent" - $http_x_forwarded_for';
``` 
#### 对于其他格式的nginx日志或者Apache日志，按照如上原则，稍作就可以使用该工具分析处理。

#### 对于异常日志的处理  
如果想靠空格或双引号来分割各段的话，主要问题是面对各种不规范的记录时(原因不一而足，而且也是样式繁多)，无法做到将各种异常都考虑在内，所以项目中采用了`re`模块而不是简单的`split()`函数的原因。代码里对一些“可以容忍”的异常记录通过一些判断逻辑予以处理；对于“无法容忍”的异常记录则返回空字符串并将日志记录于文件。  
其实对于上述的这些不规范的请求，最好的办法是在nginx中定义日志格式时，用一个特殊字符作为分隔符，例如“|”。这样就不需要re模块，直接字符串分割就能正确的获取到各段(性能会好些)。

### log_show.py使用说明：
#### 帮助信息
```
[ljk@demo ~]$ log_show --help
Usage:
  log_show <site_name> [options]
  log_show <site_name> [options] distribution <request_uri>
  log_show <site_name> [options] detail <request_uri>

Options:
  -h --help                   Show this screen.
  -f --from <start_time>      Start time.Format: %y%m%d[%H[%M]], %H and %M is optional
  -t --to <end_time>          End time.Same as --from
  -l --limit <num>            Number of lines in output, 0 means no limit. [default: 10]
  -s --server <server>        Web server hostname
  -g --group_by <group_by>    Group by every minute, every ten minutes, every hour or every day,
                              valid values: "minute", "ten_min", "hour", "day". [default: hour]

  distribution                Show distribution(about hits,bytes,time) of request_uri in every period(which --group_by specific)
  detail                      Display details of args analyse of the request_uri(if it has args)

  Notice: <request_uri> should inside quotation marks
```

#### 默认对指定站点今日已入库的数据进行分析，默认按点击量倒序排序取前10名
```
[ljk@demo ~]$ log_show api -l 5
====================
Total_hits:999205 invalid_hits:581
====================
      hits  percent           time_distribution(s)                     bytes_distribution(B)              uri_abs
    430210   43.06%  %25<0.01 %50<0.03 %75<0.06 %100<2.82   %25<42 %50<61 %75<63 %100<155                 /api/record/getR
    183367   18.35%  %25<0.02 %50<0.03 %75<0.06 %100<1.73   %25<34 %50<196 %75<221 %100<344               /api/getR/com/?/?/?
    102299   10.24%  %25<0.02 %50<0.02 %75<0.05 %100<1.77   %25<3263 %50<3862 %75<3982 %100<4512          /view/?/?/?/?.js
     62772    6.28%  %25<0.15 %50<0.19 %75<0.55 %100<2.93   %25<2791 %50<3078 %75<3213 %100<11327         /api/getRInfo/com/?/?
     54947    5.50%  %25<0.03 %50<0.04 %75<0.1 %100<1.96    %25<2549 %50<17296 %75<31054 %100<691666      /api/NewCom/list
====================
Total_bytes:1.91 GB
====================
     bytes  percent           time_distribution(s)                     bytes_distribution(B)              uri_abs
   1.23 GB   64.61%  %25<0.03 %50<0.04 %75<0.1 %100<1.96    %25<2549 %50<17296 %75<31054 %100<691666      /api/NewCom/list
 319.05 MB   16.32%  %25<0.02 %50<0.02 %75<0.05 %100<1.77   %25<3263 %50<3862 %75<3982 %100<4512          /view/?/?/?/?.js
 167.12 MB    8.55%  %25<0.15 %50<0.19 %75<0.55 %100<2.93   %25<2791 %50<3078 %75<3213 %100<11327         /api/getR/com/?/?
  96.11 MB    4.92%  %25<0.1 %50<0.18 %75<0.42 %100<1.96    %25<5225 %50<13041 %75<32832 %100<172867      /api/view/getView
  24.77 MB    1.27%  %25<0.01 %50<0.03 %75<0.06 %100<1.82   %25<42 %50<61 %75<63 %100<155                 /api/record/getR
====================
Total_time:117048s
====================
 cum. time  percent           time_distribution(s)                     bytes_distribution(B)              uri_abs
     38747   33.10%  %25<0.01 %50<0.03 %75<0.06 %100<2.82   %25<42 %50<61 %75<63 %100<155                 /api/record/getR
     22092   18.87%  %25<0.02 %50<0.03 %75<0.06 %100<1.73   %25<34 %50<196 %75<221 %100<344               /api/getR/com/?/?/?
     17959   15.34%  %25<0.15 %50<0.19 %75<0.55 %100<2.93   %25<2791 %50<3078 %75<3213 %100<11327         /api/getRInfo/com/?/?
     10826    9.25%  %25<0.02 %50<0.02 %75<0.05 %100<1.77   %25<3263 %50<3862 %75<3982 %100<4512          /view/?/?/?/?.js
      8485    7.25%  %25<0.03 %50<0.04 %75<0.1 %100<1.96    %25<2549 %50<17296 %75<31054 %100<691666      /api/NewCom/list
```
可通过`-f`，`-t`，`-s`参数对`起始时间`和`指定server`进行过滤；并通过`-l`参数控制展示条数

#### distribution 子命令：对指定uri或request_uri在指定时间段内按“分/十分/时/天”为粒度进行聚合统计
```
# 默认按小时分组，默认显示10行
[ljk@demo ~]$ python log_show.py api distribution "/"
====================
request_uri_abs: /
Total hits: 76    Total bytes: 2.11 KB
====================
      hour        hits  hits(%)       bytes  bytes(%)           time_distribution(s)                     bytes_distribution(B)            
  18011911          16   21.05%    413.00 B    19.16%  %25<0.06 %50<0.06 %75<0.06 %100<0.06   %25<23 %50<26 %75<28 %100<28                
  18011912          19   25.00%    518.00 B    24.03%  %25<0.02 %50<0.02 %75<0.02 %100<0.02   %25<26 %50<27 %75<28 %100<28                
  18011913          23   30.26%    700.00 B    32.47%  %25<0.02 %50<0.1 %75<0.18 %100<0.18    %25<29 %50<29 %75<29 %100<29                
  18011914          18   23.68%    525.00 B    24.35%  %25<0.02 %50<0.02 %75<0.02 %100<0.02   %25<28 %50<29 %75<30 %100<30 
```
可通过`-f`，`-t`，`-s`参数对`起始时间`和`指定server`进行过滤；通过`-g`参数指定聚合的粒度（minute/ten_min/hour/day）

#### detail 子命令：对某一uri进行详细分析，查看其不同参数(args)的各项指标分布
```
[ljk@demo ~]$ python log_show.py api detail "/view/?/?/?.json"
====================
uri_abs: /view/?/?/?.json
Total hits: 152080    Total bytes: 990.81 MB
====================
    hits  hits(%)      bytes  bytes(%)  time(%)           time_distribution(s)                   bytes_distribution(B)            args_abs
  147109   96.73%  964.98 MB    97.39%   96.84%  %25<0.02 %50<0.03 %75<0.03 %100<0.41   %25<17 %50<1856 %75<7163 %100<296527      channel=?&version=?
    4971    3.27%   25.83 MB     2.61%    2.42%  %25<0.02 %50<0.03 %75<0.03 %100<0.09   %25<350 %50<2063 %75<6529 %100<68296      ""
```
`detail`子命令后跟随uri（不含参数）或uri_abs
可通过`-f`，`-t`，`-s`参数对`起始时间`和`指定server`进行过滤

### log_analyse.py部署说明：
该脚本的设计目标是将其放到web server的的计划任务里，定时（例如每30分钟或10分钟，自定义）执行，在需要时通过log_show.py进行分析即可。  
`*/30 * * * * export LANG=zh_CN.UTF-8;python3 /root/log_analyse.py &> /tmp/log_analyse.log`

### Note
1. 其中`uri_abs`和`args_abs`是对uri和args进行抽象化（抽象出一个模式出来）处理之后的结果。  
 **uri**：将路径中任意一段全部由数字组成的抽象为一个"?"；将文件名出去后缀部分全部由数字组成的部分抽象为一个"?"  
 **args**：将所有的value替换成"？" 
2. `common_func.py`中还有一些其他有趣的函数

