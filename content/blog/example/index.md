---
title: "示例博客"
description: "只是一篇示例博客"
summary: ""
date: 2024-04-28T16:27:22+02:00
lastmod: 2024-04-28T16:27:22+02:00
draft: false
weight: 50
categories: []
tags: []
contributors: []
pinned: false
homepage: false
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---

做项目离不开表单验证，我们使用element自带的表单验证方法来进行表单验证

考虑以下表单:
```vue
<el-form class="login_form">
	<el-form-item>
		<el-input :prefix-icon="User" v-model="loginForm.username"></el-input>
	</el-form-item>

	<el-form-item>
		<el-input type="password" :prefix-icon="Lock" v- model="loginForm.password" show-password></el-input>
	</el-form-item>
	<el-button :loading="loading" class="login_btn" type="primary" size="default" @click="login">登录</el-button>
</el-form>
```

这个表单含有一个用户名和一个密码，现在我们要求使用element自带的验证方法对用户名和密码的长度进行简单的验证

### 不完全的表单验证

要使用element的表单验证，我们需要一些相应的准备工作：
- 首先我们应该使用v-model将整个表单绑定到一个数据对象上（注意，为了收集表单信息，我们已经将每一个input输入框绑定到了数据对象的相应属性，但是对于表单验证来说，这是不够的）
- 然后，我们在form-item组件上使用prop将form-item绑定到数据对象的相应属性，这样一来，表单就在**自己的表单元素**和**数据对象**之间建立了连接
- 紧接着我们应该向表单提供一个校验规则对象`rules`，表单将根据这个对象对数据进行校验


```vue
<el-form class="login_form" v-model="loginForm" :rules="rules">
	<el-form-item prop="username">
		<el-input :prefix-icon="User" v-model="loginForm.username"></el-input>
	</el-form-item>

	<el-form-item prop="password">
		<el-input type="password" :prefix-icon="Lock" v- model="loginForm.password" show-password></el-input>
	</el-form-item>
	<el-button :loading="loading" class="login_btn" type="primary" size="default" @click="login">登录</el-button>
</el-form>
```


经过以上准备工作，我们只需按照element的规定定义好rules对象，表单验证就可以工作了

首先我们使用element自带的简单验证方法，按照其提供的一些常用属性配置校验规则
required: 必填
min: 最小长度
max: 最大长度
message: 验证不通过时的提示信息
trigger: 触发事件，分为blur和change

```ts
const rules = {
    username: [
		{required: true, min:5, max:10, message: '请输入5~10位用户名', trigger: 'change'}
	],

    password: [
        {required: true, min:6, max:15, message: '请输入6~15位密码', trigger: 'change'}
    ]
}
```

配置好这个对象后，element就可以对表单进行简易的验证，并对用户抛出错误提示信息。

但是这样仅仅只能对用户产生提示，而不能对程序产生真正的控制，假设在输入不合格规范的情况下，用户仍然点击了登录按钮，那么程序还是会发出请求。

### 基于element规则的表单验证

我们需要在验证后获取结果，并让程序对登录业务产生实际的控制（也就是验证成功就发请求，验证不成功就拦截）。这里需要用到element的form表单组件提供的一个方法 `validate()`，它会返回一个成功或者失败的promise，我们只要在登录请求发出之前调用表单的这个方法进行校验，再做出相应的处理即可。但是首先，我们需要获取到表单实例，才能调用到这个方法。

对表单添加ref:
```vue
<el-form class="login_form" v-model="loginForm" :rules="rules" ref="loginFormInstance">
	<el-form-item prop="username">
		<el-input :prefix-icon="User" v-model="loginForm.username"></el-input>
	</el-form-item>

	<el-form-item prop="password">
		<el-input type="password" :prefix-icon="Lock" v- model="loginForm.password" show-password></el-input>
	</el-form-item>
	<el-button :loading="loading" class="login_btn" type="primary" size="default" @click="login">登录</el-button>
</el-form>
```

获取表单实例并在登录逻辑进行之前调用`validate`方法：
```ts
//获取表单元素
let loginFormInstance = ref();

const login = async() => {
    //表单验证通过后发请求
    //validate是el-form上自带的方法
    await loginFormInstance.value.validate();
    ......
    }
```

这样，我们就可以对表单进行简单的验证了。

但是这种简单的写法有很大的缺陷，那就是不够灵活，它仅仅只能对非空和长度进行验证，在大多数情况下，我们需要对用户输入的内容进行验证，比如验证身份证号，手机号，邮箱时，这些内容都具有一些特定的格式。

所以我们通常使用自定义校验的写法对表单内容进行验证。

考虑以下rules配置对象：
```ts
const rules = {

    username: [
        {trigger: 'change',validator: validateUsername}
    ],

    password: [
        {trigger: 'change',validator: validatePassword}
    ]

}
```

实际上，这种写法就是添加了一个校验器`validator`属性，它的值是一个用于表单校验的函数，把校验逻辑全部转移到了函数中。
这种函数接收三个参数：
rule:校验规则对象
value:表单元素的文本内容
callback:是一个放行函数，验证通过调用callback放行，不通过调用callback并注入错误信息
注意参数的数量和顺序必须正确

考虑以下校验函数：
```ts
const validateUsername = (rule:any, value:any, callback:any) => {
    console.log(rule)
    if(/^\d{5,10}$/.test(value)){
        callback();
    }else{
        callback(new Error('请输入5~10位用户名'));
    }
}

const validatePassword = (rule:any, value:any, callback:any) => {
    console.log(rule)
    if(/^\d{6,15}$/.test(value)){
        callback();
    }else{
        callback(new Error('请输入6~15位密码'));
    }
}
```

可以看到我们使用正则表达式对账号和密码的长度进行了验证，验证通过则调用callback()放行，不通过则抛出错误。

使用这种自定义校验函数的方法，我们就可以对表单进行灵活的校验。

### 更多

很多常见的表单业务规则会有许多复用场景，所以考虑对其进行封装

- 对校验函数进行封装

```js
/* 验证统一社会信用代码 */

export function isUSCI(rule, value, callback){

    usci(value) ? callback() : callback(new Error('请输入有效统一社会信用代码。'))

}

/* 验证IP地址 */

export function checkIp(rule, value, callback) {

    isIP(value) ? callback() : callback(new Error('请输入有效IP地址。'))

}

/* 验证身份证号 */

export function checkIdCard(rule, value, callback) {

    Validator.isValid(value) ? callback() : callback(new Error('请输入有效身份证号码。'))

}

/* 验证邮箱 */

export function checkEmail (rule, value, callback){

    isEmail(value) ? callback() : callback(new Error('请输入有效邮箱。'))

}

/* 验证邮政编码 */

export function checkZip (rule, value, callback){

    const reg = /^\d{6}$/;

    reg.test(value) ? callback() : callback(new Error('请输入有效邮政编码。'))

}

/* 手机验证 */

export function checkMobile (rule, value, callback){

    isMobilePhone(value, ['zh-CN', 'zh-HK']) ? callback() : callback(new Error('请输入有效手机号码。'))

}

/* 电话验证 */

export function checkTel (rule, value, callback){

    const reg = /^(\(\d{3,4}\)|\d{3,4}-|\s)?\d{7,14}$/;

    reg.test(value) ? callback() : callback(new Error('请输入有效电话号码。'))

}
```

- 对规则集进行封装

```js
required: (message, trigger, type) => {return {required: true, type: type, message: message, trigger: trigger || ['blur', 'change']}},

usci: (message, trigger) => {return {required: true, type: 'string', validator: isUSCI, message: message || '请输入有效统一社会信用代码。', trigger: trigger || ['blur', 'change']}},

idCard: (message, trigger, type) => {return {required: true, type: type || 'string', validator: checkIdCard, message: message || '请输入有效身份证号码。', trigger: trigger || ['blur', 'change']}},

email: (message, trigger, type) => {return [{required: true, type: type || 'string', message: message|| '请输入有效邮箱', trigger: trigger || ['blur', 'change']}, {validator: checkEmail, message: '请输入有效邮箱', trigger: ['blur', 'change']}]},

mobile: (message, trigger, type) => {return [{required: true, type: type || 'string', message: message|| '请输入有效手机号码', trigger: trigger || ['blur', 'change']}, {validator: checkMobile, message: '请输入有效手机号码', trigger: trigger || ['blur', 'change']}]},

tel: (message, trigger, type) => {return {required: true, validator: checkTel, message: '请输入有效电话号码', trigger: type || ['blur', 'change']}},

ip: (message, trigger, type) => {return {required: true, type: type || 'string', validator: checkIp, message: message|| '请输入有效IP地址', trigger: trigger || ['blur', 'change']}},

zip: (message, trigger, type) => {return {required: true, type: type || 'string', validator: checkZip, message: message|| '请输入有效邮政编码', trigger: trigger || ['blur', 'change']}}
```

使用实例：

```html
<x-form-item class="form-label" cols="1" label="作　　者:" prop="_AUTHOR" :rules="Rules.required('请输入作者姓名')">
	<el-input v-model="formData._AUTHOR" placeholder="请输入作者" />
</x-form-item>

<script>
import Rules from '...'
</script>
