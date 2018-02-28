# 目录
# 签名和验签
* 协议只支持HTTPS，确保解决域名，网络运营商相关方面的风险
* 平台与商户应该完成公钥交换
* 双向RSA签名：请求发起者需以私钥对请求内容签章；响应者需以私钥对响应内容签章；响应者应该阻止请求签章无法用对应公钥验签的请求；请求者应该阻止响应签章无法用对应公钥验签的响应；
## 请求签名算法

参与签名的字段及顺序：

| 顺序   | 参数            | 获取位置        | 描述                             | 示例                                       |
| ---- | ------------- | ----------- | :----------------------------- | ---------------------------------------- |
| 1    | method        | http_method | 请求方法，大写(POST/GET/PUT/..etc,)              | POST                                     |
| 2    | uri           | url         | 统一资源标识符[API方法]                 | /test                              |
| 3    | query_string  | url         | 查询字符串，若无则使用空字符串，切勿使用null                          | a=1&b=2&c=3                              |
| 4    | X-Pay-Timestamp         | header      | 时间戳                         | 1466399895704         |
| 5    | X-Pay-Authorization | header      | 商户的唯一标识 | 5b97b3138041437587646b37f52dc7f7         |
| 6    | request_content  | content        | 请求数据，若无就为空                           | 二进制数据 |

签名步骤：

1. 签名前以 换行符（"\n"） 拼接前5个字符串字段：

   ```
   method+"\n"+uri+"\n"+query_string+"\n"+X-Pay-Timestamp+"\n"+X-Pay-Authorization
   ```

   **不同的语言，换行符可能有所不同**

   组装成要签名的数据，例如：

   ```
   POST
   /test
   a=1&b=2&c=3
   1466399895704
   5b97b3138041437587646b37f52dc7f7
   ```


1. 以UTF-8编码将待签名字符串转换成字节数组再同request_content合并，然后使用**私钥**对其签名，签名算法为SHA1WithRSA，将签名后的结果以Base64编码转码后存放到header中，key 为 'X-Pay-Sign'。

   假如签名结果为：

   ```
   BkBa8OLkU2KzRIrWA4swP5WCSouVwXHFAUM6NlJlgDGbMQYKKvVveE30pXppPcesRLPPlpTctFdCt+nY4czX79efg19FDEizq94d+HAsE/3dtzUMiiFyxOWFH50FGO+gJZDAEpYQRCdJlOs6D380ta+y0wRfUhlfgX1+RXtcDnA=
   ```

   则最后的请求Header是：

   ```http
   POST /test HTTP/1.1
   Host: https://www.easypay.com
   ContentType: application/json;charset=utf-8
   X-Pay-Authorization: 5b97b3138041437587646b37f52dc7f7
   X-Pay-Timestamp: 1466399895704
   X-Pay-Sign: BkBa8OLkU2KzRIrWA4swP5WCSouVwXHFAUM6NlJlgDGbMQYKKvVveE30pXppPcesRLPPlpTctFdCt+nY4czX79efg19FDEizq94d+HAsE/3dtzUMiiFyxOWFH50FGO+gJZDAEpYQRCdJlOs6D380ta+y0wRfUhlfgX1+RXtcDnA=

   {"foo":"bar"}
   ```

#### 响应签名算法(与请求签名算法类似，只是参与者略有出入)

参与签名的字段及顺序：

| 顺序   | 参数            | 获取位置   | 描述                             | 示例                               |
| ---- | ------------- | ------ | :----------------------------- | -------------------------------- |
| 1    | X-Pay-Timestamp         | header | 时间戳                         | 1466399895704 |
| 2    | X-Pay-Authorization | header | 商户的唯一标识 | 5b97b3138041437587646b37f52dc7f7 |
| 3    | response_content | content   | 响应数据，若无就为空                   | 二进制数据    |

签名步骤：

1. 验签前以 '\n' 拼接前2个字符串字段：

   ```
   X-Pay-Timestamp+'\n'+X-Pay-Authorization
   ```

   组装成要验签的数据，例如：

   ```
   1466399895704
   5b97b3138041437587646b37f52dc7f7
   ```

2. 以UTF-8编码将待验签字符串转换成字节数组再跟response_content合并，然后使用**私钥**对其签名，签名算法为SHA1WithRSA。

返回的response示例：

```http
ContentType: application/json;charset=utf-8
X-Pay-Authorization: 5b97b3138041437587646b37f52dc7f7
X-Pay-Timestamp: 1466399895704
X-Pay-Sign: Lp6TovxVq1r+qgai/B7M7ovV8NDsncZ6j6GfFUlR6QGVPtvpqkliS2kgo/mfm6AgFqpVy+edOGZnjlnohDEjQ7QO4W/AzvMb+/S+UZPcvSyY4zamg8ne0+6cwh7mxu5rQvTknYKSwE99fYtTkla2IvWUfn5ch9fW6MSErRQyzRc=

{"bar":"foo"}
```

#### 注意事项

1. 对数据进行签名或验签时，字段顺序不能变；
2. 对签名后的值要进行Base64Encode 后才能往 header中存放；
3. 验签时对获取到的签名数据【X-Pay-Sign】 要进行 Base64 Decode后再进行验证；
4. 验签时要对 签名数据、商户唯一识别符、时效性(时间戳在当前时间戳1天之前的必须阻止，极有可能是暴力破解) 进行验证。
