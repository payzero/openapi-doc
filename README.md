# PayZero OpenAPI Documentation v1.12 #

**For understanding the whole work flow, please refer to this documentation first: [Payzero OpenAPI Cookbook](cookbook.md)**

**为了更好的了解业务流程，请先阅读: [Payzero OpenAPI Cookbook](./cookbook.md)** 

### 文件修改记录 ###
|文件版本|修改时间|修改人|修改内容|
|:--|:--|:--|:--|
|v1.10| 2018-10-30| Raph| 增加1.4中关于店铺提交后，自动分配账户的场景说明；增加2.1,2.2中关于其对应未审核通过的店铺相关收款账号入账时，对应如何发送预入账与实际入账通知的说明; 修正3.4中部分typo|
|v1.11| 2018-11-23 | Raph| 增加了client自主revoke access_token的方法; 增加了对创建/编辑商户和店铺接口的部分说明 |
|v1.12|2018-12-03| Raph| 交易查询接口调整，获取对账文件接口调整，文档部分用词调整等(统一调整 "技术合作伙伴/调用方/支付合作伙伴" 为 "合作伙伴") |

## 技术综述 ##

PayZero OpenAPI整体采用RESTful API的风格，以json格式进行数据的传递。

### 请求格式说明 ###

合作伙伴需先将服务器ip提供给PayZero以确保其在api调用的白名单中(测试环境除外)，PayZero将为合作伙伴分配ClientId及ClientSecret，这些信息将在之后用于接口调用。

本文所涉及的功能接口分为主动调用和被动接收通知两种形式，对于合作伙伴被动接收的通知类型的消息，合作伙伴需要提供其服务器处理PayZero异步通知消息的url，并在该url实现处理消息的逻辑。而由合作伙伴主动调用的接口，无论是何种HTTP method，其HTTP HEADER中必须加入OAuth2风格的Http header属性:

~~~
Authorization: Bearer xxxxxxxxxxx
~~~

其中xxxxxxx为access\_token, access\_token的获得是通过调用获取access\_token接口来取得，每两个小时过期一次。

### 返回值说明 ###

调用接口时的返回HTTP STATUS CODE遵循HTTP返回值的通用定义，常见HTTP返回值例如:

|HTTP STATUS CODE|说明|
|:--:|:--|
|200|SUCCESS, 调用成功(仅表示api调用成功，业务是否成功需查看返回的HTTP BODY)|
|400|BAD REQUEST, 通常用于请求的参数不正确|
|405|METHOD NOT ALLOWED, HTTP方法使用错误，例如错误的使用POST调用了一个GET方法|
|401|UNAUTHORIZED, access_token不正确|
|403|FORBIDDEN, 权限不足|
|404|NOT FOUND, 不存在的接口url|
|500|INTERNAL SERVER ERROR, 服务器内部错误|

调用方应先根据HTTP STATUS CODE作为判断，过滤所有HTTP返回值即不成功的情况。

除获取access_token的接口外，所有后续业务调用api，在HTTP正常调用成功即返回200的情况下，返回的HTTP BODY消息体内容为统一的JSON格式内容如下。

|字段名称|参数|类型|说明|
|:--:|:--:|:--|:--|
|业务层面是否正常完成| success | Boolean | 可以此字段作为标准判定业务调用是否完成，当success为false时，可以从errorMsg读取到错误原因等，errorCode作为参考便于向Payzero反馈问题 |
|业务错误代码| errorCode | String | |
|业务错误信息| errorMsg | String | |
|返回数据 | data | JSON | 具体返回的业务数据，本文后续所有api在"response"部分所述内容，为简便均表示的是data部分的内容 |

一个典型的业务成功的返回例子如下:

~~~
{
    "success": true,
    "errorMsg": null,
    "errorCode": null,
    "data": {
        "merchant_id": "MI20190417223757334",
        "merchant_status": "PENDING",
        "account_provider_code": "icbc"
    }
}
~~~

一个典型的业务失败的返回例子如下:

~~~
{
    "success": false,
    "errorMsg": "参数 individual_name 未正确设置。",
    "errorCode": "30001",
    "data": null
}
~~~




## 接口介绍##

### 0. 基础设施相关接口###
#### 0.1a 获取access_token####
* url: {payzero\_api\_url}/api/auth/oauth/token
* method: POST
* Authorization: 使用Basic Authorization, 商户名为商户的clientId, 密码为商户的clientSecret
* query parameter: 

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|授权类型| grant_type| client_credentials| 固定类型client_credentials |

* POSTMAN调用示例:
![](resource/access_token_img.png)

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|access_token|access_token| "4f4c569b-2a31-4cf1-99c7-96dcf4abe9ca"|
|token类型| toke type| "bearer" | |
|过期时间(单位/秒)|expires_in| 692| 第一次申请为2小时(7200)，重新调用授权不会刷新过期token或过期时间 |
|scope| scope | "merchant transaction object" | 权限范围 |
~~~
{
    "access_token": "4f4c569b-2a31-4cf1-99c7-96dcf4abe9ca",
    "token_type": "bearer",
    "expires_in": 692,
    "scope": "merchant transaction object"
}
~~~

#### 0.1b 主动Revoke access_token####
由于主动调用0.1a中的方法并不会刷新access\_token，本方法提供给各合作伙伴强制revoke access\_token的方法，以已授权的access\_token作为Authorization Header中的参数并调用该方法之后，原先的access\_token将被清空。

* url: {payzero\_api\_url}/api/auth/token/revoke
* method: GET
* Header: Bearer {{0.1a中所获取的access_token}}

* POSTMAN调用示例:
![](resource/revoke_token.png)

* response: 该api返回的data为String，只需用根级别的"success" flag判定是否调用成功即可。

	
#### 0.2 上传文件####
上传文件为独立api，上传成功后系统将返回文件在系统中的文件id(object\_id)，本文后续多个api接口在涉及文件传输时均需提供文件id，这些id都是由文件上传接口返回，接口同时会返回该文件的临时访问的url，该url的有效期为1天。1天内可通过该url访问原文件，之后需通过object_id进行调用接口获取文件。

* url: {payzero\_api\_url}/object-service/upload
* method: POST
* request body parameter

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|文件内容| file | | multipart form-data|
|所有者| owner | | 此参数为非必填，不需要填写 |
|文件名| file_name| "example" | 若希望之后通过接口下载该文件时显示为其他文件名，可以填写此项目 |

* 调用http request示例:

~~~
POST /api/object-service/upload HTTP/1.1
Host: 127.0.0.1:16000
Authorization: Bearer 46c0e3be-4950-485e-b3ac-905c2fe2fca1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Cache-Control: no-cache

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="address.png"
Content-Type: image/png


------WebKitFormBoundary7MA4YWxkTrZu0gW--
~~~

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|文件id|object_id| aa22e541-6417-4aa4-827a-8001b55c1989 | |
|文件url| object_url | http://dev-payzero-openapi-object-bucket.... | |

~~~
{
	"object_id": "aa22e541-6417-4aa4-827a-8001b55c1989",
	"object_url": "http://dev-payzero-openapi-object-bucket.oss-cn-hangzhou.aliyuncs.com/
	aa22e541-6417-4aa4-827a-8001b55c1989.png?
	Expires=1514355644&OSSAccessKeyId=LTAIQ7UeAsp1BUlP&
	Signature=kSuQDV3gordjmakCXlhCzIQnPGI%3D"
   
}
~~~ 

* POSTMAN调用示例
![](resource/upload_image_ex.png)

#### 0.3 获取文件url####
上传文件后，会获得文件临时访问的url和object\_id，该访问url有过期时间，一定时间后（一般情况下为1天）若想再次获得文件的访问，需通过接口请求。

* url: {payzero\_api\_url}/object-service/get
* method: GET
* param parameter

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|文件id| id | aa22e541-6417-4aa4-827a-8001b55c1989 | |

* 请求示例:

~~~
http://127.0.0.1:16000/api/object-service/get?id=f7da2416-37bb-4d5a-a4b8-919e33aaa8e2
~~~

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|文件id|object_id| aa22e541-6417-4aa4-827a-8001b55c1989 | |
|文件url| object_url | http://dev-payzero-openapi-object-bucket.... | |

~~~
{
        "object_id": "f7da2416-37bb-4d5a-a4b8-919e33aaa8e2",
        "object_url": "http://dev-payzero-openapi-object-bucket.oss-cn-hangzhou.aliyuncs.com/
        f7da2416-37bb-4d5a-a4b8-919e33aaa8e2.png?
        Expires=1514359977&OSSAccessKeyId=LTAIQ7UeAsp1BUlP&
        Signature=72F%2FvHf0iF3sZ%2BDpfg6pex3IcE0%3D"
    }
~~~

#### 0.4 配置接收通知服务器####
合作伙伴可以配置自己接收平台通知的url，和签名所需的token，是否明文发送消息等参数。若需要密文模式，可设置EncodingAESKey参数。

* url: {payzero\_api\_url}/notification-service/setup
* method: POST
* request body parameter

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|接受通知url| notify_url | http://www.test.com/receive_notify | 用于合作伙伴接收payzero所发送的通知|
|token | token | "123" | 该token为签名会使用到的token，用于生成SHA1签名，由字母+数字组成，合作伙伴可任意设置 |
|消息体模式 | encoding_mode | 1 | 0-明文模式， 1-密文模式 |
|消息体加密AES Key| encoding\_aes\_key | "e8C5GyPbDatFUfRMDPESWc" | 密文加密时会使用到的AES Key, 请设置一个22位的key值，由大小写字母及数字组成 |

* 请求示例:

~~~
{
	"token": "123",
	"encoding_mode": 1,
	"encoding_aes_key": "e8C5GyPbDatFUfRMDPESWc",
	"notify_url": "http://www.test.com/receive_notify"
}
~~~

* response: 若请求成功, 返回值为String "OK" 

---


### 1. 商户创建及配置相关接口###
"商户"在本文中是指使用PayZero账户体系的个人或企业。

商户功能相关的API提供了对商户进行操作的基本接口，合作伙伴通过对接这些接口，能够实现商户的创建、修改、查询等基本操作，以实现自身的业务逻辑。

#### 1.0 获取当前合作的账户提供方####
获取合作伙伴当前所能对接的账户提供方列表，及其所能够支持的币种、支持的商务渠道、电子商务平台等信息。

* url: {payzero\_api\_url}/merchant-service/merchant/aplist
* method: GET
* query param: N/A

* response

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|账户提供方code| account\_provider\_code| "onerway" | 账户提供方code |
|支持币种| supp\_currency| "USD, EUR" | 该账户提供方所支持的币种，以","为分隔符 |
|支付平台| supp\_channel| "amazon\_us, amazon\_jp"| 该账户提供方所支持的电商平台、服务平台，以","为分隔符 |


#### 1.1 创建/更新商户####
本接口可用于商户的创建及编辑，当请求体不包含参数"merchant\_id"时，系统判定该请求为一个创建商户的请求。当请求体包含参数"merchant\_id"时，系统判定该请求为对传入的"merchant\_id"所对应商户的一次编辑请求。

* url: {payzero\_api\_url}/merchant-service/merchant
* method: POST
* request body parameter: 

|字段名称|参数名|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id | |用于已创建的商户信息更新，若为创建新商户，此处留空即可 |
|商户类型|merchant_type| "ENTERPRISE" | merchant_type可以为"INDIVIDUAL"个人商户或 “ENTERPRISE"企业商户 |
|国家地区|country| "CN" | 可选项参见附表 A.4 |
|个人姓名|individual_name| | 商户类型为个人商户时填写个人姓名，商户类型为企业商户时留空|
|个人姓名(英文)|individual\_name\_en | | 非中国大陆个人可填写 |
|个人身份证号| individual\_identity\_number |  | 商户类型为个人商户时填写身份证号码，商户类型为企业商户商户时留空|
|个人身份证正面| individual\_id_image1 |  | 身份证正面照片图片id，企业商户留空|
|个人身份证反面| individual\_id_image2|  | 身份证反面照片图片id，企业商户留空|
|个人手持身份证照片| individual\_id_image3| | 手持身份证照片图片id，企业商户留空|
|企业名称|enterprise_name| "上海汇萃商贸有限公司" | 商户类型为企业商户时填写企业名称，商户类型为个人商户时留空|
|企业名称(英文)|enterprise\_name\_en|  | 非中国大陆企业可填写 长度为2-64位空格或英文字母 |
|营业执照号（香港则为公司注册证书编号）| enterprise_id | "123456789ABC" | 商户类型为中国大陆企业商户时填写企业营业执照号，中国香港企业商户时填写公司注册数编号；商户类型为个人商户时留空|
|营业执照照片（香港则为公司注册证书照片）| enterprise\_licence\_image1| | 中国大陆企业营业执照图片id，中国香港企业公司注册证书图片id，个人商户留空|
|法人姓名|legal\_person_name| "张三" | 商户类型为个人商户时留空|
|法人身份证号|legal\_person\_identity\_number| "310113198610032413" | 商户类型为个人商户时留空|
|法人身份证正面| legal\_person\_id\_image1 | | 法人身份证正面照片图片id，个人商户留空|
|法人身份证反面| legal\_person\_id\_image2 | | 法人身份证反面照片图片id，个人商户留空|
|企业进出口证书| int\_port\_licence | | 企业进出口证书照片图片id，个人商户留空, 非必填|
|企业开户许可证| open\_account\_licence\_image | | 企业开户许可证图片id, 个人商户留空, 非必填|
|公司章程| consititution_image | | 香港公司：公司章程图片id, 个人商户留空, 非必填|
|企业商业登记书 | register_image | | 香港公司：企业商业登记证，个人商户留空，非必填|
|地址| residence\_address | | 企业或个人地址 |
|地址证明 | residence\_address\_proof\_image | | 企业或个人地址的证明的照片，例如银行账单、水电账单、暂住证、政府信件等，应距离当前时间不超过三个月，非必填 |
|手机号| mobile | "+86 18512113934" | |
|拟营业币种| prop_currency | "USD,EUR" | 创建该商户时填写该商户拟主营业币种，该值会决定商户的主账户提供方（根据账户提供方所支持的币种是否满足用户提供出的拟主营业币种。若有多个则以逗号分隔） |
|主账户提供方| account\_provider\_code| "onerway" | 偏好的主账户提供方代码，当前可用的账户提供方可通过专门接口获取，该字段非必填，若留空时会自动根据prop\_currency的值，选择一家支持该币种的账户提供方作为该商户的主账户提供方 |


* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "MI20171227171617534" | 生成的商户id |
|商户状态| merchant_status| "PENDING" | "PENDING" - 未审核, "APPROVED" - 审核成功, "REJECTED" - 审核失败，"TO\_BE\_UPDATED" - 待更新。 初次创建均为"PENDING" - 未审核|
|主账户提供方|account\_provider\_code|"onerway"| 商户的主账户提供方code|

~~~
{
	"merchant_id" : "MI20171227171617534",
	"merchant_status" : "PENDING",
	"account_provider_code" : "onerway" 
}
~~~

#### 1.2 商户审核结果通知####
通过该方法，Payzero发送审核结果通知给合作伙伴。请注意平台上所有的由平台推送的消息通知的格式是统一的，均包含clientId, msgType, signature, timestamp, nonce, msgBody等字段，区别在于msgBody中所含的字段不同。

合作伙伴可以根据如下所述的消息签名机制，判断消息所携带的signature是否来源于payzero。
signature = SHA1.getSHA1(token, timestamp, nonce, msgBody) 其中token为合作伙伴自行保管的token(参考0.4“配置接收通知服务器”中的token参数)。SHA1的生成可参考如下代码（该代码也可向payzero技术人员索取，之后若无特殊说明，所有签名机制均和这里所述的签名机制一致)：

![](resource/sig.png)


* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|合作伙伴id | clientId | "allinpay" | 接受该信息的合作伙伴id |
|消息类型|msgType| "merchant\_status\_notify" | 商户审核结果的消息类型 |
|签名| signature| | 根据平台规则对消息体内容进行加密后的签名|
|时间戳 | timestamp | "1515601844" | |
|用于消息签名的随机字符串 | nonce | "123" | |
|消息体内容| message_content| "xxxxx" | message消息体内容，nested内容参加下表|

消息体内容:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "MI20171227171617534" |  |
|商户状态| merchant_status| "REJECTED" |  "PENDING" - 未审核, "APPROVED" - 审核成功, "TO\_BE\_UPDATED" - 待更新。 "REJECTED" - 审核失败。|
|状态说明| status_comments | "url无法访问" | 详细审核意见 | 

#### 1.3 查询可以使用的平台 (Deprecated) ####
通过该方法可以查询制定商户可以绑定店铺的平台，及每个平台下可开设的店铺数量

* url: {payzero\_api\_url}/available\_platforms
* method: GET
* query parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id| "MI20171227171617534" | |

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id| "87613" | |
|商户名称| merchant_name| "上海汇萃商贸有限公司" | | 
|可供选择的平台列表| available\_platforms\_list| [{"platform": "amazon\_us", "quota": 5}, {"platform": "amazon\_eu", "quota": 3}] | 目前仅支持amazon\_us和amazon\_eu，数组包含平台名称，以及在该平台上绑定店铺的剩余quota数量 |

~~~
{
	"merchant_id": "87613",
	"merchant_name": "上海汇萃商贸有限公司",
	"available_platforms_list": 
		[
			{
				"platform": "amazon\_us", 
				"quota": 5
			}, 
			{
				"platform": "amazon\_eu", 
				"quota": 3
			}
		] 
}
~~~

#### 1.4 创建/更新店铺 ####
本接口可用于店铺的创建及编辑，当请求体不包含参数"store\_id"时，系统判定该请求为一个创建店铺的请求。当请求体包含参数"store\_id"时，系统判定该请求为对传入的"store\_id"所对应店铺的一次编辑请求。

每个商户可以创建多家店铺。该接口用于商户创建或更新店铺，请注意创建店铺并不要求商户审核状态为通过，即审核中状态的商户也可发起创建店铺的API调用。

为提高商户的体验，合作伙伴可与其关联的账户提供方协商，要求Payzero将该合作伙伴与其账户提供方的相关参数配置为“自动分配账户”，在该情况下，若Payzero收到合作伙伴的创建店铺请求，会立即查看账户提供方的账户池内是否有剩余未分配的收款账户，若有：则立即通过"店铺分配收款账户通知(参见1.6)"将其自动分配给合作伙伴（若该账户提供方在Payzero处没有剩余未分配的账户，则需等待账户提供方人工进行账户分配并触发收款账户分配通知)。自动分配时：若为amazon\_us, amazon\_eu, amazon\_au平台，则自动使用store\_name字段作为账户名称；若为其他类型店铺，则自动使用pay\_account\_info作为账户名称。

* url: {payzero\_api\_url}/merchant-service/merchant/store
* method: POST
* request body parameter:



|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id | "ME20180104180700537" | |
|店铺id| store_id | 用于已创建的店铺进行信息更新，若为新创建店铺则留空即可 |
|店铺名称| store_name | "Amazing3C" | |
|平台通道| platform | "amazon_us" | 平台代码参见附录 |
|支持币种| currency | ['USD', 'CAD'] | 数组，币种代码见附录 |
|卖家编号| seller_id | "12345" | amazon商家访问amazonMWS所需字段，非必填 |
|accessKey| access_key | "123abc" | amazon商家访问amazonMWS所需字段，非必填 |
|secret| secret | "123123abc" | amazon商家用于访问amazonMWS所需字段，非必填 |
|MWSAuthToken| mws\_token | "amazon.mws..." | amazon商家用于访问amazonMWS所需字段，非必填 |
|授权模式| access_mode | 0 | 亚马逊授权MWS模式, 0表示通过accessKey+secret形式授权， 1表示使用MWSAuthToken形式授权，非必填 |
|网址链接| web_url | "www.amazon.com/abc" | 商铺网站链接 |
|邮箱| email | "amazing3c@amazon.com" | 商户邮箱，非必填 |
|授权码| token |  | 对于各类需要提供访问token的电商平台，如ebay等，提供其对应的授权token，非必填 |
|API账号| pay\_account\_api\_usr | "seller_api1.test.com" | 对于各类需要提供api访问账号的钱包，例如paypal，提供其api访问账号,非必填 |
|API密码| pay\_account\_api\_pwd | "XNNQUSM3ASRHUX83" | 对于各类需要提供api访问账号的钱包，例如paypal，提供其api访问密码,非必填|
|API签名| pay\_account\_api\_sig | "AAh-gjnt..." | 对于各类需要提供api访问账号的钱包，例如paypal，提供其api访问签名,非必填|
|支付账户名称| pay\_account\_info | "张三" | 电商店铺关联的支付账户(钱包)信息，例如ebay店铺所关联的paypal信息，非必填 |
|所有权证明| ownership\_proof | "15c3b5a6-326b-4d1e-82e5..." | 这里放置的是通过对象上传接口所上传的文件的object\_id，例如在店铺为ebay店铺时，需提供paypal账户信息的截图，可使用对象上传接口上传完图片之后，将对象的id置于此处，非必填 |
|支付账户在其平台的ID| pay\_account\_id | JL76ED6DP5T6G | 商家所使用的支付账户的唯一标识ID，例如商家使用PayPal进行收款，则此处应填写PayPal的商家账号信息，非必填|
|销售类别| sales\_category| "时装" | 按实际销售类目填写，非必填但建议填写|
|支付账户持有人证件类型| owner\_identity\_proof\_type | "PERSONAL_ID" | 非必填，参见附录A.3 |
|支付账户持有人证件号| owner\_identity\_number | "310113198010101234" | 非必填，证件类型对应证件的证件号 |
|支付账户持有人证件图片| owner\_identity\_proof\_image | 图片id | 非必填 |
|主账户提供方| account\_provider\_code | "onerway" | 该店铺的主账户提供方，非必填，不填时由系统自行决定 |

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id | "ME20180104180700537" ||
|店铺id| store_id | "S20180109143015275"| |
|店铺名称| store_name | "Amazing3C"| |
|店铺状态| store_status | "PENDING" | "PENDING" - 未审核, "APPROVED" - 审核成功, "REJECTED" - 审核失败。店铺初始状态为未审核|
|主账户提供方|account\_provider\_code|"onerway"| 主账户提供方code|

~~~
	"merchant_id": "ME20180104180700537",
	"store_id": "S20180109143015275",
	"store_name": "Amazing3C",
	"store_status": "PENDING",
	"acount_provider_code": "onerway"
~~~

#### 1.5 店铺审核结果通知####
该方法为平台推送通知类型，当店铺审核完成时，会发送通知给合作伙伴，告知店铺的审核结果。注意：店铺的收款账号生成时，将另外推送通知（参见1.6“分配收款账号通知”）。

* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|合作伙伴id | clientId | "allinpay" | 接受该信息的合作伙伴id |
|消息类型|msgType| "store\_status\_notify" | 店铺审核结果的消息类型 |
|签名| signature| | 根据平台规则对消息体内容进行加密后的签名|
|时间戳 | timestamp | "1515601844" | |
|随机字符串 | nonce | "123" | |
|消息体内容| message_content| "xxxxx" | message消息体内容，nested内容参加下表|

消息体内容:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "ME20180104180700537" |  |
|店铺id|store_id| "S20180109143015275" | |
|店铺状态| store_status| "APPROVED" |  "PENDING" - 未审核, "APPROVED" - 审核成功, "REJECTED" - 审核失败。|
|状态说明|status_comments|"审核通过" | |



#### 1.6 分配收款账号通知####
该方法为平台推送通知类型，当店铺所关联的虚拟收款账号生成时，系统会向合作伙伴推送这些收款账号的具体信息。

* method: POST
* request body parameter:


|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|合作伙伴id | clientId | "allinpay" | 接受该信息的合作伙伴id |
|消息类型|msgType| "account\_status\_notify" | 账号审核结果的消息类型 |
|签名| signature| | 根据平台规则对消息体内容进行加密后的签名|
|时间戳 | timestamp | "1515601844" | |
|随机字符串 | nonce | "123" | |
|消息体内容| message_content| "xxxxx" | message消息体内容，nested内容参加下表|

消息体内容:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "ME20180104180700537" |  |
|店铺id|store_id| "S20180109143015275" | |
|收款账号id| account_id | "A20180109145013252" | 收款账号payZero account id |
|收款账号银行所在地| bank_location | "USA" | |
|收款账号币种| currency | "USD" | |
|收款账号名称| account\_holder\_name| "Amazing3C" | |
|收款账号银行| bank_name | "Citi Bank" | |
|收款银行账号| bank\_account\_number | "100018316" | | 
|收款账号aba| aba | "071006651" | "USA"美国地区专用，表示银行唯一编号 |
|收款账号iban| iban | |  |
|收款账号bic| bic | | BIC或SWIFT CODE |
|收款账号bsb | bsb | | "AUS"澳洲地区专用，表示银行支行唯一编号 |
|收款账号银行代码 | bank_code | | "JP"地区专用 |
|收款账号分行代码 | branch_number | | "JP"地区专用 | 
|账户类别| type\_of\_account| "Saving" | 账户类型 "Saving"为储蓄账户, "Checking"为支票账户 |
|账户提供方| account\_provider\_code | "onerway" | |




#### 1.7 创建/更新提现账号####
<font color="red">请注意: 若合作伙伴仅使用账户提供方的账户进行资金收集，之后由合作伙伴自行管理资金，处理商户提现等，则无需对接此接口。但部分账户提供方若要求合作伙伴提供至少一个提现账户作为商户kyc审核标准，则依然可通过本接口提供商户的提现账户信息。</font>

通过本接口，可以为商户关联其对应的提现账号

* url: {payzero\_api\_url}/merchant-service/merchant/withdraw_account
* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id | "ME20180104180700537" | |
|账户类型| withdraw\_account\_type | "INDIVIDUAL" | 提现账户类型可以是对公账户或个人账户, "INDIVIDUAL" 个人账户/ "ENTERPRISE" 对公账户 |
|银行名称| bank_name | "中国银行" | |
|支行名称| branch_name | "上海市梅花路支行" | |
|支行省市| branch\_province\_city | "上海-上海" | 非必填|
|支行地址| branch_address| "上海市浦东新区梅花路111号"|
|提现账户名| account_name | "张三" | |
|提现账户英文名 | account\_name\_en | | 非必填 |
|提现账户swift code| swift_code | | 提现账户为国外账户时必填  |
|银行卡号| account_number | "6210100112344321" |  |
|账号币种| currency | "CNY" | |
|联系人手机号| contact_number| "18512113934" | |

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id | "ME20180104180700537" | | 
|提现账户id| withdraw\_account\_id | "WA20180109192320404" | |
|提现账户名| account_name | "张三" | |
|银行名称| bank_name | "中国银行" | |
|银行卡号| account_number | "6210100112344321" | |
|账号币种| currency | "CNY" | |

~~~
{
    "merchant_id": "ME20180104180700537",
    "withdraw_account_id": "WA20180109192320404",
    "account_name": "张三",
    "bank_name": "中国银行",
    "account_number": "6210100112344321",
    "currency": "CNY"
}
~~~ 

#### 1.8 提现账号审核结果通知 ####
若合作伙伴无需对接提现账户的提交接口，则可直接跳过本段。
该方法为平台推送通知类型，当提现账号审核完成时，系统会向合作伙伴推送审核结果通知。

* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|合作伙伴id | clientId | "allinpay" | 接受该信息的合作伙伴id |
|消息类型|msgType| "withdraw\_account\_status\_notify" | 提现账户审核结果的消息类型 |
|签名| signature| | 根据平台规则对消息体内容进行加密后的签名|
|时间戳 | timestamp | "1515601844" | |
|随机字符串 | nonce | "123" | |
|消息体内容| message_content| "xxxxx" | message消息体内容，nested内容参加下表|

消息体内容:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "ME20180104180700537" |  |
|提现账户id|withdraw\_account\_id| "WA20180109192320404" | |
|提现账户状态| withdraw\_account\_status| "APPROVED" |  "PENDING" - 未审核, "APPROVED" - 审核成功, "REJECTED" - 审核失败。|
|状态说明| status_comments |  | |

---
### 2. 商户提现流程相关接口 ###

#### 2.1 预入账通知 ####
该方法为平台推送通知类型，当有资金进入账户时，会发送预入账通知给相应合作伙伴，注意，收到预入账通知表明该资金计入预入账余额。

若店铺未审核通过时，商户就错误地将为该店铺分配的收款账户绑定至各电商平台进行收款并实际发生收款，此时Payzero依旧会发送预入账通知至合作伙伴，但由于店铺的审核状态尚未通过，则Payzero会暂时冻结该笔交易（设置交易状态为INCOME_PAUSED），即不发送“实际入账通知”(参见2.2)。直到此店铺审核通过时，才会自动触发后续的“实际入账通知”。

* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|合作伙伴id | clientId | "allinpay" | 接受该信息的合作伙伴id |
|消息类型|msgType| "pre\_income\_notify" | 预入账通知的消息类型 |
|签名| signature| | 根据平台规则对消息体内容进行加密后的签名|
|时间戳 | timestamp | "1515601844" | |
|随机字符串 | nonce | "123" | |
|消息体内容| message_content| "xxxxx" | message消息体内容，nested内容参加下表|

消息体内容:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "ME20180104180700537" |  |
|店铺id| store_id | "S20180109143015275" | |
|账户id| account_id | "A20180109145013252" | |
|交易流水号| transaction_id | T20180213172632258 | |
|币种| currency | "USD" | |
|额度| amount | "10000" | |
|付款方名称| opp_name | | 可能为空，表示该入账的交易对手方 |
|预入账时间 | entry_time | 1516348181594 | 预入账的时间戳|
|入账记录时间| posted_time | | 对于预入账通知，此字段可忽略|
|账户提供方 | account\_provider\_code | "onerway" | 入账渠道代码 |

#### 2.2 实际入账通知 ####
该方法为平台推送通知类型，当有资金完成资金入账后（即由预入账资金进入可提现状态），即会发送入账通知给合作伙伴。

若店铺尚未审核通过时，即有入账交易进入其关联账户，则该笔交易只会触发“预入账通知”(参见2.1)。待该店铺完成审核并通过之后，会自动触发实际入账通知。

* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|合作伙伴id | clientId | "allinpay" | 接受该信息的合作伙伴id |
|消息类型|msgType| "actual\_income\_notify" | 实际入账通知的消息类型 |
|签名| signature| | 根据平台规则对消息体内容进行加密后的签名|
|时间戳 | timestamp | "1515601844" | |
|随机字符串 | nonce | "123" | |
|消息体内容| message_content| "xxxxx" | message消息体内容，nested内容参加下表|

消息体内容:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "ME20180104180700537" |  |
|店铺id| store_id | "S20180109143015275" | |
|账户id| account_id | "A20180109145013252" | |
|交易流水号| transaction_id | T20180213172632258 | |
|币种| currency | "USD" | |
|额度| amount | "10000" | |
|付款方名称| opp_name | | 可能为空，表示该入账的交易对手方 |
|预入账时间 | entry_time | 1516348181594 | 该笔入账的预入账的时间戳|
|入账记录时间| posted_time | | 该笔入账的入账时间戳|
|账户提供方 | account\_provider\_code | "onerway" | 入账渠道代码 |

#### 2.3 入账失败通知####

该方法为平台推送通知类型，当有预入账资金审核失败时，即会发送入账失败通知给合作伙伴。

<font color="red">请注意：本通知的处理需根据合作伙伴与账户提供方的实际运营形态具体分析，请在对接时与Payzero技术人员进行讨论。</font>

* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|合作伙伴id | clientId | "allinpay" | 接受该信息的合作伙伴id |
|消息类型|msgType| "income\_fail\_notify" | 实际入账通知的消息类型 |
|签名| signature| | 根据平台规则对消息体内容进行加密后的签名|
|时间戳 | timestamp | "1515601844" | |
|随机字符串 | nonce | "123" | |
|消息体内容| message_content| "xxxxx" | message消息体内容，nested内容参加下表|

消息体内容:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "ME20180104180700537" |  |
|店铺id| store_id | "S20180109143015275" | |
|账户id| account_id | "A20180109145013252" | |
|交易流水号| transaction_id | T20180213172632258 | |
|币种| currency | "USD" | |
|额度| amount | "10000" | |
|付款方名称| opp_name | | 可能为空，表示该入账的交易对手方|
|预入账时间 | entry_time | 1516348181594 | 该笔入账的预入账的时间戳|
|入账失败记录时间| posted_time | | 该笔入账失败发生时间戳|
|账户提供方 | account\_provider\_code | "onerway" | 入账渠道代码 |



#### 2.4 入账结果更新接口####

<font color="red" >请注意: 本接口为特殊情况下的定制接口，若技术对接时无特别通知，无需对接本接口。</font>

在合作伙伴实际资金入账对账完成之后，可将处理的结果主动更新至平台，例如某笔入账是否成功，是否失败等。

* url: {payzero\_api\_url}/transaction-service/income\_wrap\_up
* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|需更新状态的交易列表|transaction\_list |  | 注意，该字段为数组，数组的每个元素的属性值见下|
|元素属性值|
|交易ID| transaction_id | "T20180213172632258" | |
|交易状态| status_code | "SUCCESS" | 交易在合作方的入账若成功，则应发送状态 "SUCCESS", 若合作方不准备入账该笔交易，或入账最终失败且不会再次入账，则可传递状态 "FAIL" |

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|需更新状态的交易列表|transaction\_list |  | 注意，该字段为数组，数组的每个元素的属性值见下|
|元素属性值|
|交易ID| transaction_id | "T20180213172632258" | |
|交易下发状态| status_code | "SUCCESS" | 此字段为传入的值，会不作修改的返回 |
|状态更新结果| is\_update\_competed | "COMPLETED" | 表示该笔交易状态的更新是否完成，"COMPLETED"表示完成, "FAIL"表示状态更新失败。（注意，这里是指状态更新这个动作的成功/失败） |
|信息| message | "更新状态完成" | 状态更新成功/失败的文字描述 |

~~~
{
    "transaction_list": [
      {
        "transaction_id": "T20180213172632258",
        "status_code": "FAIL",
        "is_update_completed": "COMPLETED",
        "message": "更新状态完成"
      },
      {
        "transaction_id": "T20180213172632258",
        "status_code": "SUCCESS",
        "is_update_completed": "COMPLETED",
        "message": "更新状态完成"
      }
    ]
  }
~~~


#### 2.5 创建提现申请接口####

<font color="red">请注意: 若合作伙伴仅使用账户提供方的账户进行资金收集，资金收集后均由合作伙伴自行管理资金，则无需对接本接口。</font>

合作伙伴与账户提供方的合作模式若为由账户提供方进行实际资金的管理，则合作伙伴每发起一笔提现，均应将提现的详细信息告知本接口，进而推送给实际管理资金的账户提供方。

* url: {payzero\_api\_url}/transaction-service/withdraw
* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id	|merchant_id| "87613" ||
|提现来源账号| account_id | "A876131_1" | |
|提现账号| withdraw\_account\_id | "W87613_1" | |
|提现币种| target\_currency | "CNY" | |
|提现金额| withdraw\_amount| 100000 | 注意：以提现来源账号币种为准！例如来源账号的币种为USD,则即使target\_currency为CNY，提现金额仍应填写USD的额度|
|是否需要下发| direct_withdraw| 0 | 对可以自行下发的支付公司，填写0表示不需要下发到提现账户，而是统一打给支付公司，由支付公司进行下发|
|关联交易组| related_transactions | [T111111, T222222] | 部分账户提供方可能要求

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id | "87613"| |
|交易id| transaction_id | "8174123455" | |


#### 2.6 提现处理失败通知####
<font color="red">请注意: 与2.5“创建提现申请接口“的情况一致，若合作伙伴与账户提供方的合作模式为仅使用账户提供方的账户进行资金归集，实际资金由合作伙伴自行管理，则无需处理本通知。</font>

在提交提现请求后，会立即收到一个同步的response，给予该提现所对应的transaction_id等，Payzero服务器端会异步的进行验证和其他业务操作，若中途出现任何问题导致失败，则Payzero会发送提现失败通知给合作伙伴，让合作伙伴可以及时的、尽早的进行处理。

* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|合作伙伴id | clientId | "allinpay" | 接受该信息的合作伙伴id |
|消息类型|msgType| "withdraw\_failure\_notify" | 提现请求失败的消息类型 |
|签名| signature| | 根据平台规则对消息体内容进行加密后的签名|
|时间戳 | timestamp | "1515601844" | |
|随机字符串 | nonce | "123" | |
|消息体内容| message_content| "xxxxx" | message消息体内容，nested内容参加下表|

消息体内容:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "87613" |  |
|交易流水号| transaction_id | T20180213172632258 | |
|失败备注|comments | 提现数据验证失败，提现失败 | |


#### 2.7 提现下发结果更新接口 ####

<font color="red">请注意: 与2.5“创建提现申请接口“的情况一致，若合作伙伴与账户提供方的合作模式为仅使用账户提供方的账户进行资金归集，实际资金由合作伙伴自行管理，则无需对接本接口。</font>

在合作伙伴处理完提现之后，可将处理的结果主动更新至平台，例如某笔提现是否成功，是否失败等。注意，平台这里不搜集具体的下发信息，仅用于tracking提现下发是否顺利完成。

* url: {payzero\_api\_url}/transaction-service/withdraw\_wrap\_up
* method: POST
* request body parameter:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|需更新状态的交易列表|transaction\_list |  | 注意，该字段为数组，数组的每个元素的属性值见下|
|元素属性值|
|交易ID| transaction_id | "T20180213172632258" | |
|交易下发状态| status_code | "SUCCESS" | 交易在合作方的最后下发步骤中若成功，则应发送状态 "SUCCESS", 若合作方不准备下发该笔交易，或下发最终失败且不会再次下发，则可传递状态 "FAIL" |

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|需更新状态的交易列表|transaction\_list |  | 注意，该字段为数组，数组的每个元素的属性值见下|
|元素属性值|
|交易ID| transaction_id | "T20180213172632258" | |
|交易下发状态| status_code | "SUCCESS" | 此字段为传入的值，会不作修改的返回 |
|状态更新结果| is\_update\_competed | "COMPLETED" | 表示该笔交易状态的更新是否完成，"COMPLETED"表示完成, "FAIL"表示状态更新失败。（注意，这里是指状态更新这个动作的成功/失败） |
|信息| message | "更新状态完成" | 状态更新成功/失败的文字描述 |

~~~
{
    "transaction_list": [
      {
        "transaction_id": "T20180213172632258",
        "status_code": "FAIL",
        "is_update_completed": "COMPLETED",
        "message": "更新状态完成"
      },
      {
        "transaction_id": "T20180213172632258",
        "status_code": "SUCCESS",
        "is_update_completed": "COMPLETED",
        "message": "更新状态完成"
      }
    ]
  }
~~~

#### 2.8 提现批次生成接口####
内部用
#### 2.9 提现批次汇率更新通知####
内部用
#### 2.10 提现批次资金到账通知####
内部用

---
### 3. 商户查询相关接口###
#### 3.1 商户信息查询接口####

用于查询指定商户的详细信息

* url: {payzero\_api\_url}/merchant-service/merchant
* method: GET
* query body parameter: 

|字段名称|参数|是否必填|例子|说明|
|:--|:--|:--|:--|:--|
|商户id| merchant_id | 是 |"MI20180109160941764" | | 

* response:

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id | "MI20180109160941764" | | 
|商户状态| merchant_status | "PENDING" | |
|商户类型|merchant_type| "ENTERPRISE" | merchant\_type可以为"INDIVIDUAL"个人商户或 “ENTERPRISE"企业商户 |
|国家|country| "CN" | 中国 - CN, 香港 - HK|
|个人姓名|individual_name| | 商户类型为个人商户时填写个人姓名，商户类型为企业商户时留空|
|个人姓名（英文) | individual\_name\_en | | |
|个人身份证号| individual\_identity_number |  | 商户类型为个人商户时填写身份证号码，商户类型为企业商户时留空|
|个人身份证正面| individual\_id_image1 |  | 身份证正面照片图片url，企业商户留空|
|个人身份证反面| individual\_id_image2|  | 身份证反面照片图片url, 企业商户留空|
|个人手持身份证照片| individual\_id_image3| | 手持身份证照片图片url, 企业商户留空|
|企业名称|enterprise_name| "上海汇萃商贸有限公司" | 商户类型为企业商户时填写企业名称，商户类型为个人商户时留空|
|企业名称(英文)| enterprise\_name\_en | | |
|营业执照号| enterprise_id | 123456789ABC | 商户类型为企业商户时填写企业营业执照号，商户类型为个人商户时留空|
|营业执照照片| enterprise\_licence\_image1| | 营业执照图片url，个人商户留空|
|法人姓名|legal\_person_name| "张三" | 商户类型为个人商户时留空|
|法人身份证号|legal\_person\_identity\_number| 310113198610032413| 商户类型为个人商户时留空|
|法人身份证正面| legal\_person\_id\_image1 | | 法人身份证正面照片图片url，个人商户留空|
|法人身份证反面| legal\_person\_id\_image2 | | 法人身份证反面照片图片url，个人商户留空|
|企业进出口证书| int\_port\_licence\_image | | 企业进出口证书照片图片url，个人商户留空|
|企业开户许可证| open\_account\_licence\_image | | 企业开户许可证图片id，个人商户留空，非必填|
|公司章程| consititution_image | | 香港公司：公司章程图片id，个人商户留空，非必填|
|企业商业登记书 | register_image | | 香港公司：企业商业登记证，个人商户留空，非必填|
|地址| residence\_address | | 企业或个人地址 |
|地址证明 | residence\_address\_proof\_image | | 企业或个人地址的证明的照片，例如银行账单、水电账单、暂住证、政府信件等，应距离当前时间不超过三个月 |
|手机号| mobile | "+86 18512113934" | |

~~~
{
    "merchant_id": "MI20180109160941764",
    "merchant_status": "PENDING",
    "merchant_type": "INDIVIDUAL",
    "country": "CN",
    "individual_name": "张三",
    "individual_name_en": null,
    "individual_identity_number": "410327199005133077",
    "individual_id_image1": "ca0bc7ac-96c5-4b11-bfda-ef1fb67c881d",
    "individual_id_image2": "46e2f0d8-c66c-4935-8c7e-78f9ad60eced",
    "individual_id_image3": "81491e41-dbb1-420e-b9c5-ada3dec3a1ad",
    "enterprise_name": null,
    "enterprise_name_en": null,
    "enterprise_id": null,
    "enterprise_licence_image1": null,
    "legal_person_name": null,
    "legal_person_identity_number": null,
    "legal_person_id_image1": null,
    "legal_person_id_image2": null,
    "int_port_licence": null,
    "residence_address": "湖北省黄冈市CN测试地址",
    "residence_address_proof_image": null,
    "open_account_licence_image": null,
    "consititution_image": null,
    "register_image": null,
    "mobile": "15311439451"
}
~~~

#### 3.2 店铺信息查询####
用于查询商户的店铺，若参数传入店铺id，则只显示该店铺的信息

* url: {payzero\_api\_url}/merchant-service/merchant/store
* method: GET
* query body parameter: 

|字段名称|参数|是否必填|例子|说明|
|:--|:--|:--|:--|:--|
|商户id| merchant_id | 是 |"MI20180109160941764" | | 
|店铺id| store_id | 否 |"S87613_1" | |
|平台通道| platform | 否 | "amazon_us" |  |

* response: 注意，返回值为数组，这里显示数组每个元素的attribute

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id | "MI20180109160941764" | |
|店铺id| store_id | "S87613_1 | |
|店铺状态| store_status| "PENDING" | "PENDING" - 未审核, "APPROVED" - 审核成功, "REJECTED" - 审核失败。店铺初始状态为未审核|
|店铺名称| store_name | "Amazing3C" | |
|平台通道| platform | "amazon_us" | |
|网址链接| web_url | "www.amazon.com/abc" | 商铺链接 |
|邮箱| email | "amazing3c@amazon.com" | 商户邮箱 |
|销售类目| sales_category| "时装" |商户销售类目 |

~~~
{
	[{
		"merchant_id": "MI20180109160941764",
		"store_id": "S87613_1",
		"store_status": "PENDING",
		"store_name": "Amazing3C",
		"platform": "amazon_us",
		"web_url": "www.amazon.com/abc",
		"email": "amazing3c@amazon.com"
	},...]
}
~~~

#### 3.3 账户信息及余额查询####
用于查询商户的账户，若传入参数店铺id，则只显示此商户此店铺的账户信息；若参数传入账户id，则只显示该账户的信息。

* url: {payzero\_api\_url}/transaction-service/account
* method: GET
* query body parameter: 

|字段名称|参数|是否必填|例子|说明|
|:--|:--|:--|:--|:--|
|商户id| merchant_id | 是 | "MI20181204141313372" | | 
|店铺id| store_id | 否 | "S20181204141453847" | |
|账户id| account_id | 否 | "A20181204142848287" | |

* response: 注意，返回值为数组，这里显示数组每个元素的attribute

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id|merchant_id| "MI20181204141313372" |  |
|店铺id|store_id| "S20181204141453847" | |
|收款账号id| account_id | "A20181204142848287" | 收款账号payZero account id |
|收款账号银行所在地| bank_location | "USA" | |
|收款账号币种| currency | "USD" | |
|收款账号名称| account\_holder\_name| "Amazing3C" | |
|收款账号银行| bank_name | "Citi Bank" | |
|收款银行账号| bank\_account\_number | "100018316" | | 
|收款账号aba| aba | "071006651" | "US"地区专用 |
|收款账号iban| iban | |  |
|收款账号bic| bic | | "EU"地区专用，等于SWIFT CODE |
|收款账号bsb | bsb | | "AUS"澳洲地区专用，表示银行支行唯一编号 |
|收款账号银行代码 | bank_code | | 部分国家地区使用 |
|收款账号分行代码 | branch_number | | 部分国家地区使用 | 
|账户类别| type\_of\_account| "Saving" | 部分国家地区使用 |
|预入账金额| pre\_income\_amount| "100000.00" | |
|可提现金额| available_amount| "2000000.00" | |
|提现中金额| processing_amount| "200000.00" | |
|账户提供方| account\_provider\_code | "onerway" | |

~~~
[{
    "merchant_id": "MI20180109160941764",
    "store_id": "S20180117162647312",
    "account_id": "A20180119163854673",
    "bank_location": "LUX",
    "currency": "USD",
    "account_holder_name": "TL_Test",
    "bank_name": "ONPEX S.A.",
    "bank_account_number": "888666777555",
    "aba": null,
    "iban": "654321",
    "bic": "123456",
    "bsb": "null",
    "bank_code": null,
    "branch_name": null,
    "type_of_account": null,
    "pre_income_amount": "0.00",
    "available_amount": "23750.00",
    "processing_amount": "6250.00"
  }...]
~~~

#### 3.4 交易记录查询####
用于查询商户的账户交易记录，若参数传入账户id或交易id，则只显示该账户的交易记录信息，或某条指定的交易信息

* url: {payzero\_api\_url}/transaction-service/account/transactions
* method: GET
* query body parameter: 

|字段名称|参数|是否必填|例子|说明|
|:--|:--|:--|:--|:--|
|商户id| merchant_id | 是 | "MI20181204141313372" | | 
|店铺id| store_id | 否 | "S20181204141453847" | |
|账户id| account_id | 否 | "A20181204142848287" | |
|交易id| transaction_id| 否 |"T201812041442487627" | |

* response: 注意，返回值为数组，这里显示数组每个元素的attribute

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|商户id| merchant_id | "MI20181204141313372" | |
|店铺id| store_id | "S20181204141453847" | |
|账户id| account_id | "A20181204142848287" | | 
|交易id| transaction_id | "T201812041442487627" | |
|账户提供方| account\_provider\_code | "novatti" |
|币种| currency_code | "USD" | |
|交易类型| transaction_type| "INCOME" | "INCOME"-到账, "WITHDRAW"-提现|
|交易状态| status| "INCOME\_POSTED" | "INCOME\_PENDIND" 预入账, "INCOME\_POSTED" 已实际到账, "INCOME\_PAUSED" 预入账暂停, "WITHDRAW\_PENDING" 提现中, "WITHDRAW\_POSTED" 提现完成 |
|交易金额| amount| "100000.00" |  |
|交易对手名| opp_name | "PayPal Inc."  | 例如该笔到账交易的交易对手为PayPal Inc，即PayPal打款的一笔入账，则交易对手为"PayPal Inc" |
|入账时间| entry_time | 1543905769000 | Timestamp，该字段表示在账户提供方处该笔交易的初次发生时间 |
|交易完成时间 | posted_time | 1543906097000 | Timestamp，表示该笔交易的最终完成时间。若交易为入账交易，则表示该笔交易的入账成功时间，若交易为提现交易，则表示该笔交易的最后提现成功时间。（P.S. 若由客户自行负责下发，则此时间戳表示该笔交易从账户提供方打出至合作伙伴的时间) |
|其他信息 | additional_info | | 预留字段 |

~~~
{[
	 {
      "transaction_id": "T201812041430086611",
      "transaction_type": "INCOME",
      "account_provider_code": "icbc",
      "account_id": "A20181204142848287",
      "merchant_id": "MI20181204141313372",
      "store_id": "S20181204141453847",
      "amount": "399.00",
      "currency_code": "USD",
      "status": "INCOME_POSTED",
      "additional_info": null,
      "opp_name": null,
      "entry_time": 1530115200000,
      "posted_time": 1543905051000
    }, 
    ...
]}
~~~
---
### 4. 对账文件相关接口###
#### 4.1 获取对账文件链接####
用于合作伙伴获取对账文件。

* url: {payzero\_api\_url}/transaction-service/recon/file
* method: GET
* query body parameter: 

|字段名称|参数|是否必填|例子|说明|
|:--|:--|:--|:--|:--|
|账户提供方 | account\_provider\_code | 是 | "onerway" | 合作伙伴与账户提供方进行对账，这里填写账户提供方代码 |
|对账文件类型| recon\_file\_type | 是 | 固定填写"I" | | 
|对账日期|recon\_date | 是 | "20180207" | 查询的对账文件日期， 使用yyyyMMdd格式|
|当日对账文件序号| sequential | 是 | "01" | 由于每天会进行两次对账，此处可以填写"01"或"02"|
|国家/地区| country | 否 | "CN" | 不同提现区域的对账文件是分开的，目前支持"HK"和"CN"两个区域的对账文件 | 
|对账币种| currency | 否 | "CNY" | 与账户提供方的对账币种 |

* response: 

|字段名称|参数|例子|说明|
|:--|:--|:--|:--|
|文件对象id| object_id | "15c3b5a6-326b-4d1e-82e5-54bce25920a9" | |
|文件url| object_url | "http://dev-payzero...Tmc%3D" | | 

~~~
{
    "object_id": "15c3b5a6-326b-4d1e-82e5-54bce25920a9",
    "object_url": "http://dev-payzero-openapi-object-  
    bucket.oss-cn-hang6-326b-4d1e-8...5Tmc%3D"
  }
~~~

---
### A. 附录###
#### A.1 系统支持币种####
|币种| 货币编码 |
|:--|:--|
|美元| USD |
|加拿大元 | CAD |
|欧元| EUR |
|英镑| GBP |
|日元| JPY |
|澳元| AUD |
|人民币| CNY |

#### A.2 系统支持平台通道####
|通道| 通道代码 |
|:--|:--|
|亚马逊美国| amazon_us|
|亚马逊欧洲| amazon_eu|
|亚马逊日本| amazon_jp|
|亚马逊澳洲| amazon_au|
|速卖通| aliexpress|
|Qoo10| qoo10|
|Ebay美国| ebay_us |
|独立站| other_site |
|房地产行业相关| real_estate|
|服务贸易独立站| service_site|


#### A.3 系统支持证件类型####
|证件类型| 证件代码 |
|:--|:--|
|身份证| PERSONAL_ID|
|护照| PASSPORT|
|营业执照| BUSINESS_LICENCE|

#### A.4 国家地区代码####
|国家/地区| 国家/地区代码|
|:--| :--|
|中国| CN |
|香港| HK |
|菲律宾| PH |