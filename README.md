# Cordova/PhoneGap 微信分享插件

Inspired by https://github.com/xu-li/cordova-plugin-wechat  
其实本来 fork 了, 但因为一方面没有支持 Windows Phone, 另一方面 Android 的支持也很有限, 所以重写了.

支持 iOS, WP, Android, 都有回调 (WP 有限制, 具体看下面的例子). 分享内容只支持文本, 图片, 和链接.

另外需要注意的是 Android 不仅需要审核通过, 还需要那什么签名吻合, 所以得要 release 的 keystore 来签名.
关于这个问题写了个指南, 如果 Android 搞不定的可以看看
[Android 微信 SDK 签名问题](https://github.com/vilic/cordova-plugin-wechat/wiki/Android-%E5%BE%AE%E4%BF%A1-SDK-%E7%AD%BE%E5%90%8D%E9%97%AE%E9%A2%98).
因为 Android 的开放性, 可能是出于安全考虑, 微信 SDK 除了核对应用包名外, 还会核对应用签名, 所以调试 Android 时, 需要保证应用签名与提交审核的签名一致.

首先, 应用务必要通过审核. 至于审核后修改签名是否立即生效, 我没有做验证.

获得最终可用的应用签名的前提是, 应用是以自己的生成的 keystore 签名的, 所以第一个问题应该是, 如何生成自己的 keystore.

JDK 有一个叫 keytool 的工具可以做这个, 一般情况下既然 Cordova 能正常用, 默认 JDK 已经加入 PATH 了, 那么可以直接运行下面的命令.

keytool -genkey -alias [别名] -keyalg RSA -validity 20000 -keystore [文件名.keystore]
别名要记下来, 之后会用到.

执行该命令后会要求输入一些信息, 除了密码不要乱填, 其他应该怎么填都可以. 密码貌似有两个, 一个是 keystore 的密码, 一个是 alias 的密码. 当然还有最后确认的时候要填 yes (多半 y 也可以).

现在就算有一个 keystore 了, 把这个文件存到一个安全的地方.

现在打开 platforms/android/ 目录, 新建一个文件 ant.properties, 里面写上

key.store=[到 keystore 文件的路径]
key.alias=[keystore 的别名]
key.store.password=[keystore 的密码]
key.alias.password=[keystore 别名对应的密码]
到这里, 准备工作就基本就绪了. 执行下面的命令在设备上部署应用:

cordova run android --release --device
要不要加 --device 可以根据自己的情况来, --release 是一定要加的.

应用部署完成后, 需要在设备上安装下面的这个 apk, https://github.com/mobileresearch/weibo_android_sdk/blob/master/app_signatures.apk?raw=true

安装完成后执行, 输入自己应用的包名, 就可以获得一串签名了. 小心仔细地把签名填写到微信平台 Android 下的相关位置并提交.

到这里, 应该就没有大问题了, 注意需要调试微信相关功能的时候记得用加上 --release, wishes~

## 安装

一定不要忘记加上后面的 `--variable APP_ID=********`.

```sh
cordova plugin add com.wordsbaking.cordova.wechat --variable APP_ID=[你的APPID]
```

另外貌似 Cordova 的变量信息是按平台保存的, 如果安装插件时尚未添加某个平台, 即便之前加上了变量,
之后添加平台时依旧会报错. 此时可以先卸载插件, 添加平台后带上变量重新安装.

如果是 Visual Studio Tools for Apache Cordova, 可以这样配置 App ID:

```xml
<vs:plugin name="com.wordsbaking.cordova.wechat" version="0.2.9">
    <param name="APP_ID" value="[你的APPID]" />
</vs:plugin>
```

## 配置

其实安装后就可以用了! 零配置哟哈哈哈! 下面列一些注意事项.

### config.xml

可能需要添加一些权限, 比如安卓貌似是要添加这些:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
```

反正我没添加, debug/release 都没问题, 不知道提交商店会不会有情况.

### iOS 相关: libWeChatSDK.a

src/ios/libWeChatSDK.a 这个文件有两个版本, 一个是 iPhone Only 的, 要小一些, 应该是最后生产环境用的.
我放进去的是完整版本, 要大一倍 (应该是包含了 x86 架构方便模拟器 debug), 可以自己去下载官方 SDK
然后替换掉 platforms/ios/应用名称/Plugins 目录下的 libWeChatSDK.a.

## 使用

```javascript
// 在 device ready 后.
WeChat
    .share('文本', WeChat.Scene.session, function () {
        console.log('分享成功~');
    }, function (reason) {
        console.log(reason);
    });

// 或者 (更多选项见后).
WeChat
    .share({
        title: '链接',
        description: '链接描述',
        url: 'https://wordsbaking.com/'
    }, WeChat.Scene.timeline, function () {
        console.log('分享成功~');
    }, function (reason) {
        // 分享失败
        console.log(reason);
    });

// WP 下有个问题是, 回调会调用 WP 中的另一个页面,
// 而当回调完成返回 Cordova 的页面时页面会重新加载.
// 为了让 WP 能处理回调结果, 另外添加了方法 getLastResult.
// 其他平台该方法永远失败并且 reason 的值为 'NO_RESULT'.
WeChat
    .getLastResult(function () {
        console.log('分享成功~');
    }, function (reason) {
        if (reason == 'NO_RESULT') {
            // 正常加载
        } else {
            // 分享失败
            console.log(reason);
        }
    });
```

## API 定义

```typescript
declare module WeChat {
    /** 分享场景 */
    enum Scene {
        /** 用户自选, 安卓不支持 */
        chosenByUser,
        /** 聊天 */
        session,
        /** 朋友圈 */
        timeline
    }

    /** 多媒体分享类型, 目前只支持 image, webpage */
    enum ShareType {
        app = 1,
        emotion,
        file,
        image,
        music,
        video,
        webpage
    }
    
    /** 分享选项 */
    interface IMessageOptions {
        /** 多媒体类型, 默认为 webpage */
        type: ShareType;
        /** 标题 */
        title?: string;
        /** 描述 */
        description?: string;
        /** 缩略图的 base 64 字符串 */
        thumbData?: string;
        /** 分享内容的 url, type 为 image 是就是图片的 url, 为 webpage 时就是链接的 url */
        url?: string;
        /** 分享内容的 base 64 字符串, 与 url 二选一 */
        data?: string;
    }

    // 分享.
    function share(text: string, scene: Scene, onfulfill: () => void, onreject: (reason) => void): void;
    function share(options: IMessageOptions, scene: Scene, onfulfill: () => void, onreject: (reason) => void): void;
    
    // 下面两个是我自己用的哈哈哈, 因为需要用到我的 ThenFail Promise 库.
    function share(text: string, scene: Scene): ThenFail<void>;
    function share(options: IMessageOptions, scene: Scene): ThenFail<void>;

    // 用于 WP 下获得回调结果.
    function getLastResult(onfulfill: () => void, onreject: (reason) => void): void;
    
    // 这个也需要 ThenFail 的库, 可以忽略.
    function getLastResult(): ThenFail<void>;
}
```
