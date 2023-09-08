在生产环境中，出于安全或者其他因素的考虑，部分接口需要进行签名以验证身份信息。常见的一种的签名方式，是将相关账号和密钥发送给调用方，调用方通过账号和密钥对请求参数进行签名，提供方通过解签来验证请求是否合法。

但是在实际开发、测试过程中，手动拼接接口签名是繁琐且复杂的，对此我们可以借助于 postman 的 pre-request Script 功能使用脚本自动生成签名。下面我们将详细介绍具体实现。

### 签名规则
假设当前签名的规则如下：
1. 获取接口中要签名的参数
2. 添加 appId（分配的账号）、appSecret（分配的密钥）、timestamp（当前时间戳）、random（随机值）字段值
3. 将上述提及的字段值按照字母升序的方式进行排序，按照 `${key}=${value}` 格式序列成字符串，再使用 & 符号连接
4. 使用 sha1 算法对上述字符串进行计算，得到签名（sign 字段）
5. 将 appId、timestamp、random、sign 字段写入请求头（用于服务端解签）

### 代码实现
```javascript
;(function({Signer, setSignParams, MODE}) {
	'use strict';

    // Usage
    // 接口调用方的 appId
    const appId = 123;
    // 接口调用方的 appSecret
    const appSecret = 'secret';
    // 接口中参与签名的参数
    const signParams = ['name', 'startTime', 'endTime'];
    // 请求格式 Content-Type
    const mode = MODE.raw;
    // 生成签名信息
    let signInfo = new Signer(appId, appSecret, mode).generateSign(signParams);
    // 将签名信息写入请求的参数中
    setSignParams(mode, signInfo);
} (
    function() {
		'use strict';

        // 请求模式
        const MODE = {
            // 请求内容格式为：application/json
            raw: 'raw',
            // 普通的 get 请求
            query: 'query',
            // 请求内容格式为：multipart/form-data
            formdata: 'formdata',
            // 请求内容格式为：application/x-www-form-urlencode
            urlencoded: 'urlencoded',
        };
 
        function Signer(appId, appSecret, mode) {
            this.appId = appId;
            this.appSecret = appSecret;
            this.mode = mode;
        }
 
        Signer.prototype.generateSign = function(signParams) {
            let random = randomeInt(0, 1_000_000);
            let timestamp = new Date().getTime();
            let originParams;
 
            if (this.mode === MODE.urlencoded) {
                originParams = pm.request.body.urlencoded.toObject();
            } else if (this.mode === MODE.formdata) {
                originParams = pm.request.body.formdata.toObject();
            } else if (this.mode === MODE.raw) {
                originParams = JSON.parse(pm.request.body.raw);
            } else {
                originParams = pm.request.url.query.toObject();
            }
 
            let allParmas = signParams.map(p => [p, originParams[p]]);
            allParmas.push(...[
                ['appId', this.appId],
                ['appSignKey', this.appSecret],
                ['timestamp', timestamp],
                ['random', random],
            ]);
            allParmas.sort((p1, p2) => p1[0].localeCompare(p2[0]));
 
            let str = allParmas.map(p => `${p[0]}=${p[1]}`).join('&');
            let sign = CryptoJS.SHA1(str).toString();
 
            return {
                sign,
                timestamp,
                random,
                appId: this.appId
            };
        }
 
        function setSignParams(mode, params) {
            let newBody;
 
            if (mode === MODE.urlencoded) {
                newBody = {
                    mode: MODE.urlencoded,
                    urlencoded: object2array(Object.assign(pm.request.body.urlencoded.toObject(), params))
                };             
            } else if (mode === MODE.formdata) {
                newBody = {
                    mode: MODE.formdata,
                    formdata: object2array(Object.assign(pm.request.body.formdata.toObject(), params))
                };
            } else if (mode === MODE.raw) {
                newBody = {
                    mode: 'raw',
                    raw: JSON.stringify(Object.assign(JSON.parse(pm.request.body.raw), params)),
                    options: {
                        raw: {language: 'json'}
                    }
                };
            } else {
                return Object.entries(params)
                        .forEach(([key, value]) =>
                            pm.request.addQueryParams(`${key}=${value}`)
                        );
            }
 
            pm.request.body.update(newBody);
        }
 
        function randomeInt(min, max) {
            return Math.floor(Math.random() * (max - min + 1)) + min;
        }
 
        function object2array(obj) {
            return Object.entries(obj)
                .map(([key, value]) => {
                    return {key, value};
                });
        }
 
        return {
            Signer, setSignParams, MODE
        };
    }()
));
```

### 使用
按照之前一样的方式创建请求，将上述代码片段复制到请求的 pre-request Script 中，按照实际情况调整第一个闭包中的内容。具体代码解释如下：
```javascript
;(function({
	Signer        /* 签名类 */, 
	setSignParams /* 工具方法，将参数写入请求中 */, 
	MODE          /* 接口内容格式 Content-Type */
}) {
	'use strict';

    // Usage
    // 接口调用方的 appId
    const appId = 123;
    // 接口调用方的 appSecret
    const appSecret = 'secret';
    // 接口中参与签名的参数
    const signParams = ['name', 'startTime', 'endTime'];
    // 请求格式 Content-Type
    const mode = MODE.raw;
    // 生成签名信息
    let signInfo = new Signer(appId, appSecret, mode).generateSign(signParams);
    // 将签名信息写入请求的参数中
    setSignParams(mode, signInfo);
} (function() {
    // ....
}));
```

