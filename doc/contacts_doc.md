[TOC]

## 一、常量定义

### <span id="relation">1、与本人关系</span>

| 枚举     | code | value |
| -------- | ---- | ----- |
| SELF     | 0    | 本人  |
| PARENT   | 1    | 父母  |
| CHILDREN | 2    | 子女  |
| SPOUSE   | 3    | 配偶  |
| OTHER    | 4    | 其他  |

### <span id="cardType">2、证件类型</span>[^2]

| 枚举                  | code | value    |
| --------------------- | ---- | -------- |
| IDENTITY_CARD         | 1    | 身份证   |
| OFFICER_CARD          | 2    | 警官证   |
| MILITARY_OFFICER_CARD | 3    | 军官证   |
| OTHER_CARD            | 4    | 港澳台证 |
| PASSPORT              | 5    | 护照     |

### <span id="married">3、婚否[^3]</span>

| 枚举   | code | value |
| ------ | ---- | ----- |
| YES    | 1    | 已婚  |
| NO     | 2    | 未婚  |
| COMMON | 3    | 通用  |

### <span id="gender">4、性别[^4]</span>

| 枚举  | code | value |
| ----- | ---- | ----- |
| MAN   | 1    | 男性  |
| WOMAN | 2    | 女性  |
| BOTH  | 3    | 通用  |

### <span id="default">5、是否默认</span>

| 枚举 | code | value |
| ---- | ---- | ----- |
| NO   | 0    | 否    |
| YES  | 1    | 是    |



## 二、接口文档

### 新增联系人

#### 1、描述

> 体检预约订单确认页面新增联系人

#### 2、请求路径

​	`/api/contacts/add`

#### 3、请求入参

| 字段     | 类型   |是否必传| 描述                                       |
| -------- | ------ | ------ | ------------------------------------------ |
| name     | string |Y|姓名|
| relation | string |Y| 与本人关系，[**枚举**](#relation)固定值       |
| mobile   | string |Y| 手机号                                     |
| cardType | int    |Y| 证件类型，[**枚举**](#cardType)固定值 |
| cardNo   | string |Y| 证件号码                                   |
| married  | string |Y| 婚否，[**枚举**](#married)固定值 |
| gender   | string |Y| 性别，[**枚举**](#gender)固定值  |
| birthday | string |Y| 出生日期                                   |
| default  | int    |N| 是否设置为默认联系人(默认值为0)  <br />0：非默认 1：默认 |

#### 4、请求响应

| 字段      | 类型 | 描述     |
| --------- | ---- | -------- |
| contactId | int  | 联系人ID |

```json
{
    "code": 0,
    "msg": "",
    "data": {
        "contactId": 1
    }
}
```

### 编辑联系人

#### 1、描述

> 体检预约订单确认页面编辑联系人

#### 2、请求路径

​	`/api/contacts/update`

#### 3、请求入参

| 字段      | 类型   | 是否必传 | 描述                                                     |
| --------- | ------ | -------- | -------------------------------------------------------- |
| contactId | int    | 是       | 联系人id                                                 |
| name      | string | Y        | 姓名                                                     |
| relation  | string | Y        | 与本人关系，[**枚举**](#relation)固定值                  |
| mobile    | string | Y        | 手机号                                                   |
| cardType  | int    | Y        | 证件类型，[**枚举**](#cardType)固定值                    |
| cardNo    | string | Y        | 证件号码                                                 |
| married   | string | Y        | 婚否，[**枚举**](#married)固定值                         |
| gender    | string | Y        | 性别，[**枚举**](#gender)固定值                          |
| birthday  | string | Y        | 出生日期                                                 |
| default   | int    | N        | 是否设置为默认联系人(默认值为0)  <br />0：非默认 1：默认 |

#### 4、请求响应

| 字段    | 类型    | 描述         |
| ------- | ------- | ------------ |
| updated | boolean | 是否更新成功 |

```json
{
    "code": 0,
    "msg": "",
    "data": {
        "updated": true
    }
}
```

### 联系人列表

#### 1、描述

> 体检预约订单确认页面获取联系人列表

#### 2、请求路径

​	`/api/contacts/list`

#### 3、请求入参

无

#### 4、请求响应

| 字段            | 类型    | 描述                                   |
| --------------- | ------- | -------------------------------------- |
| availableList   | list    | 可用联系人                             |
| unavailableList | list    | 不可用联系人，**已下字段两个列表通用** |
| contactId       | int     | 联系人ID                               |
| name            | string  | 姓名                                   |
| age             | int     | 年龄                                   |
| gender          | string  | 性别                                   |
| married         | string  | 婚否                                   |
| default         | boolean | 是否默认 <br />0：默认， 1：非默认     |
| relation        | string  | 与本人关系                             |
| mobile          | string  | 手机号                                 |
| cardType        | int     | 证件类型                               |
| cardNo          | string  | 证件号码                               |

```json
{
    "code": 0,
    "msg": "",
    "data": {
        "availableList": [
            {
                "contactId": 1,
                "name": "张三",
                "age": 18,
                "gender": "男性",
                "married": "未婚",
                "default": true,
                "relation": "本人",
                "mobile": "13122223333",
                "cardType": 1,
                "cardNo": "310101199003070051"
            },
            {
                "contactId": 1,
                "name": "李四",
                "age": 18,
                "gender": "女性",
                "married": "未婚",
                "default": false,
                "relation": "配偶",
                "mobile": "13233334444",
                "cardType": 4,
                "cardNo": "1234567"
            },
        ],
        "unavailableList": [
            {
                {
                "contactId": 1,
                "name": "王五",
                "age": 38,
                "gender": "男性",
                "married": "已婚",
                "default": false,
                "relation": "父母",
                "mobile": "13344445555",
                "cardType": 1,
                "cardNo": "310101199003074618"
            },
            }
        ]
    }
}
```

### 删除联系人

#### 1、描述

> 体检预约订单确认页面删除联系人

#### 2、请求路径

​	`/api/contacts/delete`

#### 3、请求入参

| 字段      | 类型 | 是否必传 | 描述     |
| --------- | ---- | -------- | -------- |
| contactId | int  | Y        | 联系人ID |

#### 4、请求响应

| 字段    | 类型    | 描述         |
| ------- | ------- | ------------ |
| deleted | boolean | 是否删除成功 |

```json
{
    "code": 0,
    "msg": "",
    "data": {
        "deleted": true
    }
}
```









































[^2]: 见枚举：com.dffl.bl.mall.dict.CertificatesTypeEnum
[^3]: 见枚举：com.dffl.ml.medical.consts.MarriedConst
[^4]: 见枚举：com.dffl.ml.medical.consts.GenderConst