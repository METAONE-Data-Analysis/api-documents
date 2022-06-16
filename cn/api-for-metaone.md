# MetaOne数据平台对接用接口v0.1_20220427：

## 1、服务接入地址

- 测试环境：

  https://test-data.metaone.gg/sb/
  
- 正式环境：

  https://data.metaone.gg/sb/

## 2、接口

### 2.1、统一请求数据结构

#### 2.1.1、请求头（Request Headers）：

- Accept-Language/lang：国际化语言【使用ISO标准的locale。不区分大小写，不区分横杠"-"与下划线"_"】
  - 英文：en或 en-US、en_uk等
  - 中文：zh或zh-cn、zh-hans、zh-hant、zh-hk、zh-tw、zh-sg等
- Authorization：应用全局唯一的动态接口调用凭据，也就是通过获取token接口返回的token。开发者需要每日获取更新并进行妥善保存。

### 2.2、统一返回数据结构

```json
{
    "stateCode": 1, // 业务返回状态码
    "resultInfo": { // 业务返回主体内容
        "desc": "Success",  // 返回内容描述
        "data": {}  // 业务返回数据
    }
}
```

- stateCode：
  - 1：请求成功
  - 0：参数格式相关错误
  - -1：token无效或过期
  - -5：服务端错误异常
  - -1105：无效的AppId
  - -1101：应用被封锁
  - -102：密钥不正确
  - -101：找不到对应数据
- resultInfo.desc：
  - Success：请求成功
  - 其他：请求失败后的错误描述
- resultInfo.data：请求成功后的数据，注意，数据有可能为空

#### 日期相关参数格式：yyyy-MM-dd

## 3、API接口：

### 3.1、鉴权模块：

#### 3.1.1 获取应用调用接口凭证-token


- 功能说明：获取token，token需要作为下面所有接口的认证信息带到请求头`Authorization`中。每一个接入方需要使用自己的appId和密钥换取的token，请勿借用其他接入方的token。

- 请求方式：POST

- 请求地址：/metaoneDataLight/getToken

- 请求内容：

  ```json
  {
  	"appId": "06be07ff82754a718a96ef46d1b82b2c", // 应用ID，由metaone数据服为每一个接入方颁发
  	"secret": "7531b58338104dcd9e7eddcce96c4ae0" // 秘钥，由metaone数据服为每一个接入方颁发，注意：非常重要，不能泄露
  }
  ```

- 正确时响应内容：

  ```json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"token": "at_2a015a53d41548f5a6d7879b75a5fff3_gAP6sHl2xnh5WEWxgiFYUUfgXaPjtViY_mo",
          }
      }
  }
  ```

##### 应用调用接口凭证-token机制相关说明：

1. 建议基于微服务架构的开发者使用中控微服务统一获取和刷新token，其他业务逻辑微服务所使用的token均来自于该中控服务，不建议各自去getToken；
2. 目前token的有效期为24～48小时。中控服务每日固定时间通过计划任务等方式获取一次token，该token自getToken调用起算能够确保24小时内有效。在获取过程中，原有的token会在新token获取之后一段时间内仍然持续有效一小段时间，这保证了token有效期的平滑过渡；
3. token的有效时间可能会在未来有调整，所以中控服务除每日定时主动获取当天的token以外，建议提供被动刷新token的接口，这样便于业务服务器在API调用获知token已超时的情况下，可以触发token的刷新流程。
4. 每一个接入方都应该有自己的appId和secret。因此不同的接入方一定是使用各自的token去调用接口。数据服接口会直接将token解析为对应的接入方权限。

### 3.2 玩家活跃数据模块：

#### 3.2.1 玩家收益数据查询：

- 功能说明：查询玩家每日的收益数据(可指定游戏)

- 请求方式：POST

- 请求地址：/metaoneData/queryUserValueGain

- 请求头：

HeaderName | 是否必须 | HeaderValue说明
------- | ------- | -------
Authorization | 是 | 当前保存的token<br/>【token需要每日通过调用获取token接口获取并更新/保存起来】
lang | 否 | 语言，缺省值"en"英文，当前仅支持中文与英文 

- 请求内容：

  ```json
  {
  	"userId": "06be07ff82754a718a96ef46d1b82b2c", // 用户ID
  	"gameId": "7531b58338104dcd9e7eddcce96c4ae0", // 游戏ID。可选参数，缺省值为"all"
  	"startDate" : "2022-04-01", //查询范围的起始日期（为yyyy-MM-dd字符串格式）查询结果会包含这个日期
  	"endDate" : "2022-04-05", //查询范围的截止日期（为yyyy-MM-dd字符串格式
  }
  ```
	- 关于gameId参数的特别说明：

   如果要查询玩家在所有游戏的每日总收益，那么gameId可以留空或设置为"all"。查询单款游戏和查询所有游戏总和的情况返回结果的结构会有略微的差异。参见下方正确时响应内容。
  
	- 关于日期参数的特别说明：

 日期参数采用包含开始不包含截止的方式，例如 startData="2022-04-01" endDate="2022-04-05"时，表示拉取的是以下这4天的数据
 ```json
["2022-04-01","2022-04-02","2022-04-03","2022-04-04"]
 ```
基础参数要求：startDate 必须 < endDate；为避免查询阻塞，限制了一次查询的跨度不可以超过100天


- 正确时响应内容 指定了gameId时：

  ```json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"details": [
          		{
          			"date": "2022-04-01", // "yyyy-MM-dd"格式日期
          			"valueGain": 123512, // 该玩家该日期收益
          		},
          		{
          			"date": "2022-04-02", // "yyyy-MM-dd"格式日期
          			"valueGain": 1232, // 该玩家该日期收益
          		}
          		{
          			"date": "2022-04-03", // "yyyy-MM-dd"格式日期
          			"valueGain": 0, // 该玩家该日期收益
          		},
          		{
          			"date": "2022-04-04", // "yyyy-MM-dd"格式日期
          			"valueGain": 72, // 该玩家该日期收益
          		},
          	],
          	"totalValueGain": 124816, //该玩家在该游戏、该区间日期内的总收益（等于details内所有valueGain的加和）
          	"gameId": "7531b58338104dcd9e7eddcce96c4ae0",
          }
      }
  }
  ```
  
- 正确时响应内容 没有指定gameId或gameId设置为"all"时：

  ```json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"details": [
          		{
          			"date": "2022-04-01", // "yyyy-MM-dd"格式日期
          			"valueGain": 123512, // 该玩家该日期在所有游戏的总收益
          		},
          		{
          			"date": "2022-04-02", // "yyyy-MM-dd"格式日期
          			"valueGain": 1232, // 该玩家该日期在所有游戏的总收益
          		}
          		{
          			"date": "2022-04-03", // "yyyy-MM-dd"格式日期
          			"valueGain": 0, // 该玩家该日期在所有游戏的总收益
          		},
          		{
          			"date": "2022-04-04", // "yyyy-MM-dd"格式日期
          			"valueGain": 72, // 该玩家该日期在所有游戏的总收益
          		},
          	],
          	"totalValueGain": 124816, //该玩家在该区间日期内的总收益（等于details内所有valueGain的加和）
          }
      }
  }
  ```
  

#### 3.2.2 玩家在指定游戏在线时长数据查询：


- 功能说明：查询玩家在指定游戏的每日在线时长

- 请求方式：POST

- 请求地址：/metaoneData/queryUserGameOnlineTime

- 请求头：

HeaderName | 是否必须 | HeaderValue说明
------- | ------- | -------
Authorization | 是 | 当前保存的token<br/>【token需要每日通过调用获取token接口获取并更新/保存起来】
lang | 否 | 语言，缺省值"en"英文，当前仅支持中文与英文 

- 请求内容：

  ```json
  {
  	"userId": "06be07ff82754a718a96ef46d1b82b2c", // 用户ID
  	"gameId": "7531b58338104dcd9e7eddcce96c4ae0", // 游戏ID。这是必须的。暂不支持gameId=all的情况。
  	"startDate" : "2022-04-01", //查询范围的起始日期（为yyyy-MM-dd字符串格式）查询结果会包含这个日期
  	"endDate" : "2022-04-05", //查询范围的截止日期（为yyyy-MM-dd字符串格式
  }
  ```
  
	- 关于日期参数的特别说明：

 日期参数采用包含开始不包含截止的方式，例如 startData="2022-04-01" endDate="2022-04-05"时，表示拉取的是以下这4天的数据
 ```json
["2022-04-01","2022-04-02","2022-04-03","2022-04-04"]
 ```
基础参数要求：startDate 必须 < endDate；为避免查询阻塞，限制了一次查询的跨度不可以超过100天


- 正确时响应内容：

  ```json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"details": [
          		{
          			"date": "2022-04-01", // "yyyy-MM-dd"格式日期
          			"onlineTime": 123512, // 该玩家该日期在线时长（单位秒）
          		},
          		{
          			"date": "2022-04-02", // "yyyy-MM-dd"格式日期
          			"onlineTime": 1232, // 该玩家该日期在线时长（单位秒）
          		}
          		{
          			"date": "2022-04-03", // "yyyy-MM-dd"格式日期
          			"onlineTime": 0, // 该玩家该日期在线时长（单位秒）
          		},
          		{
          			"date": "2022-04-04", // "yyyy-MM-dd"格式日期
          			"onlineTime": 72, // 该玩家该日期在线时长（单位秒）
          		},
          	],
          	"totalOnlineTime": 124816, //bigint长整型，该玩家在该游戏该区间日期内总体在线时长
          	"gameId": "7531b58338104dcd9e7eddcce96c4ae0",
          }
      }
  }
  ```
  
  - gameId=all的情况对应玩家当天游戏时长，当前版本暂无可支持的原始数据。如果要查询的是玩家在每日的所有游戏在线时长的直接累加，请使用接口3.2.3。

#### 3.2.3 玩家每日在所有游戏在线时长直接累加数据查询:

- 功能说明：查询玩家每日在各个游戏在线时长的累加。【注：同一个玩家账号若有同时用于多款游戏，该累加值有可能会超过24小时。请接入方注意数据类型的设置】

- 请求方式：POST

- 请求地址：/metaoneData/queryUserGameOnlineTimeDirectSum

- 请求内容：

  ```json
  {
  	"userId": "06be07ff82754a718a96ef46d1b82b2c", // 用户ID
  	"startDate" : "2022-04-01", //查询范围的起始日期（为yyyy-MM-dd字符串格式）查询结果会包含这个日期
  	"endDate" : "2022-04-05", //查询范围的截止日期（为yyyy-MM-dd字符串格式
  }
  ```
  
	- 关于日期参数的特别说明：

 日期参数采用包含开始不包含截止的方式，例如 startData="2022-04-01" endDate="2022-04-05"时，表示拉取的是以下这4天的数据
 ```json
["2022-04-01","2022-04-02","2022-04-03","2022-04-04"]
 ```
基础参数要求：startDate 必须 < endDate；为避免查询阻塞，限制了一次查询的跨度不可以超过100天


- 正确时响应内容：

  ````json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"details": [
          		{
          			"date": "2022-04-01", // "yyyy-MM-dd"格式日期
          			"sumOfOnlineTime": 123512, // 该玩家该日期所有游戏在线时长的加和（单位秒），该值可能超过24小时，注意使用的数据类型
          		},
          		{
          			"date": "2022-04-02", // "yyyy-MM-dd"格式日期
          			"sumOfOnlineTime": 1232, // 该玩家该日期所有游戏在线时长的加和（单位秒），该值可能超过24小时，注意使用的数据类型
          		}
          		{
          			"date": "2022-04-03", // "yyyy-MM-dd"格式日期
          			"sumOfOnlineTime": 0, // 该玩家该日期所有游戏在线时长的加和（单位秒），该值可能超过24小时，注意使用的数据类型
          		},
          		{
          			"date": "2022-04-04", // "yyyy-MM-dd"格式日期
          			"sumOfOnlineTime": 72, // 该玩家该日期所有游戏在线时长的加和（单位秒），该值可能超过24小时，注意使用的数据类型
          		},
          	],
          	"totalSumOfOnlineTime": 124816, //bigint长整型，该玩家在该区间日期内所有游戏的在线时长的累加值
          }
      }
  }
  ````

#### 3.2.4 玩家每日活跃整体情况数据查询(待定)：

### 3.3 公会数据模块(待定)：

含基于公会的数据数据查询和公会玩家关系同步相关流程的接口

### 3.4 道具信息数据模块：

#### 3.4.1 通过道具Id(tokenId）查询道具信息：

- 功能说明：通过道具Id，查询道具信息。

- 请求方式：GET

- 请求地址：/metaoneData/queryGameItemById

- 请求头：

HeaderName | 是否必须 | HeaderValue说明
------- | ------- | -------
Authorization | 是 | 当前保存的token<br/>【token需要每日通过调用获取token接口获取并更新/保存起来】
lang | 否 | 语言，缺省值"en"英文，当前仅支持中文与英文 

- 请求参数：


参数名 | 是否必须 |说明
------- | ------- | -------
itemId | 是 | 道具ID/tokenId



- 正确时响应内容：

  ````json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"ownerId": "道具拥有者的ID/钱包ID",
          	"details": {
  ````

					"itemImage": "https://xxxx/xxx.png", //道具配图url地址
					"gameId": "游戏id", 
					"attributes" :[  //所有扩展字段会以如下格式放在attributes数组内
						{
							"attrCode" : "属性的code",  // 平台需要根据与游戏的约定，用attrCode做国际化
							"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
						},	
						{
							"attrCode" : "属性的code",   // 平台需要根据与游戏的约定，用attrCode做国际化
							"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
						},
						...
					]
				},
	      }
	  }
  }
  ````
  
#### 3.4.2 查询指定玩家（钱包ID）名下的道具：

- 功能说明：通过道具Id，查询道具信息。由于道具信息数据量可能偏大。建议选择合适的分页大小进行分页加载。返回结果中，itemId、itemImage和gameId都是必选字段

- 请求方式：GET

- 请求地址：/metaoneData/listUserGameItems

- 请求头：

HeaderName | 是否必须 | HeaderValue说明
------- | ------- | -------
Authorization | 是 | 当前保存的token<br/>【token需要每日通过调用获取token接口获取并更新/保存起来】
lang | 否 | 语言，缺省值"en"英文，当前仅支持中文与英文 

- 请求参数：


参数名 | 是否必须 |说明
------- | ------- | -------
userId | 是 | 玩家Id/钱包ID
pageNum | 是 | 分页页码，第一页从0开始
pageSize | 是 | 分页大小，必须为正整数


- 正确时响应内容：

  ````json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"totalCount": 23131, //该玩家名下的道具总数
          	"itemCount": 20, //当前请求查询到的结果数量
          	"items": [
          		{
		          	"itemId": "道具ID/tokenID",
		          	"details": {
						"itemImage": "https://xxxx/xxx.png", //道具配图url地址
						"gameId": "游戏Id", 
						"attributes" :[  //所有扩展字段会以如下格式放在attributes数组内
							{
								"attrName" : "字段的标题或名称", 
								"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
							},	
							{
								"attrName" : "字段的标题或名称", 
								"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
							},
						]
					},
          		},
          		{
		          	"itemId": "道具ID/tokenID",
		          	"details": {
						"itemImage": "https://xxxx/xxx.png", //道具配图url地址
						"gameId": "游戏ID", 
						"attributes" :[  //所有扩展字段会以如下格式放在attributes数组内
							{
								"attrName" : "字段的标题或名称", 
								"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
							},	
							{
								"attrName" : "字段的标题或名称", 
								"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
							},
						]
					},
          		},
          		...
          	]
          }
      }
  }
  ````

## 4 NFT Lease

### 4.1 角色

| **角色** | **说明**                                 |
| -------- | ---------------------------------------- |
| admin    | lease合约的管理员（钱包地址）            |
| lender   | 出租人（钱包地址）                       |
| renter   | 租用人（玩家钱包地址）                   |
| 游戏平台 | 中心化游戏平台                           |
| manager  | 管理员，参与从合约签名提币（ERC20token） |
| guild    | 公会 (分红与平台链下结算)                |

### 4.2 租赁模式

| Rent  模式（纯租金）                                         | Staking 模式（租金+游戏奖励）                                |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 租金：玩家钱包里的现金。  支持admin设置的ERC20token（包括USDT、metaone token、其它） | 租金：现金 + 游戏币（ERC20类型）  游戏奖励：每款游戏各自的奖励币（ERC20类型） |
| 租金计算：单位数量 现金/天 x 租借天数                        | 租金计算：  现金：单位数量 现金/天 x 租借天数  游戏币计算：游戏平台计算（给到游戏平台NFT地址、tokenID和租借时间段） |
| 支付方：renter                                               | 支付方：renter & 游戏平台                                    |
| 支付方式：形成租赁订单时，主动一次性支付给lease合约          | 支付方式：租期到后，将游戏币转入lease合约                    |
| 收益方：admin、lender                                        | 收益方：admin和公会、lender、renter                          |
| 提取方式：收益方发出提取申请，admin/manager手动发放          | 提取方式：收益方发出提取申请，admin/manager手动发放          |

### 4.3 租赁状态

| 状态         | 描述                                                     |
| ------------ | -------------------------------------------------------- |
| for_rent     | 可以被租赁                                               |
| rented       | 已被租（玩家发起租用后生效，包含使用前和使用中两种情况） |
| not_rentable | 不可被租赁，NFT在合约里                                  |
| empty        | NFT已被lender提出                                        |

### 4.4 业务功能

#### 4.4.1 出租

##### 4.4.1.1 发布出租信息

- 模式：Rent、Staking

- 触发角色：lender

- 简述

  - lender把自己的NFT存放放到合约用于出租；

  - 支持ERC721和ERC1155两种token，ERC1155出租张数为1张。

  - 填写出租信息（NFT地址和token ID、每天多少租金、游戏抽成、最少、最多出租天数、是否可以续租）。

  - 提交成功后，会分配给一个lendItemID（uint类型），所有的出租状态，指的都是lendItemID的状态。

  - **[注意]现金的精度是18位，租金为1USDT时，合约需要读入1000000000000000000，下文不在赘述。**

- 业务流程图

​          ![image-20220606163232132](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220606163232132.png)                     

- 对接代码(接口号：1.1)

```solidity
function deposit(
TokenType _tkType,  // NFT类型（ERC721/ERC1155）
address _NFTaddr,  //NFT合约地址
uint _tokenID,    // NFT tokenID
bool _renewable,    //是否可续租
uint _coinIndex,   // 支付租金使用的现金币种
uint8 _minimumLeaseTime,   //最少租用天数
uint8 _maximumLeaseTime,   //最长租用天数
uint _price,   // 租金价格（USDT/天）
uint8 _gameBonus,   // lender游戏币抽成百分比,Rent模式为0；Staking模式：范围：1～99 抽成
) external returns(bool); 
```

##### 4.4.1.2 修改是否可以续租

- lendItem状态:for_rent, rented, not_rentable

- 触发角色：lender

- 简述

  - 一个rented状态的租借item；
  - 如果可以续租，在租期到了之后，所有的玩家都新发起对这个NFT的租用；
  - 如果不可以续租，则所有玩家都不可以发起租用请求，直到lender开启该NFT的可租用设置。

- 业务流程图

  ![image-20220606164028031](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220606164028031.png)

- 对接代码(接口号：1.2)

  ```solidity
  function renewableStatus(
  uint _lendItemID,
  bool _renewable
  ) external returns(bool);
  
  ```

  

##### 4.4.1.3 修改出租信息

- lendItem状态：all

- 触发角色：lender

- 简述

  - rented和empty状态无法修改信息。
  - 修改信息后，新gamer租借该NFT，则按照新的信息要求。

- 业务流程图

  ![image-20220606164243645](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220606164243645.png)

- 对接代码(接口号：1.3)

  ```solidity
  function resetDeposit(
  uint _lendItemID,
  uint8 _minimumLeaseTime,
  uint8 _maximumLeaseTime,
  uint _price,
  uint8 _gameBonus
  ) external returns(bool);
  
  ```

  

##### 44.1.4 提出NFT token

- lendItem状态

  - for_rent
  - not_rentable

- 触发角色：lender

- 简述

  - lender从合约提出NFT到自己的钱包。
  - rented与empty状态的NFT不可被提取。
  - 如果NFT可续租，lender为了防止不断有gamer租用走NFT而导致无法提出NFT，则需要先把状态设置为不可租用，在本次租期到了之后，就可以提出了。
  - 提出后，该lendItemID的状态为：empty

- 业务流程图

  

- ![image-20220606164630107](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220606164630107.png)

- 对接代码(接口号：1.4)

  ```solidity
  function withdraw(uint _lendItemID) external;
  ```

  

##### 4.4.1.5 查看我出租的NFT

- lendItem状态：ALL

- **触发角色**：lender

- **简述**

  - lender可以查看所有自己出租过的lendItemID信息

- **对接代码(接口号：1.5)**

  ```solidity
  function getMyDepositsList() external returns(AllMsg[] memory);
  ```

  

#### 8.4.2 租借

##### 4.4.2.1 Gamer租借

##### 4.4.2.2.	Gamer租借

- 租赁模式

  - Rent
  - Staking

- 触发角色：renter

- 简述

  - 玩家选择“for_rent”状态的NFT进行租借。

  - 填写要租用的天数，提交租赁请求，成功后，租用次日生效。

  - 现金租金，在租用生效的同时一次性全部发给合约。

  - [注意]

    - 发送现金到合约之前，需要gamer先对合约地址做approve操作（调用IERC20标准接口：接口号2.5）
    - gamer入会后，首次租借要绑定公会，下次租借，直接读已绑定的公会数据

  - 业务流程图

    ![image-20220606165840283](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220606165840283.png)

  - 对接代码(接口号：1.6)

    ```solidity
    function rent(
    uint _lendItemID,
    uint _rentTimes，//租借天数
    ) external returns(bool);
    
    ```

    

##### 4.4.2.3 查看所有我租用的token

- 触发角色：renter

- 简述：gamer可以查看自己所租用的所有lendItemID的信息

- 对接代码(接口号：1.7)

  ```solidity
  function getMyRentsList() external returns(AllMsg[] memory);
  ```

  

#### 4.4.3 Admin设置

##### 4.4.3.1 admin设置平台manager

- 触发角色：admin
- 简述
  - admin可以添加/删除多个管理员。
  - 管理员可参与多重签名（从合约提币时需要3个admin/manager的签名才能成功）。
- 对接代码(接口号：3.1&3.2)

```solidity
function addManager(address manager) external ;
```

```solidity
function removeManager(address manager) external;
```



##### 4.4.3.2 admin 修改平台佣金百分比

- 触发角色：admin

- 简述

  - renter支付的现金、获得的游戏奖励，均按照此比例支付给平台
  - 在合约初始化时设定改值
  - admin修改后，即时生效（“rented”状态除外）

- 业务流程

  ![image-20220606170255894](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220606170255894.png)

- 对接代码(接口号：1.8)

  ```solidity
  function setPlantformBonus(uint8 _plantformBonus) external;
  ```

  

##### 4.4.3.3 admin 添加支持的租金类型

- 触发角色：admin

- 简述

  - ender只可以收取平台支持的租金类型（合约初始化时已设置，如：USDT、metaone等）。
  - admin可以添加支持支付租金所使用的ERC20代币。
  - lender新发起的出租信息，只能选择1种平台支持的现金token，发布后不允许修改。
  - 修改后，即时生效（“rented”状态除外）。
  - 注：添加后无法删除或替换

- 对接代码(接口号：1.10)

  ```solidity
  function addRentCoin(address _rentCoin) external returns(bool)
  ```

  

##### 4.4.3.4 admin 从合约提取佣金

- 触发角色：Admin、公会

- 简述

  - Admin、公会向服务器发起领取收益申请（包括现金和游戏奖励，由admin/manager调用合约进行发放。
  - admin与manager可直接调用合约领取收益。
  - 领取条件：需要admin或manager调用合约签名（暂定至少3个人签名）才能领取。

- 对接代码(接口号：3.4)

  ```solidity
  function managerClaim(
  IERC20 _addr,     //token地址
  uint _amount      //提币数额
  ) external returns(bool);
  
  ```

  

#### 4.4.4其它

##### 4.4.4.1 user现金领取

- 触发角色：lender、renter

- 简述
  - lender租金现金领取：
  - Lender获得的现金租金记录在合约，
  - 可以直接领取，领取数额不得大于自己的余额。
  - lender与renter的游戏奖励领取：
  - 链下申请领取。
  - 由admin固定时间统一发放。

对接代码(接口号：1.9)

```solidity
function userclaim(
IERC20 _addr,
uint _amount
) external returns(bool);

```



##### 4.4.4.2 签名

- 触发角色：Admin、manager

- 简述

  - 从合约提出租金/游戏奖励时，需要admin/manager3个人签名才能领取

- 对接代码(接口号：3.3)

  ```solidity
  function signTransaction(uint transactionId) external;
  ```

  

##### 4.4.4.3 查看租约信息

- 触发角色：ALL

- 简述

  - 根据lendItemID，查看该条租约当前信息

- 对接代码(接口号：1.11)

  ```solidity
  function getLendItemMsg(uint _lendItemID) external view returns(AllMsg memory _AllMsg);
  ```

  

### 4.5 所有接口列表

#### 4.5.1 IVaultManager.sol文件

- **接口号 1.1**

  ```solidity
  function deposit(
  TokenType _tkType,  // NFT类型（ERC721/ERC1155）
  address _NFTaddr,  //NFT合约地址
  uint _tokenID,    // NFT tokenID
  bool _renewable,    //是否可续租
  uint _coinIndex,   // 支付租金使用的现金币种
  uint8 _minimumLeaseTime,   //最少租用天数
  uint8 _maximumLeaseTime,   //最长租用天数
  uint _price,   // 租金价格（USDT/天）
  uint8 _gameBonus,   // lender游戏币抽成百分比,Rent模式为0；Staking模式：范围：1～99 抽成
  ) external returns(bool);  
  
  ```

  

- **接口号 1.2**

  ```solidity
  function renewableStatus(
  uint _lendItemID,
  bool _renewable
  ) external returns(bool);
  
  ```

  

- **接口号 1.3**

  ```solidity
  function resetDeposit
  (uint _lendItemID,
  uint8 _minimumLeaseTime,
  uint8 _maximumLeaseTime,
  uint _price,
  uint8 _gameBonus
  ) external returns(bool);
  
  ```

  

- **接口号 1.4**

  ```solidity
  function withdraw(uint _lendItemID) external;
  ```

  

- **接口号 1.5**

  ```solidity
  function getMyDepositsList() external returns(AllMsg[] memory);
  ```

  

- **接口号 1.6**

  ```solidity
  function rent(
  uint _lendItemID,
  uint _rentTimes，//租借天数
  ) external returns(bool);
  
  ```

  

- **接口号 1.7**

  ```solidity
  function getMyRentsList() external returns(AllMsg[] memory);
  ```

  

- **接口号 1.8**

  ```solidity
  function setPlantformBonus(uint8 _plantformBonus) external;
  ```

  

- **接口号 1.9**

  ```solidity
  function userclaim(
  IERC20 _addr,
  uint _amount
  ) external returns(bool);
  
  ```

  

- **接口号 1.10**

  ```solidity
  function addRentCoin(address _rentCoin) external returns(bool);
  ```

  

- **接口号 1.11**

  ```solidity
  function getLendItemMsg(uint _lendItemID) external view returns(AllMsg memory _AllMsg);
  ```

  

- **接口号 1.12**

  ```solidity
  event Deposit(
  address _lender,
  address _NFTaddr,
  uint _tokenID,
  bool _renewable,
  uint8 _minimumLeaseTime,
  uint8 _maximumLeaseTime,
  uint _price,
  uint8 _gameBonus);
  
  ```

  

- **接口号 1.13**

  ```solidity
  event Rent(
  uint _lendItemID,
  address _NFTaddr,
  uint _tokenID,
  address _renter,
  uint _nowtime,
  uint _endtime,
  uint8 _gameBonus,
  uint _rentTimes)
  
  ```

  

- **接口号 1.14**

  ```solidity
  event Withdraw
  (uint _lendItemID,
  address _NFTaddr,
  uint _tokenID, 
  address _lender);
  
  ```

  

- **接口号 1.15**

  ```solidity
  event SetPlantformBonus(
  uint8 _plantformBonus);
  
  ```

  

- **接口号 1.16**

  ```solidity
  event RenewableStatus(
  uint _lendItemID,
  bool _renewable);
  
  ```

  

- **接口号 1.17**

  ```solidity
  event ResetDeposit(
  uint _lendItemID,
  uint8 _minimumLeaseTime,
  uint8 _maximumLeaseTime,
  uint _price,
  uint8 _gameBonus); 
  
  ```

  

#### 4..5.2 IERC20.sol文件

- **接口号 2.1**

  ```solidity
  function totalSupply() external view returns (uint256);
  ```

  

- **接口号 2.2**

  ```solidity
  function balanceOf(address account) external view returns (uint256);
  ```

  

- **接口号 2.3**

  ```solidity
  function transfer(address to, uint256 amount) external returns (bool);
  ```

  

- **接口号 2.4**

  ```solidity
  function allowance(address owner, address spender) external view returns (uint256);
  ```

  

- **接口号 2.5**

  ```solidity
  function approve(
  address spender,    //被approve对象，合约地址
  uint256 amount     //approve数额：price x 天数
  ) external returns (bool);
  
  ```

  

- **接口号 2.6**

  ```solidity
  function transferFrom(
  address from,
  address to,
  uint256 amount
  ) external returns (bool);
  
  ```

  

#### 4.5.3 IMultiSigWallet.sol文件

- **接口号 3.1**

  ```solidity
  function addManager(address manager) external ;
  ```

  

- **接口号 3.2**

  ```solidity
  function removeManager(address manager) external;
  ```

  

- **接口号 3.3**

  ```solidity
  function signTransaction(uint transactionId) external;
  ```

  

- **接口号 3.4**

  ```solidity
  function managerClaim(
  IERC20 _addr,     //token地址
  uint _amount      //提币数额
  ) external returns(bool);
  
  ```

  

## 文档更新日志

- 初始编写 [2022-3-30]
- 更新：增加智能合约对接API代码[ Lyon 2022-06-15 ] [内容来源：智能合约团队Hayley提供]