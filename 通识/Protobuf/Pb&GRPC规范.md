本篇分为两节，一部份是GRPC的规范，另一部份是Protobuf的规范。我们绝大部分的移动应用都使用了grpc作为传输协议，剩下的webview运行时则还是使用传统的http协议。grpc目前的兼容性还比较差，所以长期内我们还得提供双栈的服务。

# GRPC规范

grpc本质上和web api有很大的相似之处，因此绝大部分的规范是直接继承自web api的规范的，请移步查看，这里只列出有所不同的地方。主要的区别在于命名规范和数据类型（数据类型的问题会在Protobuf章节详细展开说明）和错误处理。

## 命名规则

为了方便我们进行反向代理的处理，grpc的命名空间必须以service.xx的形式来命名（好比web api项目中，我们要求用/api作为所有路径的前缀一样）

与Java类似，Service以及方法名使用小驼峰风格进行命名。

在web api中，我们通常使用restful api，但是在grpc中没有类似HTTP Method之类的标识，因此我们需要将请求的动作写在函数名称上。同时，为了区分request和response，对应的传输对象需要加上Req或者Reply后缀，看如下示例：

```protobuf
package service.user; // 使用service.xxx表示命名空间，后面使用小驼峰形式

// 服务则使用xxxxService的驼峰命名法，大写开头，service的角色相当于是编程语言中的class
service UserService {
	/* 
	1，rpc同样是使用驼峰命名法，其中必须带上“动作”，例如此处的get就表示是获取xx信息。
		同样的道理，也可以有addUserInfo，updateUserInfo等等
	2，request和response必须单独创建message对象，即便里面有很多字段是复用的也不可以重复用。
		因为你永远无法断定以后这边的功能是否会进行修改，容易引起强耦合的问题。请看后面的例子
	3，命名中严禁使用编程语言的关键字，例如select，interface，class，description，message之类的
		例如，你要表达“聊天消息”的时候，不应该直接起名message，用chatMessage会更好
	 */
	rpc getUserInfo (GetUserInfoReq) returns (GetUserInfoReply);
	
	/*
	1，其实，上面的GetUserInfoReply和UpdateUserInfoReq肯定有相当大一部份字段是一样的
		但是严禁这俩直接使用UserInfoDto这个对象，至少应该在外面包裹一层
		看下方几个message的定义即可
	*/
	rpc updateUserInfo (UpdateUserInfoReq) returns (UpdateUserInfoReply);
}

// 把共用的对象抽象出来，并且加上Dto后缀，以防跟编程语言里面的对象出现重名的问题
message UserInfoDto {
 // 省略内容
}

// 预留以后可能会有扩展的可能，所以所有rpc方法的Req和Reply都必须是独立的，不能偷懒复用
message GetUserInfoReply {
	UserInfoDto userInfo = 1;
}

message UpdateUserInfoReq {
	UserInfoDto userInfo = 1;
}
```

总结来说，就是以下几个规则：

### 1，命名空间以service.开头

### 2，service与rpc遵循小驼峰规则

### 3，rpc命名必须带上“动作”

### 4，传输对象必须用Req、Reply、Dto等后缀进行功能区分

### 5，严禁复用Req与Reply



## 权限认证方式

这个部份完全照搬web api中常用的套路，即jwt。使用grpc的拦截器或直接在网关处获取http header中的Authorization字段，然后按照路径和jwt进行判断即可。

## 错误处理

由于不同平台对于grpc的实现都存在很大差异，并且grpc自身也没有关于错误处理的标准规范，因此我们习惯性使用公司自己规定的一个“村规”。我们把grpc里面的错误分成两大类，一种是简单错误，也就是说这个请求只有二元的结果，要么成功，要么失败，并且不对失败的原因和内容深究；另一种则是复杂错误，可能需要返回相对应的错误信息以供前端处理，那么我们就需要用到protobuf的oneof特性进行处理。

### 简单错误

对于简单错误，调用方只需要知道是“成功”还是“失败”就行了，不需要管究竟是什么原因失败的。这样的应用场景是很多的，举个例子，上面的getUserInfo接口，获取到了固然好，没获取到也就是一个弹窗报错就完事，并不需要去深究发生了什么错误。这种情况下，我们倾向于完全不作处理，让Rpc抛出运行时异常就可以了。

### 复杂错误

有的情况下，错误是需要返回具体消息来提示用户或者展示界面的。比如说，登录接口，有可能是因为输错密码，也有可能是因为这个用户被锁定，或者等等一系列的问题……看下面代码的例子

```protobuf
service UserService {
	rpc login(LoginReq) returns (LoginReply);
}
// 忽略一些细节，我们直接讲关于LoginReply的错误处理

message LoginReq {
	// 首先我们可以在定义一个错误信息对象
	enum LoginErrorCode {
		WrongPassword = 1;
		InvalidUser = 2;
		// ...
	}
	
	message LoginErrorMessage {
		LoginErrorCode errCode = 1;
		string errMsg = 2;
	}
	
	message LoginSuccessMessage {
		// ...
	}
	
	// 此时这个rpc返回的值就是一个oneof对象了
	// 此时前端就可以非常清楚知道发生了什么错误并且进行相对应的处理
	oneof loginResult {
		LoginSuccessMessage loginSuccessMessage = 1;
		LoginErrorMessage loginErrorMessage = 2;
	}
}
```

想必看到这里，很多人会觉得这样十分繁琐，直接raise一个RpcException或者StatusException不就好了吗？我大概讲如下的几个理由：

1，Exception并非在所有平台都有实现，比如说Kotlin里面的wire-grpc，还有swift里面的grpc，它们对这个Exception的态度是比较暧昧的，因此这个就会出现兼容性问题，导致在不同平台或者不同框架下的体验差异，这个是我们不希望看到的。

2，把更多的错误检查留在了编译期，而不是运行期。举个例子来说，如果我们用StatusException的话，它必须提供两个值，一个是int status，另一个是string errorMessage。关键问题就在这个int类型的status，仔细查看上面的代码，我把错误类型是定义为枚举类型。int类型的错误，你只能在代码里面有if else或者switch进行很粗暴的判断，一旦后面错误类型多了起来，或者说后端跟前端约定的错误码发生变化，这里就是屎山代码，完全没有办法维护。

换句话说，如果我用枚举进行表示的话，一方面枚举类型属于强类型，如果这里的代码发生改变，一定会引发编译期错误，从而让程序员尽早发现错误，而避免把bug带进去运行时。而且，因为枚举是强类型，即便以后的case很多，它的可维护性也肯定是远远强于那些“约定”的整数。

3，大大增强可读性，减少后端和前端的“纸面”约定。这一点会在后面说到Pb的时候着重强调。

# Protobuf规范

protobuf是一个非常好的“api文档”，它既可以当代码用，也可以当接口文档用，一个构建良好，结构清晰的Protobuf文件，要远远强于一个所谓的“文档”的表达能力。这一点相信你们看到那些web api的接口文档的时候会深有体会。

虽然Protobuf本身声称是跨语言的，但是并不是所有编程语言都对里面的数据类型有支持，所以必须对一些特性作出限制，否则容易引发不兼容的麻烦。

## 不应该使用的数据类型

### 1，protobuf内置的timestamp类型

如果需要传输时间，我们强行规定使用ISO8601格式的时间戳，然后用string作为传输过程中的数据类型，不能使用这个timestamp类型，它很可能会搞乱时区问题。

### 2，无符号整型（看情况）

因为安卓系统的主流编程语言依然是基于jvm的，因此没有原生的无符号整型的支持，如果我们在protobuf中使用了uint32或者uint64类型，就有概率导致溢出问题。

当然，如果你很确定你用的平台完美支持无符号数，那你可以无视这条警告。

遇到这种问题的时候，我们应该尝试用大一级的数据类型来容纳它，例如，本来设计时这里需要用uint32类型，那么我们只需要把它换成int64类型，就一定不会出错。如果需要用到比uint64还大的数据的时候，直接使用string进行传输，前端根据对应的数据类型使用大整数算法即可。

### 3，诸如sint，fixedint之类的衍生整型

虽然谷歌“声称”这些特殊的类型在一些特定情况下性能会更好，但实际从我们的体验上来说……完全没必要，几乎察觉不到任何体感上的差异。反而这些特殊的类型还容易在某些语言中引起问题，因此我们排除这些类型。

## 坚决不应该做的操作

由于之前我们说到过，很多时候因为grpc的兼容性问题，我们需要构建grpc和web api并行的双栈系统。于是乎，就有“大聪明”会想到，既然在Pb里面定义了相关的传输对象，我是不是可以直接拿json序列化来处理对应的类？

记住这句话！千万不要干这种笨蛋事情！我说几个潜在的理由：

1，由Pb编译生成的这些类往往都含有非常多的辅助函数、后台变量、以及反射之类的东西。这样的一个类被json序列化出来的结果是完全未知的，甚至有可能把源代码信息甚至二进制信息给带出来，给网络攻击创造机会。

2，一些protobuf里面的特殊类型未必可以被json序列化器识别，例如repeated，optional之类的字段，json序列化根本无法保证相关的东西能够正常工作，从而导致一些诡异的bug。

3，违反单一职责原则……虽然说得很牵强，但是一个正经的项目的确不应该这么做。

## 需要注意的操作

### 1，处理时间类型

之前说过，我们公司内统一用ISO8601格式的字符串处理时间。

### 2，巧用optional修饰

optional是一个字段的修饰符，可以表明某个字段是否是必填项目，这个可以非常友好地提示前端人员是否需要传递某些参数。

### 3，关于之前提到的错误处理

### 4，关于之前提到的命名空间问题

### 5，慎用bytes类型

一般而言，除开某些特殊场景下（比如说传递一个hash之后的digest result），我们不应该在接口中定义二进制数据的字段。因为往往用到这个字段，都意味着你需要使用文件上下载功能了。我们不太希望程序直接发送二进制对象给后端（除非涉及文件上传或文件下载），因为非常容易引发安全事故，我们无法保证每一个程序员都能写出彻底安全的文件上下载服务，因此只能作出限制。

关于文件上下载问题，我们公司内部有统一的解决方案KCos框架，因此严禁在除KCos框架之外的接口中发送二进制对象，容易引发安全问题。