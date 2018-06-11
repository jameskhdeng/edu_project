# 腾讯互动课堂SDK（TICSDK）集成使用文档
## 1. 简介
腾讯互动课堂（Tencent Interact Class，TIC）SDK 是一个提供在线教育场景下综合解决方案的 iOS 静态库，它对`ILiveSDK`、`IMSDK`和`TXBoardView.framework`、`COSSDK`  四个SDK进行了集成封装，提供了【多人音视频】，【多人即时通信】，【多人互动画板】【文档云端转码预览】等功能。适用于在线互动课堂，在线会议，你画我猜等场景。

> 注：由于在线课堂场景下老师主要在PC端进行操作，所以移动端TICSDK相较于PC端SDK功能会有少部分缺失，主要集中在文档处理相关功能

## 2. 集成SDK

`TICSDK`支持 iOS8+ 系统，集成方式分为 Cocoapods 集成（推荐）和手动集成，集成完之后还需进行相应的工程配置。

#### Cocoapods 集成（推荐）

在 Podfile 文件中加入

```
pod 'TICSDK'    
```

安装

```
pod install // 由于SDK源文件较大，这步可能需要等待几分钟
```
  
如果无法安装 SDK 最新版本，运行以下命令更新本地的 CocoaPods 仓库列表

```
pod repo update
```
#### 手动集成
[下载TICSDK]() ，将其拖进工程中，并添加以下依赖库

|需添加依赖库|
|---|
|Accelerate.framework|
|AssetsLibrary.framework|
|AVFoundation.framework|
|CoreGraphics.framework|
|CoreMedia.framework|
|CoreTelephony.framework|
|CoreVideo.framework|
|ImageIO.framework|
|JavaScriptCore.framework|
|OpenAL.framework|
|OpenGLES.framework|
|QuartzCore.framework|
|SystemConfiguration.framework|
|VideoToolbox.framework|
|libbz2.tbd|
|libc++.tbd|
|libiconv.tbd|
|libicucore.tbd|
|libprotobuf.tbd|
|libresolv.tbd|
|libsqlite3.tbd|
|libstdc++.6.tbd|
|libstdc++.tbd|
|libz.tbd|
|libstdc++.6.0.9.tbd|

#### 工程配置
为了工程能够正常编译，需要修改以下工程配置：

* 在`Build Settings` -> `Other Linker Flags`里添加选项 `-ObjC`
* 在`Build Settings` 中将 `Allow Non-modular includes in Framework Modules`设置为`YES`
* 在`Build Settings` 中将 `Enable Bitcode`设置为`NO`
* 由于要用到手机的相机和麦克风，所以别忘了在项目的`info.plist`文件中增加`Privacy - Camera Usage Description`和`Privacy - Microphone Usage Description`两项。
 
## 3. 使用SDK
### 3.1 头文件概览

先总体说明下SDK中暴露的公开头文件的主要功能：

类名 | 主要功能
--------- | ---------
TICSDK.h | 整个SDK的入口类，提供了SDK【初始化】以及【获取版本号】的方法
TICManager.h | 互动课堂管理类，互动课堂SDK对外主要接口类，提供了【添加白板】、【登录/登出SDK】、【创建/加入/销毁课堂】、【音视频操作】、【IM操作】等接口
TICClassroomOption.h | 加入课堂时的课堂配置类，主要用来配置加入课堂时的角色（学生 or 老师）、是否自动开启摄像头，麦克风等，另外课堂配置对象还带有两个可选的代理对象，一个是复制监听课堂内部事件，另一个则负责监听课堂内的IM消息
TICFileManager.h | 文件管理类，内部封装了腾讯云对象云存储COSSDK，负责文件（PPT、wrod、Excel、pdf、图片等）的上传、下载、在线转码预览等（移动端目前只支持上传和下载）

### 3.2 使用流程

TICSDK使用的一般流程如下：

<img src="https://main.qcloudimg.com/raw/ad9baefa8be1c7da8d82e98508bda4a0.png" width = 60% height = 60% alt="使用流程" align=center />


<!--```flow
st=>start: 1. 初始化【initSDK:accountType:】
op0=>operation: 2. 配置COS云存储（可选）【configCOS:】
op1=>operation: 3. 登录【loginWithUid:userSig:】
op2=>operation: 4. 创建课堂【createClassroomWithRoomID:】
op3=>operation: 5. 加入课堂（设置课堂事件代理和IM消息代理）【joinClassroom:option:】
op4=>subroutine: 6.1 创建并添加白板，进行白板相关操作【addBoardView:】
op5=>subroutine: 6.2 音视频相关操作
op6=>subroutine: 6.3 Im相关操作
op7=>operation: 7. 退出课堂【quitClassroomSucc:】
e=>end: 8. 登出【logout:】


st->op0->op1->op2->op3->op4->op5->op6->op7->e

```-->
 
> 其中：
> 步骤2为可选步骤，如果您的APP中需要上传课件，图片等功能，则需要配置COS云存储；
> 步骤4为老师端特有步骤，学生在得知课堂ID之后，可直接加入课堂；
> 步骤6.1、6.2、6.3 代表课堂内操作，顺序不固定

下面将SDK按照功能划分，遵循一般的使用顺序，介绍一下`TICSDK`中各功能的使用方法和注意点:

### 3.3 初始化SDK
要使用`TICSDK`，首先得进行初始化，初始化方法位于`TICSDK`单例类中，先导入头文件`<TICSDK/TICSDK.h>`该头文件包含了TICSDK中所有公开的头文件，所以导入这一个文件就行：

```objc
> TICSDK.h (该行表示方法所处文件名，下同)

// 导入头文件
#import <TICSDK/TICSDK.h>

/**
@brief 初始化SDK

@param SDKAppID    腾讯云控制台注册的应用ID
@param accountType 腾讯云控制台注册的应用的账号类型
@return 初始化结果，0代表成功，其他代表失败(返回错误码 8021，表示参数无效)
*/
- (int)initSDK:(NSString *)SDKAppID accountType:(NSString *)accountType;

```
初始化方法很简单，但是开发者在初始化之前必须保证已经在[腾讯云后台](https://console.cloud.tencent.com/rav)注册成功，并创建了应用，这样才能拿到腾讯云后台分配的SDKAppID和accountType。

### 3.4 COS配置（可选）
COS为[腾讯云对象存储](https://cloud.tencent.com/document/product/436/6225)，如果您的APP中需要用到上传图片，文件到白板上展示的功能，则需要先在腾讯云对象存储开通了服务，然后再在SDK中将相关参数配置好，TICSDK内部会将调用SDK接口上传的图片，文件上传到您配置的COS云存储桶中。

具体配置接口如下：

```objc
> TICFileManager.h

/**
 COS配置类，其属性参数都可从腾讯云COS控制台获取到
 */
@interface TICCosConfig : NSObject

/// @brief COS服务的appId，用以标识资源
@property (nonatomic, copy) NSString *cosAppID;
/// @brief 存储桶名称
@property (strong, nonatomic) NSString *bucket;
/// @brief 服务地域名称
@property (nonatomic, copy) NSString *region;
/// @brief 开发者拥有的项目身份识别 ID，用以身份认证
@property (nonatomic, copy) NSString *secretID;
/// @brief 开发者拥有的项目身份密钥
@property (nonatomic, copy) NSString *secretKey;

@end

/**
 @brief COS存储配置

 @param config 配置对象
 @see TICCosConfig
 @return 0 配置成功，否则配置失败(返回错误码 8021，表示参数无效)
 */
- (int)configCOS:(TICCosConfig *)config;
```

### 3.5 登录/登出
初始化完成之后，因为涉及到IM消息的收发，所以还必须先登录：

```objc
> TICManager.h

/**
 @brief 登录SDK
 
 @param uid    用户id
 @param userSig    用户签名（由腾讯云后台生成）
 */
- (void)loginWithUid:(NSString *)uid userSig:(NSString *)userSig succ:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;
```
该方法需要传入两个参数，uid和userSig，uid为用户ID，userSig为腾讯云后台用来鉴权的用户签名，相当于登录TICSDK的用户密码，需要开发者服务器遵守腾讯云生成userSig的规则来生成，并传给客户端用于登录，详情请参考：[生成签名](https://cloud.tencent.com/document/product/647/17275)

> 注意：
> 1. 开发调试阶段， 开发者可以使用腾讯云实时音视频控制台的开发辅助工具来生成临时的uid和userSig用于开发测试
> 2. 如果此用户在其他终端被踢，登录将会失败，返回错误码（ERR_IMSDK_KICKED_BY_OTHERS：6208）。为了保证用户体验，建议开发者进行登录错误码 ERR_IMSDK_KICKED_BY_OTHERS 的判断，在收到被踢错误码时，提示用户是否重新登录。
> 3. 如果用户保存用户票据，可能会存在过期的情况，如果用户票据过期，login 将会返回 70001 错误码，开发者可根据错误码进行票据更换。
> 4. 关于以上错误的详细描述，参见[用户状态变更](https://cloud.tencent.com/document/product/269/9148#.E7.94.A8.E6.88.B7.E7.8A.B6.E6.80.81.E5.8F.98.E6.9B.B4)。


登出方法比较简单，如下：

```objc
> TICManager.h

/**
 @brief 登出SDK
 */
- (void)logout:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;
```

### 3.6 课堂管理

* 创建课堂

登录成功之后，就可以创建或者加入课堂了，创建课堂接口如下，创建课堂时需要传入一个`roomID`参数（roomID是一个课堂的唯一标识）：

```objc
> TICManager.h

 /**
 @brief 创建课堂
 
 @param roomID 课堂ID，课堂唯一标识（必须为正整数）
 */
- (void)createClassroomWithRoomID:(int)roomID Succ:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;
```

创建课堂接口，只是根据传进去的roomID创建了一个IM群组，并进行了一些准备工作，老师端创建课堂后还需调用`加入课堂`方法加入课堂。

* 加入课堂

```objc
> TICManager.h


/**
 @brief 加入指定roomID的课堂

 @param configOption 加入课堂配置项
 @see TICClassroomOption
 @discussion 注意：TICClassroomOption的 controlRole 参数必填，该参数代表进房之后使用什么规格音视频参数，参数具体值为客户在腾讯云实时音视频控制台画面设定中配置的角色名（例如：默认角色名为user, 可设置controlRole = @"user"）
 */
- (void)joinClassroomWithOption:(TICClassroomOption * (^)(TICClassroomOption *option))configOption succ:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;
```
该接口需要传入一个配置block `configOption`，该block接收一个方法内部创建好的`TICClassroomOption`默认配置对象，开发者需在该block中修改自定义配置，然后将修改后的option返回，`TICClassroomOption`类如下：

```objc
/**
 课堂配置类
 */
@interface TICClassroomOption : ILiveRoomOption
@property (nonatomic, assign) int roomID; // 课堂ID，课堂的唯一标识（必须为正整数）
@property (nonatomic, assign) TICClassroomRole role; // 课堂内角色枚举（老师 or 学生）
@property (nonatomic, weak) id<TICClassroomEventListener> eventListener; // 课堂事件监听对象
@property (nonatomic, weak) id<TICClassroomIMListener> imListener; // IM事件监听对象

@end
```
`roomID`即为课堂的唯一标识；`role`表示加入课堂后的角色身份（老师或学生，一般创建课堂的人为老师，其他人应该以学生身份加入课堂）。

另外该类还有两个代理对象，用来监听课堂内的一些事件，这个我们后面再说。

为了保证课堂内的正常逻辑和事件都能被监听到，进房时`TICClassroomOption`的这些属性都是必填参数，然后还有一个必填参数为**`controlRole`**，继承自父类`ILiveRoomOption`，该参数代表进房之后使用哪些音视频参数，参数具体值为客户在腾讯云实时音视频控制台画面设定中配置的角色名（例如：默认角色名为user, 可设置controlRole = @"user"），实例代码如下：

```objc
[[TICManager sharedInstance] joinClassroomWithOption:^TICClassroomOption *(TICClassroomOption *option) {
    option.roomID = #房间号#;
    option.role = kClassroomRoleStudent;
    option.eventListener = #课堂事件监听对象#;
    option.imListener = #课堂消息监听对象#;
    option.controlRole = @“user”;
    return option;
} succ:^{
    NSLog(@"进房成功");
} failed:^(NSString *module, int errId, NSString *errMsg) {
    NSLog(@"进房失败：%d %@", errId, errMsg);
}];

```

* 退出课堂

```objc
> TICManager.h

/**
 @brief 退出课堂
 @discussion 在创建该课堂的老师退出课堂后，课堂相关的音视频房间、IM群组将会被销毁
 */
- (void)quitClassroomSucc:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;
```

学生退出课堂时，只是本人退出了课堂，老师调用`退出课堂`方法退出课堂时，该课堂将会被销毁，另外退出课堂成功后，课堂内的资源包括课堂的`roomID`将会被回收，所以开发者应尽量保证在加入另一个课堂前，已经退出了前一个课堂。

### 3.7 白板相关操作

TICSDK 中只有一个关于白板的接口，就是添加一个白板视图对象：

```objc
> TICManager.h

/**
 @brief 添加白板到 TICManager【使用白板必调】
 @discussion 方法内部不会对 TXBoardView 对象进行强引用，只是将boardView的代理设置为 TICManager
 
 @param boardView 用户创建的白板对象
 */
- (void)addBoardView:(TXBoardView *)boardView;
```
该方法只是将传进来的白板视图参数与TICSDK进行了绑定（但是不会强引用），将boardView的代理对象设置为了TICManager，并实现了其代理方法，外部无需关心。

开发者使用时，应该创建一个boardView对象，并将其添加到TICManager中（同时只能添加一个，重复添加以后添加的为准），然后直接调用TXBoardView中的接口来操作白板即可，详见[TXBoardView白板SDK使用手册](./iOS白板SDK使用手册.md)。


#### 3.8 IM相关操作

IM相关的接口封装于腾讯云通信SDK`IMSDK`，同样，TICSDK中也只封装了一些常用接口：

```objc
/**
 @brief 向当前课堂发送群聊文本消息
 
 @param text 文本内容
 */
- (void)sendGroupTextMessage:(NSString *)text succ:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;

/**
 @brief 向当前课堂发送群聊自定义消息
 
 @param data 自定义data
 */
- (void)sendGroupCustomMessage:(NSData *)data succ:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;

/**
 @brief 发送C2C（单聊）文本消息
 
 @param identifer 消息接收者
 @param text 发送的文本内容
 */
- (void)sendC2CTextMessage:(NSString *)identifer text:(NSString *)text succ:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;

/**
 @brief 发送C2C（单聊）自定义消息
 
 @param identifer 消息接收者
 @param data 自定义data
 */
- (void)sendC2CCustomMessage:(NSString *)identifer context:(NSData *)data succ:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;
```
课堂内成员在调用以上方法发送消息时，会触发IM事件，如果在加入课堂前设置了IM事件监听代理 `id<TICClassroomIMListener> imListener;`，一端发送IM消息时，另一端就可以在课堂内IM消息回调对应方法中得到通知:

```objc
/**
 课堂内IM消息回调方法
 */
@protocol TICClassroomIMListener <NSObject>

@optional

/**
 @brief 收到群聊文本消息
 
 @param fromId 消息发送方的ID
 @param text 自定义消息内容
 */
- (void)onRecvGroupTextMsg:(NSString *)fromId text:(NSString *)text;


/**
 @brief 收到群聊自定义消息
 
 @param fromId 消息发送方的ID
 @param data 自定义消息内容
 */
- (void)onRecvGroupCustomMsg:(NSString *)fromId context:(NSData *)data;

/**
 @brief 收到C2C单聊文本消息

 @param fromId 消息发送方的ID
 @param text 文本消息内容
 */
- (void)onRecvC2CTextMsg:(NSString *)fromId text:(NSString *)text;

/**
 @brief 收到C2C单聊自定义消息

 @param fromId 消息发送方的ID
 @param data 自定义消息内容
 */
- (void)onRecvC2CCustomMsg:(NSString *)fromId context:(NSData *)data;

@end
```

这4个代理方法，分别对应了前面4个消息发送的方法，对应类型的消息会在对应类型的代理方法中回调给课堂内所有成员（发消息本人除外），其他端收到后可以将消息展示在界面上。

### 3.9 音视频相关操作

这部分功能封装于腾讯云实时音视频SDK `ILiveSDK`，TICSDK中只封装了一些常用的接口：打开/关闭摄像头、麦克风，切换当前相机方向等，如下：

```objc
/**
 @brief 打开/关闭 摄像头
 
 @param cameraPos 摄像头位置枚举
 @param isEnable YES:打开 NO:关闭
 */
- (void)enableCamera:(cameraPos)cameraPos enable:(BOOL)isEnable succ:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;

/**
 @brief 切换当前相机方向（前置切到后置，后置切到前置）
 */
- (void)switchCamera:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;

/**
 @brief 打开/关闭 麦克风
 
 @param isEnable YES:打开 NO:关闭
 */
- (void)enableMic:(BOOL)isEnable succ:(TCIVoidBlock)succ failed:(TCIErrorBlock)failed;
```

课堂内成员在进行打开/关闭摄像头、麦克风操作时，会触发音视频事件，如果在加入课堂前设置了课堂事件监听代理 `id<TICClassroomEventListener> eventListener`，一端进行音视频操作时，另一端就可以在课堂内音视频事件回调中得到通知：

```objc

@protocol TICClassroomEventListener <NSObject>
@optional
/**
 * @brief 课堂内音视频事件回调（注意该回调中的进出房间通知不准确，已废弃进出房间事件通知）
 * @param  event   事件类型
 * @param  users   用户ID数组
 */
- (void)onUserUpdateInfo:(QAVUpdateEvent)event users:(NSArray *)users;
...

@end
```
课堂内的音视频事件都会通过该方法回调到其他端（包括操作者的），event表示事件类型（开关摄像头等），user表示触发事件的用户ID，其他段触发回调之后，可以根据事件类型，进行相应的处理，比如，收到开摄像头事件，就添加一个对应用户的渲染视图，收到关摄像头时间，就移除对应用户的渲染视图（详细用法可以参照demo）。


### 3.10 课堂内其他事件监听

进入课堂的配置对象中的课堂事件监听代理还有一些其他的协议方法：

```objc
/**
 *  @brief 有人加入课堂时的通知回调
 *
 *  @param members 加入成员的identifier（NSString*）列表
 */
-(void)onMemberJoin:(NSArray*)members;

/**
 *  @brief 有人退出课堂时的通知回调
 *
 *  @param members 退出成员的identifier（NSString*）列表
 */
-(void)onMemberQuit:(NSArray*)members;

/**
 * @brief 课堂被解散通知
 * @discussion 老师端主动解散课堂时，老师端不会收到该通知，还在课堂内的学生端会收到；只有后台解散课堂时，老师端才有可能收到
 */
-(void)onClassroomDestroy;
```

以上协议方法分别代表有人加入课堂，有人退出课堂和课堂被解散的回调，开发者可以根据自己的业务需求，对回调事件进行相应的处理，比如：在收到课堂解散回调时（老师退出课堂即触发该回调），课堂内的学生端可以弹出一个提示框，提示学生课堂已经结束。




