# REST API v2

> 特别提示：建议不要在客户端里写代码直接调用此 API。因为 Android apk 比较容易破解，别人很容易从客户端代码里找出来调用 JPush Remote API 所需要的保密信息，从而可以模拟到你的身份来发起恶意的推送。


> 建议的使用方式是：调用 JPush Remote API 的代码放在你自己的应用服务器上。你自己的应用服务器对自己的客户端提供接口来推送消息。具体请参考推聊的作法：[示例与代码。]()

## 推送全功能接口

### 功能说明

开发者调用此远程接口，向指定的用户推送通知消息，或者自定义消息。

此 API 可用于开发者想要在自己的业务服务器上，灵活地向指定的某个或者某些用户推送消息的场景。

### 调用地址

http://api.jpush.cn:8800/v2/push

> 1. 请使用域名访问 JPush API，不要直接使用 IP。
> 2. 本接口只支持 HTTP Post 请求。
> 3. 若无特殊说明，接口中统一使用 utf-8  编码。
> 4. HTTP Post 的Content-Type 需采用 application/x-www-form-urlencoded
> 5. 考虑内容里可能有一些特殊字符，有必要在调用接口前对内容进行 URL Encode。更详细说明请参考：特殊字符问题。
> 6. 如果你很重视接口安全，请使用 SSL 接口，默认走443ssl加密协议端口，即接口URL改为: [https://+ api.jpush.cn/v2/push](https://+ api.jpush.cn/v2/push)。
> 7. 无论你在极光推送Portal上的应用是生产环境还是测试环境，都使用这个 API 地址推送消息。

### 调用参数



| 参数名称	     | 参数类型    | 是否可选 |  说明   |
| ------------- |:-------------:| -----:|----:|
| sendno    | int | 必须 |发送编号（最大支持32位正整数(即 4294967295 )）。由开发者自己维护，用于开发者自己标识一次发送请求。|
| app_key      | String      |   必须 |待发送的应用程序(appKey)，只能填一个。|
| receiver_type | int      |    必须 |接收者类型。<p>2、指定的 tag。<p>3、指定的 alias<p>4、广播：对 app_key 下的所有用户推送消息。<>5、根据 RegistrationID 进行推送。Android SDK r1.6.0 及以上版本支持。相关文档：JPush Android SDK r1.6.0 版本更新支持 RegistrationID 精确对点推送。|
|receiver_value|String|可选|发送范围值，与 receiver_type 相对应。<p>2、App 调用 SDK API 设置的 tag （标签）。支持多达 10 个，使用 "," 间隔。填写多个 tag 时，最后推送对象是这多个 tag 的 user set 的并集，而不会有重复用户。<p>3、App 调用 SDK API 设置的 alias （别名）。支持多达 1000 个，使用 "," 间隔。<p>4、不需要填<p>5、目标设备的 RegistrationID。支持多达 1000 个，使用 “,” （逗号）间隔
|verification_code|String|必须	|验证串，用于校验发送的合法性。<p>由 sendno, receiver_type, receiver_value, master_secret 4个值拼接起来（直接拼接字符串）后，进行一次MD5 (32位大写) 生成。<p>参考：[verification code 拼接示例]()<p>由于验证串的组成部分有 String 内容，而 JPush采用 UTF-8 编码。所以，如果你的 API 调用没有使用 UTF-8 编码时，首先会遇到 verification_code 不正确验证失败的错误返回，导致调用不成功。|
|msg_type|int|必须|发送消息的类型：<p>１、通知 <p>２、自定义消息（只有 Android 支持） |
|msg_content|String|可选|描述此次发送调用。<p>不会发到用户。|
|platform|String|必须	|目标用户终端手机的平台类型，如： android, ios, winphone 多个请使用逗号分隔。|
|apns_production	|int	|可选	|指定 APNS 通知发送环境：0: 开发环境，1：生产环境。<p>如果不携带此参数则推送环境与 JPush Web 上的应用 APNS 环境设置相同。|
|time_to_live|int|可选	|从消息推送时起，保存离线的时长。秒为单位。最多支持10天（864000秒）。<p> 0 表示该消息不保存离线。即：用户在线马上发出，当前不在线用户将不会收到此消息。<p>此参数不设置则表示默认，默认为保存1天的离线消息（86400秒）。|
|override_msg_id|String|可选	|待覆盖的上一条消息的 ID。<p>指明了此参数，则当前消息会覆盖指定 ID 的消息。覆盖的具体行为是：1）如果被覆盖的消息用户暂时未收到，则以后也不会收到；2）如果被覆盖的消息 Android 用户已经收到，但未清除通知栏，则 Android 通知栏上展示时新的一条将覆盖前一条。<p>覆盖功能起作用的时限是：1 天。<p>如果在覆盖指定时限内该 msg_id 不存在，则返回 1003 错误，提示不是一次有效的消息覆盖操作，当前的消息不会被推送。|

### 调用返回

当调用接口时，极光Push Server会进行简单的校验检查，并立即返回结果。

正常情况下返回码为 200，返回内容类型为字符串，形式为 JSON。

|Key名称|Value内容说明|
|-|-|
|errcode|错误码。[参考：错误码定义]()|
|errmsg	|错误说明|
|msg_id|该消息的 ID|

### 消息内容格式

消息发送接口调用参数里 msg_content 格式有特定要求，此节做具体说明。

首先，其要求为 JSON 格式。具体内容根据类型有不同。

#### 通知类型

当调用参数 msg_type = 1 时，msg_content JSON 要求：

|Key名称|是否必须|Value内容说明|
|-|-|-|
|n_builder_id|可选	|1-1000的数值，不填则默认为 0，使用 极光Push SDK 的默认通知样式。只有 Android 支持这个参数。进一步了解请参考文档 [通知栏样式定制 API]()|
|n_title|可选	|通知标题。不填则默认使用该应用的名称。只有 Android支持这个参数。|
|n_content|	必须	|通知内容。|
|n_extras|	可选	|通知附加参数。JSON格式。客户端可取得全部内容。|
	
		

>关于长度限制，请参考：通知长度限制说明。

>另外，对于 iOS 通知，有些特别的地方需要说明，请参考：iOS APNs 特别说明。

#### iOS APNs 特别说明

通过 JPush API 同时推送 iOS APNs 消息时，有一些内容需要与 APNs 适配。

关于 APNs 的具体详细定义，请参考官方文档：Apple Push Notification Service。


|JPush 字段	|APNs 字段|
|-|-|
|n_content	|alert|
|n_extras -> ios -> badge	|badge|
|n_extras -> ios -> sound	|sound|
|n_extras -> ios >content-available	|content-available|


n_content 字段必须存在，但可以为空字符串。这时，如果 extras 里带了 badge 字段，则收到的通知会显示应用图标右上角数字，而没有通知栏内容。

n_extras 整体可以不填。

指定了 iOS 特定参数的完整的通知 msg_content JSON串示例：

	{"n_content":"通知内容", "n_extras":{"ios":{"badge":88, "sound":"default", "content-available":1}, "user_param_1":"value1", "user_param_2":"value2"}}

####通知长度限制说明

JPush API 同时支持 Andorid 与 iOS 平台的通知推送。

由于 APNs 限制是 255 个字节，所以 JPush 推送通知也依据 APNs 来统一限制。

>通知长度限制的设计，一方面的确是因为 APNs 的原因。另一方面，我们也认为，对于“通知”，在通知栏展示的信息，这么长的长度足够了。
>
>另外请留意：这里说的长度，是指字节。由于使用 UTF-8 编码，所以一个中文字符占 3 个字节。


由于组装 APNs 有几个 JSON 字段，所以 JPush API 通知限制具体大小为：220 个字节。

具体限制算法为：n_content 里边的内容，加上 n_extras 里边的内容，其总长度。

####自定义消息类型


>只有 Android 支持自定义消息

当调用参数 msg_type = 2 时，msg_content JSON 要求：

|Key名称|	是否必须|	Value内容说明|
|-|-|-|
|message|	必须|	自定义消息的内容。
|content_type|	可选|	message 字段里的内容类型。用于特定的 message 内容解析
|title|	可选|	消息标题
|extras|	可选|	原样返回，JSON 格式的更多的附属信息


注：自定义消息所有字段信息的总长度，不得超过 1000 个字节。

>值得注意的是，这些信息极光 Push SDK 本身都不去解析，而是客户端提供 API 给到开发者自己的应用程序去处理。请参考相关文档 接收推送消息。

### verification_code 拼接示例


	int sendno = 3321;
	int receiverType = 2;
	String receiverValue = "game, oldman, student";
	String masterSecret = "71638202938228382811FCB1CB308ADC"; //极光推送portal 上分配的 appKey 的验证串(masterSecret)
	 
	String input = String.valueOf(sendno) + receiverType + receiverValue + masterSecret;
	String verificationCode = StringUtils.toMD5(input);
	
## 特殊字符问题

如果你在调用 API 时直接使用官方的 Java Library - Java v2，在内容里可以发送任何特殊字符。

但是，如果你要自己直接调用本 API，则会有特殊字符问题需要考虑。

原理上，有二部分有特殊字符问题：

+ 推送内容 msg_content 里边是 JSON 字符串，所以 JSON 相关的特殊字符，需要有转义的问题。
+ HTTP API 的方式调用接口，本质上还是基于 URL 来传输，所有有 URL encode 的问题。

所以，在正式调用接口发出内容前，你的代码需要对内容做两个部分的处理： JSON encode, URL encode。

对于 JSON encode，如果你直接使用一些 JSON library来处理 JSON，那么，这些包会自动帮你处理特殊字符转义。

如果你完全手动拼装 JSON字符串，则有必要你自己来写 JSON 特殊字符串转义。请参考 JSON 官方文档：[http://www.json.org/](http://www.json.org/)

URL encode 一般来说，各开发语言平台都提供了这方面的工具方法，来做 URL encode 操作。

## 错误码定义

HTTP 返回码为 200 时，是业务相关的错误。

|错误码|	错误描述|
|-|-|
|0|	调用成功
|10|	系统内部错误
|1001|	只支持 HTTP Post 方法，不支持 Get 方法
|1002|	缺少了必须的参数
|1003|	参数值不合法
|1004|	verification_code 验证失败
|1005|	消息体太大。
|1006|	用户名或密码错误
|1007|	receiver_value 参数 非法
|1008|	appkey参数非法
|1010|	msg_content 不合法
|1011|	没有满足条件的推送目标	<p>如果群发：则此应用还没有一个客户端用户注册。请检查 SDK 集成是否正常。<p>如果是推送给某别名或者标签：则此别名或者标签还没有在任何客户端SDK提交设置成功。
|1012|	iOS 不支持推送自定义消息。只有 Android 支持推送自定义消息。
|1013|	content-type 只支持 application/x-www-form-urlencoded
|1014|	消息内容包含敏感词汇。
	
## See Also

查询消息送达情况请参考： [Report-API](../report_api)

了解API 频率限制：[API 频率限制](../api_rate_limiting/)





