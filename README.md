# dysms
send sms interface for aliyun

最近需要接入 [阿里云](https://www.aliyun.com/) [短信服务](https://www.aliyun.com/product/sms) 用来发送验证码。由于官方并未提供Golang相应SDK，自己就尝试实现了发送短信的接口，这里分享给大家。

项目地址：[https://github.com/panshiqu/dysms](https://github.com/panshiqu/dysms)

使用示例：
```GO
	if err := dysms.SendSms(accessKeyID, accessSecret, phoneNumbers, signName, templateParam, templateCode); err != nil {
		log.Println("dysms.SendSms", err)
	}
```

其实就是翻译了 [HTTP协议及签名](https://help.aliyun.com/document_detail/56189.html) 这篇官方API文档中的示例代码，Java -> Go

若你也想实现该接口（重复造轮子），我建议你把所有字段值暂设为官方示例给出的值，譬如时间戳设为”2017-07-12T02:42:19Z“，这样你就可以每步都与官方文档中的打印结果作对比，进而发现自己代码中的问题。

列举我遇见的错误码：

InvalidTimeStamp.Expired
时间戳错误，发出请求的时间和服务器接收的时间不在15分钟内。经常出现该错误的原因是时区造成的，目前网关使用的时区是GMT等同于UTC。

SignatureNonceUsed
唯一随机数重复，SignatureNonce是唯一随机数，用于防止网络重放攻击。不同请求间要使用不同的随机数值。关于网络重放攻击，我们可以构造一次合法的请求，打印并记录URL，因为Timestamp容忍15分钟内的误差，我们就可以在15分钟内重复访问该URL进而发送短信。SignatureNonce的存在终结了这种可能，服务器应该会缓存SignatureNonce值至少15分钟，每次请求到来都会检查是否存在相同的SignatureNonce值，若存在则请求失败，返回该错误码。上面实现的接口中用```rand.Int63()```来产生SignatureNonce值，你可能担心15分钟内会产生重复的随机值，其实大可不必考虑这点，因为收到该错误码会递归调用生成新的随机值再次请求。

实现该接口大部分时间都在解决一个简单的错误：```malformed HTTP status code "HTML"```。不要高看这个错误，它其实就是字面意思，畸形的，难看的HTTP。之所以会遇到这个错误，主要因为我会打印拼接好的URL，为了方便我看，在```fmt.Sprintf```中习惯性加了```\n```。其实你单去搜索这个错误，真是找不到有价值的解释，因为很少会有人犯这种错误。至于我是如何发现错在哪里的，这里作简短说明。先是发现复制打印的URL直接用浏览器和CURL都可以请求成功，这时还意识不到错在哪里。接着用Wireshark抓包发现跟在URL后面的```HTTP/1.1```，一个在同一行，一个在下一行，因为不同点不只有这一处，所以虽然发现，但未能给我启发。再接着就是在代码中强行用拼接好的URL与复制打印的URL判断是否相等，果然不相等，同时打印字符串16进制进行对比，终于发现了那深坑。不到三小时实现接口，但是为了发现这个错误，我那天的剩余时间全部搭进去啦。

这里有必要提下两种不同的API，[阿里大于](https://open.taobao.com/doc2/apiDetail.htm?apiId=25450)已升级为[阿里云旗下云通信品牌](https://help.aliyun.com/document_detail/55284.html)，现在应该是买不了阿里大于的短信服务吧，只能买升级后的，我就是这样。两者有关系但又不兼容，实在是大坑，起初我就是用老的阿里大于Golang第三方SDK去尝试发送短信，总是回复错误！
```JSON
{
    "error_response": {
        "code": 11,
        "msg": "Insufficient isv permissions",
        "sub_code": "isv.permission-api-package-limit",
        "sub_msg": "scope ids is 11022 11600 11863",
        "request_id": "eonzx7q4qjvg"
    }
}
```
搜索了半天才发现真是风马牛不相及啊。

老的阿里大于Golang第三方SDK
[https://github.com/ltt1987/alidayu](https://github.com/ltt1987/alidayu)
[https://github.com/northbright/alidayu](https://github.com/northbright/alidayu)
