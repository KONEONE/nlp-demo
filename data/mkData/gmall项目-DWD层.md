#node 
## 日志格式
#### 页面日志
页面日志，以页面浏览为单位，即一个页面浏览记录，生成一条页面埋点日志。一条**完整**的页面日志包含，一个页面浏览记录，若干个用户在该页面所做的动作记录，若干个该页面的曝光记录，以及一个在该页面发生的报错记录。除上述行为信息，页面日志还包含了这些行为所处的各种环境信息，包括用户信息、时间信息、地理位置信息、设备信息、应用信息、渠道信息等。
```json
{
"common": {                     -- 环境信息
	"ar": "230000",             -- 地区编码
	"ba": "iPhone",             -- 手机品牌
	"ch": "Appstore",           -- 渠道
	"is_new": "1",              -- 是否首日使用，首次使用的当日，该字段值为1，过了24:00，该字段置为0。
	"md": "iPhone 8",           -- 手机型号
	"mid": "YXfhjAYH6As2z9Iq",  -- 设备id
	"os": "iOS 13.2.9",         -- 操作系统
	"uid": "485",               -- 会员id
	"vc": "v2.1.134"            -- app版本号
},
"actions": [{                   -- 动作(事件)
	"action_id": "favor_add",   -- 动作id
	"item": "3",                -- 目标id
	"item_type": "sku_id",      -- 目标类型
	"ts": 1585744376605         -- 动作时间戳
    }],
"displays": [{                  -- 曝光
		"displayType": "query", -- 曝光类型
		"item": "3",            -- 曝光对象id
		"item_type": "sku_id",  -- 曝光对象类型
		"order": 1,             -- 出现顺序
		"pos_id": 2             -- 曝光位置
	},{
		"displayType": "promotion",
		"item": "6",
		"item_type": "sku_id",
		"order": 2,
		"pos_id": 1
	},{
		"displayType": "promotion",
		"item": "9",
		"item_type": "sku_id",
		"order": 3,
		"pos_id": 3
	},{
		"displayType": "recommend",
		"item": "6",
		"item_type": "sku_id",
		"order": 4,
		"pos_id": 2
	},{
		"displayType": "query ",
		"item": "6",
		"item_type": "sku_id",
		"order": 5,
		"pos_id": 1
}],
"page": {                          -- 页面信息
	"during_time": 7648,           -- 持续时间毫秒
	"item": "3",                -- 目标id
	"item_type": "sku_id",         -- 目标类型
	"last_page_id": "login",       -- 上页类型
	"page_id": "good_detail",      -- 页面ID
	"sourceType": "promotion"      -- 来源类型
}
```
#### 启动日志
启动日志以启动为单位，及一次启动行为，生成一条启动日志。一条完整的启动日志包括一个启动记录，一个本次启动时的报错记录，以及启动时所处的环境信息，包括用户信息、时间信息、地理位置信息、设备信息、应用信息、渠道信息等
```json
{
  "common": {
	    "ar": "370000",
	    "ba": "Honor",
	    "ch": "wandoujia",
	    "is_new": "1",
	    "md": "Honor 20s",
	    "mid": "eQF5boERMJFOujcp",
	    "os": "Android 11.0",
	    "uid": "76",
	    "vc": "v2.1.134"
  },
  "start":{   
    "entry": "icon",         --icon手机图标  notice 通知   install 安装后启动
    "loading_time": 18803,  --启动加载时间
    "open_ad_id": 7,        --广告页ID
    "open_ad_ms": 3449,    -- 广告总共播放时间
    "open_ad_skip_ms": 1989   --  用户跳过广告时点
  },
"err":{                     --错误
"error_code": "1234",      --错误码
    "msg": "***********"       --错误信息
},
  "ts": 1585744304000
}
```
#### 数据格式分析
前端埋点获取的 JSON 字符串（日志）可能存在 common、start、page、displays、actions、err、ts 七种字段。其中
* common 对应的是公共信息，是所有日志都有的字段
* err 对应的是错误信息，所有日志都可能有的字段
* start 对应的是启动信息，启动日志才有的字段
* page 对应的是页面信息，页面日志才有的字段
* displays 对应的是曝光信息，曝光日志才有的字段，曝光日志可以归为页面日志，因此必然有 page 字段
* actions 对应的是动作信息，动作日志才有的字段，同样属于页面日志，必然有 page 字段。动作信息和曝光信息可以同时存在。
* ts 对应的是时间戳，单位：毫秒，所有日志都有的字段
综上，我们可以将前端埋点获取的日志分为两大类：启动日志和页面日志。二者都有 common 字段和 ts 字段，都可能有 err 字段。页面日志一定有 page 字段，一定没有 start 字段，可能有 displays 和 actions 字段；启动日志一定有 start 字段，一定没有 page、displays 和 actions 字段

## 需求分析
![[Pasted image 20240307141951.png]]
1. 过滤非Json数据
2. 判断新老客户
3. 对数据进行分流
	1. 曝光日志
	2. 页面日志
	3. 启动日志
	4. 动作日志
	5. 错误日志
#### 曝光日志
```json
{  
    "display_type": "activity",  
    "page_id": "home",  
    "item": "2",  
    "common": "{\"ar\":\"440000\",\"uid\":\"532\",\"os\":\"Android 10.0\",\"ch\":\"wandoujia\",\"is_new\":\"0\",\"md\":\"vivo iqoo3\",\"mid\":\"mid_732092\",\"vc\":\"v2.1.134\",\"ba\":\"vivo\"}",  
    "item_type": "activity_id",  
    "pos_id": 1,  
    "order": 1,  
    "ts": 1592069314000  
}
```
#### 页面日志
```json
{  
    "common": {  
        "ar": "440000",  
        "uid": "532",  
        "os": "Android 10.0",  
        "ch": "wandoujia",  
        "is_new": "0",  
        "md": "vivo iqoo3",  
        "mid": "mid_732092",  
        "vc": "v2.1.134",  
        "ba": "vivo"  
    },  
    "page": {  
        "page_id": "home",  
        "during_time": 14732  
    },  
    "ts": 1592069314000  
}
```
#### 启动日志
```json
{  
    "common": {  
        "ar": "440000",  
        "uid": "532",  
        "os": "Android 10.0",  
        "ch": "wandoujia",  
        "is_new": "0",  
        "md": "vivo iqoo3",  
        "mid": "mid_732092",  
        "vc": "v2.1.134",  
        "ba": "vivo"  
    },  
    "start": {  
        "entry": "notice",  
        "open_ad_skip_ms": 4183,  
        "open_ad_ms": 5023,  
        "loading_time": 18836,  
        "open_ad_id": 5  
    },  
    "ts": 1592069314000  
}
```
#### 动作日志
```json
{  
    "page_id": "good_detail",  
    "item": "29",  
    "action_id": "favor_add",  
    "item_type": "sku_id",  
    "comon": "{\"ar\":\"440000\",\"uid\":\"532\",\"os\":\"Android 10.0\",\"ch\":\"wandoujia\",\"is_new\":\"0\",\"md\":\"vivo iqoo3\",\"mid\":\"mid_732092\",\"vc\":\"v2.1.134\",\"ba\":\"vivo\"}",  
    "ts": 1592069320458  
}
```
#### 错误日志
```json
{  
    "common": {  
        "ar": "230000",  
        "uid": "610",  
        "os": "Android 11.0",  
        "ch": "wandoujia",  
        "is_new": "0",  
        "md": "Xiaomi 10 Pro ",  
        "mid": "mid_329572",  
        "vc": "v2.1.134",  
        "ba": "Xiaomi"  
    },  
    "err": {  
        "msg": " Exception in thread \\  java.net.SocketTimeoutException\\n \\tat com.atgugu.gmall2020.mock.bean.log.AppError.main(AppError.java:xxxxxx)",  
        "error_code": 2984  
    },  
    "page": {  
        "page_id": "login",  
        "during_time": 6523,  
        "last_page_id": "good_detail"  
    },  
    "ts": 1592069320000  
}
```
## 1.流量域独立访客事务事实表
![[Pasted image 20240308021020.png]]

## 2.流量域用户跳出事务事实表
![[Pasted image 20240308043618.png]]


## 3.交易域加购事务事实表【搁置】
![[Pasted image 20240309004605.png]]


## 4.交易域订单预处理表
![[Pasted image 20240310160155.png]]


```sql

table:order_detail
sku_num -> sku_num
order_price -> split_original_amount

table:order_info
type -> 缺失
old -> 缺失

```

## 5.交易域取消订单事务事实表
![[Pasted image 20240311121150.png]]


## 6.交易域支付成功事务事实表
![[Pasted image 20240311125049.png]]


## 7.交易域退单事务事实表
![[Pasted image 20240311132022.png]]

## 8.交易域退款成功事务事实表

## 9.工具域优惠券使用（下单）事务事实表

## 10.工具域优惠券使用（支付）事务事实表

## 11.互动域收藏商品事务事实表

## 12.互动域评价事务事实表

## 13.用户域用户注册事务事实表


