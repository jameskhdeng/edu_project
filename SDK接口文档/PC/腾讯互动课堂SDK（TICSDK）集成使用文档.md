# 腾讯互动课堂SDK（TICSDK）集成使用文档
## 1.1 简介
腾讯互动课堂（Tencent Interact Class，TIC）SDK 是一个提供在线教育场景下综合解决方案的动态链接库，对`ILiveSDK`、`IMSDK`、`BroadSDK`和`COSSDK`  四个SDK进行了集成封装，提供了【多人音视频】，【多人即时通信】，【多人互动画板】，【文档云端转码预览】等功能。适用于在线互动课堂，在线会议，你画我猜等场景。

## 1.2 准备工作
TICSDK使用了实时音视频服务（iLiveSDK）、云通讯服务（IMSDK）、COS服务等腾讯云服务能力，在使用腾讯互动课堂服务时，请对点时间了解以上服务的基本概念和基本业务流程。

[实时音视频](https://cloud.tencent.com/document/product/268/8424)

[云通讯服务（IMSDK）](https://cloud.tencent.com/document/product/269/1504)

[COS服务](https://cloud.tencent.com/document/product/436/6225)

## 2. 集成SDK
待补充
## 3. 使用SDK
### 3.1 头文件概览

先总体说明下SDK中暴露的公开头文件的主要功能：

类名 | 主要功能
--------- | ---------
TICSDK.h | 整个SDK的入口类，提供了SDK【初始化】以及【获取版本号】的方法
TICManager.h | 互动课堂管理类，互动课堂SDK对外主要接口类，提供了【添加白板】、【登录/登出SDK】、【创建/加入/销毁课堂】、【音视频操作】、【IM操作】等接口
TICClassroomOption.h | 加入课堂时的课堂配置类，主要用来配置加入课堂时的角色（学生 or 老师），另外课堂配置对象还带有三个可选的代理对象，一个是复制监听课堂内部事件，一个则负责监听课堂内的IM消息，还有一个负责监听课堂内白板消息
TICSDKCosConfig.h | COS管理类，内部封装了腾讯云对象云存储COSSDK，负责文件（PPT、wrod、Excel、pdf、图片等）的上传、下载、在线转码预览等（移动端目前只支持上传和下载）
TICWhiteboardManager.h|白板管理类，对白板BoardSDK.dll进行了封装

### 3.2 使用流程

TICSDK使用的一般流程如下：
![教师业务流程](../../资源文件/Android_主流程.png) 

 > 其中【创建课堂】为教师角色特有流程，学生角色不需调用。

下面将SDK按照功能划分，遵循一般的使用顺序，介绍一下`TICSDK`中各功能的使用方法和注意点:

### 3.3 初始化SDK
要使用`TICSDK`，首先得进行初始化，初始化方法位于`TICSDK`单例类中：

```objc
> TICSDK.h (该行表示方法所处文件名，下同)

/**
@brief 初始化TICSDK
@param iLiveSDKAppId 腾讯云控制台注册的应用ID
@param iLiveAccountType腾讯云控制台注册的应用的账号类型
@return 初始化结果，0代表成功，其他代表失败
*/
virtual int initSDK(int iLiveSDKAppId, int iLiveAccountType) = 0;

```
初始化方法很简单，但是开发者在初始化之前必须保证已经在[腾讯云后台](https://console.cloud.tencent.com/rav)注册成功，并创建了应用，这样才能拿到腾讯云后台分配的SDKAppID和accountType。

### 3.4 登录/登出
初始化完成之后，因为涉及到IM消息的收发，所以还必须先登录：

```objc
> TICManager.h

/**
@brief 登录iliveSDK
 
@param uid    用户id
@param userSig    用户签名（由腾讯云后台生成）
@param success 登录成功回调
@param err      登录错误回调
@param data   用户自定义数据
@return 登录结果，0表示成功
 */
int login(const char * id, const char * userSig, ilive::iLiveSucCallback success, ilive::iLiveErrCallback err, void* data);
```
该方法需要传入uid和userSig，uid为用户ID，userSig为腾讯云后台用来鉴权的用户签名，相当于登录TICSDK的用户密码，由腾讯云后台生成，success和err为登录SDK成功和失败回调，data为用户自定义数据

登录的流程如下：

![](https://main.qcloudimg.com/raw/e8a833d3e3e05d1402ea67e754232ff0.png)

客户端先以开发者的账号体系登录自己的服务器，然后再由开发者服务器调用腾讯云后台API，来为每一个开发者已有的账号生成对应的userSig，客户端拿到userSig之后再调用该登录方法登录TICSDK。

该流程基于腾讯云通信账号集成的独立模式，详见[官方文档](https://cloud.tencent.com/document/product/269/1508)。

当然，在开发调试阶段，用户可以在自己的腾讯云应用控制台使用开发辅助工具，来生成临时的uid和userSig用于开发测试
![](https://main.qcloudimg.com/raw/fd6da0bbe51cfa2ccf2faf1d4188c03e.jpg)


> 注意：
> 1. 如果此用户在其他终端被踢，登录将会失败，返回错误码（ERR_IMSDK_KICKED_BY_OTHERS：6208）。开发者必须进行登录错误码 ERR_IMSDK_KICKED_BY_OTHERS 的判断。
> 2. 如果用户保存用户票据，可能会存在过期的情况，如果用户票据过期，login 将会返回 70001 错误码，开发者可根据错误码进行票据更换。
> 3. 关于以上错误的详细描述，参见[用户状态变更](https://cloud.tencent.com/document/product/269/9148#.E7.94.A8.E6.88.B7.E7.8A.B6.E6.80.81.E5.8F.98.E6.9B.B4)。


登出方法比较简单，如下：

```objc
> TICManager.h

/**
@brief 登出iliveSDK
@param success 成功回调
@param err 错误回调
@param data   用户自定义数据
 */
void logout(ilive::iLiveSucCallback success, ilive::iLiveErrCallback err, void* data);
```
其中参数success和err为登录SDK成功和失败回调，data为用户自定义数据

### 3.5 课堂管理

* 创建课堂

登录成功之后，就可以创建或者加入课堂了，创建课堂接口如下，需要用户生成课堂房间roomID并传入：

```objc
> TICManager.h

/**
@brief 创建课堂
@param roomID 课堂房间ID
@param listener 创建课堂回调指针
*/
virtual void createClassroom(uint32_t roomID, IClassroomEventListener* listener) = 0;
```

创建课堂接口只是进行了一些准备工作，老师端创建课堂后还需调用`加入课堂`方法加入课堂。

* 加入课堂

```objc
> TICManager.h

/**
@brief 加入课堂
@param opt 课堂配置类对象
@param success 加入课堂成功回调
@param err 加入课堂失败回调
*/
virtual void joinClassroom(TICClassroomOption& opt, ilive::iLiveSucCallback success, ilive::iLiveErrCallback err, void* data) = 0;
```

该接口需要参数中，opt是`TICClassroomOption`对象，代表加入课堂时的一些配置：

```objc
/**
 课堂配置类
 */
class TICSDK_API TICClassroomOption
{
public:
	TICClassroomOption();
	~TICClassroomOption();

	bool getIsTeacher(); 
	void setIsTeacher(bool bTeacher); // 设置课堂内角色是否为老师

	uint32_t getRoomID;
	void setRoomID(uint32_t roomId);  ();  //设置课堂房间ID
	
	void setRoomOption(ilive::iLiveRoomOption& opt); //设置进课堂参数项
	ilive::iLiveRoomOption& getRoomOption();

	void setClassroomEventListener(IClassroomEventListener* listener); //设置监听课堂内部事件
	IClassroomEventListener* getClassroomEventListener() const;

	void setClassroomIMListener(IClassroomIMListener* listener); //设置监听课堂内IM消息
	IClassroomIMListener* getClassroomIMListener() const;

	void setClassroomWhiteboardListener(IClassroomWhiteboardListener* listener); //设置监听课堂内白板消息
	IClassroomWhiteboardListener* getClassroomWhiteboardListener();
}


/**
@brief 课堂事件监听对象
*/
class IClassroomEventListener

/**
@brief 课堂IM消息监听对象
*/
class IClassroomIMListener

/**
@brief 课堂白板消息监听对象
*/
class IClassroomWhiteboardListener
```

基础配置有3个，加入课堂时是否为老师，进入课的房间ID，以及透传给iliveSDK的roomOption参数项。该类还有三个代理对象，用来监听课堂内的一些事件，这个后面再说。

* 退出课堂

```objc
> TICManager.h

/**
@brief 退出课堂
@param success 退出课堂成功回调
@param err 退出课堂失败回调
*/
virtual void quitClassroom(ilive::iLiveSucCallback success, ilive::iLiveErrCallback err, void* data) = 0;
```

学生退出课堂时，只是本人退出了课堂，老师调用`退出课堂`方法退出课堂时，该课堂将会被销毁，另外退出课堂成功后，课堂的资源将会被回收，所以开发者应尽量保证再加入另一个课堂前，已经退出了前一个课堂。

### 3.6 COS上传相关操作

TICSDKCosConfig内部封装了COS上传所需要的CosAppId，Bucket，Region等参数，用户填好参数后通过TICManager的`setCosHandler`方法传给TICSDK。cos上传预览功能被封装在了TICManger里面，调用uploadFile将文件名路径作为参数填入即可。
```objc
> TICManager.h

/**
@brief 设置COS参数配置
@param cfg  COS配置
*/
virtual void setCosHandler(TICSDKCosConfig cfg) = 0;

/**
@brief 上传文件到cos
@param fileName   文件名
*/
virtual void uploadFile(const std::wstring& fileName) = 0;

/**
@brief 上传文件到cos
@param fileName   文件名
@param sig			cos签名
*/
virtual void uploadFile(const std::wstring& fileName, std::string& sig) = 0;

```

上传结果通过`IClassroomWhiteboardListener`的回调传给上层处理
```objc
/**
* \brief 通知文件上传进度
* \param percent	进度按百分比
*/
virtual void onUploadProgress(int percent) = 0;

/**
* \brief 通知文件上传结果
* \param success	上传结果
* \param objName	cos文件名
* \param fileName	本地文件名
*/
virtual void onUploadResult(bool success, std::wstring objName, std::wstring fileName) = 0;

/**
* \brief 通知PPT文件上传结果
* \param success	上传结果
* \param objName	cos文件名
* \param fileName	本地文件名
* \param fileId	    文件Id
*/
virtual void onFileUploadResult(bool success, std::wstring objName,std::wstring fileName, int pageCount) = 0;
```

### 3.7 白板相关操作

TICSDK 中将白板SDK封装在一个白板管理类当中，用户可在进入房间后调TICSDK.h里面的initWhiteBoard方法进行初始化，也可以自己初始化白板SDK后通过initWhiteBoard方法传入

```objc
> TICSDK.h

/**
@brief 初始化白板SDK，在加入房间之后
@param id 用户id
@param classID 课堂ID
@param parentHWnd 白板父窗口句柄
@return 结果，0表示成功
*/
virtual int initWhiteBoard(const char* id, HWND parentHWnd = nullptr) = 0;

/**
@brief 初始化白板SDK
@param boardsdk 外部初始化的sdk指针
@return 结果，0表示成功
*/
virtual int initWhiteBoard(BoardSDK* boardsdk) = 0;

/**
@brief 获取白板管理类实例指针
@return 白板管理类指针
*/
virtual TICWhiteboardManager* getTICWhiteBoardManager() = 0;
```
开发者可以通过getTICWhiteBoardManager()获得白板管理类里面封装好的方法，也可以直接调用BoardSDK.h里面的接口对白板进行操作。
```objc
> TICWhiteboardManager.h
/**
@brief 获得白板窗口句柄
*/
virtual HWND getRenderWindow() = 0;

/**
@brief 清空白板数据
*/
virtual void clearWhiteBoard() = 0;

/**
@brief 使用画板工具
@param tool  画板工具
*/
virtual void useTool(BoardTool tool) = 0;

/**
@brief 设置线宽
@param width  宽度
*/
virtual void setWidth(uint32_t width) = 0;

/**
@brief 设置颜色
@param rgba  颜色RGBA值
*/
virtual void setColor(uint32_t rgba) = 0;

/**
@brief 设置填充
@param fill  是否填充
*/
virtual void setFill(bool fill) = 0;

/**
@brief 撤销
*/
virtual void undo() = 0;

/**
@brief 重做
*/
virtual void redo() = 0;

/**
@brief 删除
*/
virtual void remove() = 0;

/**
@brief 清除白板
*/
virtual void clear() = 0;

/**
@brief 清除涂鸦
*/
virtual void clearDraws() = 0;

/**
@brief 设置白板背景
@param url  背景图地址
@param pageID 白板ID，默认为当前白板
*/
virtual void setBackground(const wchar_t *url, const char *pageID = nullptr) = 0;

/**
@brief 设置白板背景色
@param rgba  颜色RGBA值
*/
virtual void setBackgroundColor(uint32_t rgba) = 0;

/**
@brief 设置全局背景色
@param rgba  颜色RGBA值
*/
virtual void setAllBackgroundColor(uint32_t rgba) = 0;

/**
@brief 拉取离线数据
*/
virtual void getBoardData() = 0;
```

#### 3.8 IM相关操作

IM相关的接口封装于腾讯云通信SDK`IMSDK`，同样，TICSDK中也只封装了一些常用接口：

```objc
/**
@brief 发送C2C文本消息
@param identifier   消息接收者
@param msg  发送内容
@param OnSuccess 发送成功回调
@param OnError   发送失败回调
*/
virtual void sendC2CTextMsg(const char * identifier, const char * msg) = 0;

/**
@brief 发送群文本消息
@param msg  发送内容
@param OnSuccess 发送成功回调
@param OnError   发送失败回调
*/
virtual void sendGroupTextMsg(const char * msg) = 0;

/**
@brief 发送C2C自定义消息
@param identifier   消息接收者
@param msg  发送内容
@param OnSuccess 发送成功回调
@param OnError   发送失败回调
*/
virtual void sendC2CCustomMsg(const char * identifier, const char * msg) = 0;

/**
@brief 发送群组自定义消息
@param msg  发送内容
@param OnSuccess 发送成功回调
@param OnError   发送失败回调
*/
virtual void sendGroupCustomMsg(const char * msg) = 0;
```
课堂内成员在调用以上方法发送消息时，会触发IM事件，如果在加入课堂前设置了IM事件监听代理 `IClassroomIMListener`，一端发送IM消息时，另一端就可以在课堂内IM消息回调对应方法中得到通知:

```objc
/**
@brief 课堂IM消息监听对象
*/
class IClassroomIMListener

/**
* \brief 接收C2C文本消息
* \param identifier	消息发送者
* \param msg	消息内容
*/
virtual void onRecvC2CTextMsg(const char * identifier, const char * msg) = 0;

/**
* \brief 接收群组文本消息
* \param identifier	消息发送者
* \param msg	消息内容
*/
virtual void onRecvGroupTextMsg(const char * identifier, const char * msg) = 0;

/**
* \brief 接收C2C自定义消息
* \param identifier	消息发送者
* \param msg	消息内容
*/
virtual void onRecvC2CCustomMsg(const char * identifier, const char * msg) = 0;

/**
* \brief 接收群组自定义消息
* \param identifier	消息发送者
* \param msg	消息内容
*/
virtual void onRecvGroupCustomMsg(const char * identifier, const char * msg) = 0;

/**
* \brief 接收群组系统消息
* \param msg	消息内容
*/
virtual void onRecvGroupSystemMsg(const char * msg) = 0;

/**
* \brief 发送消息回调
* \param err	错误码
* \param errMsg	错误描述
*/
virtual void onSendMsg(int err, const char * errMsg) = 0;

/**
* \brief 发送白板消息回调
* \param err	错误码
* \param errMsg	错误描述
*/
virtual void onSendWBData(int err, const char * errMsg) = 0;

```

前4个代理方法，分别对应了前面4个消息发送的方法，对应类型的消息会在对应类型的代理方法中回调给课堂内所有成员（发消息本人除外），其他端收到后可以将消息展示在界面上。接下来`onRecvGroupSystemMsg`监听了课堂内房间解散消息，`onSendMsg`和`onSendWBData`则对应发普通消息和IM消息是否成功的回调。

### 3.9 音视频相关操作

这部分功能封装于腾讯云实时音视频SDK `ILiveSDK`，TICSDK中只封装了一些常用的接口：打开/关闭摄像头、麦克风，扬声器， 屏幕分享等，如下：

```objc
/**
@brief 打开/关闭摄像头
@param enable   true：打开默认摄像头；false：关闭
*/
virtual void enableCamera(bool bEnable) = 0;

/**
@brief 切换摄像头
@param cameraId   摄像头设备标识
*/
virtual void switchCamera(const char* cameraId) = 0;

/**
@brief 打开/关闭麦克风
@param enable   true：打开默认麦克风；false：关闭
*/
virtual void enableMic(bool bEnable) = 0;

/**
@brief 切换麦克风
@param deviceID   麦克风设备标识
*/
virtual void switchMic(const char* deviceID) = 0;

/**
@brief 打开/关闭扬声器
@param enable   true：打开默认扬声器；false：关闭
*/
virtual void enablePlayer(bool bEnable) = 0;

/**
@brief 打开屏幕分享(指定窗口)
@param hWnd 所要捕获的窗口句柄(NULL表示全屏)
@param fps 捕获帧率
*/
virtual void openScreenShare(HWND hWnd, uint32& fps) = 0;

/**
@brief 打开屏幕共享(指定区域)
@param left/top/right/bottom 所要捕获屏幕画面的区域的左上角坐标(left, top)和右下角坐标(right, bottom)
@param fps 捕获帧率
*/
virtual void openScreenShare(int32& left, int32& top, int32& right, int32& bottom, uint32& fps) = 0;

/**
@brief 动态修改屏幕分享的区域
@param left/top/right/bottom 所要捕获屏幕画面的区域的左上角坐标(left, top)和右下角坐标(right, bottom)
@param fps 捕获帧率
*/
virtual int changeScreenShareSize(int32& left, int32& top, int32& right, int32& bottom) = 0;

/**
@brief 关闭屏幕共享
*/
virtual void closeScreenShare() = 0;
```

课堂内成员在进行打开/关闭摄像头、麦克风操作时，会触发音视频事件，如果在加入课堂前设置了课堂事件监听代理`IClassroomEventListener`，一端进行音视频操作时，另一端就可以在课堂内音视频事件回调中得到通知：

```objc

class IClassroomEventListener

/**
* \brief 创建房间返回回调
* \param code		错误码，0为OK
* \param desc	错误描述
*/
virtual void onCreateClassroom(DWORD code, const char *desc) = 0;

/**
* \brief 视频房间断开回调
* \param reason		错误码
* \param errorinfo	错误描述
* \param data		用户自定义数据的指针
*/
virtual void onLiveVideoDisconnect(int reason, const char *errorinfo, void* data) = 0;

/**
* \brief 成员状态改变回调
* \param event_id		事件id
* \param ids		发生状态变化的成员id列表
* \param data		用户自定义数据的指针
*/
virtual void onMemStatusChange(ilive::E_EndpointEventId event_id, const ilive::Vector<ilive::String> &ids, void* data) = 0;

```
创建课堂这步通过`onCreateClassroom`方法通知上层是否成功；课堂内断线事件会通过`onLiveVideoDisconnect`方法通知给上层也便做异常处理。课堂内的成员音视频事件都会通过`onMemStatusChange`方法回调到其他端（包括操作者的），event_id表示事件类型（开关摄像头等），ids表示触发事件的用户ID集合，其他端触发回调之后，可以根据事件类型，进行相应的处理，比如，收到开摄像头事件，就添加一个对应用户的渲染视图，收到关摄像头时间，就移除对应用户的渲染视图（详细用法可以参照demo）。


