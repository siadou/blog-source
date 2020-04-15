---
title: 如何去掉浏览器的自动填充
date: 2020-04-02 15:02:43
tags: [前端技术, 浏览器]
---
# 问题

展示和登录页复用的表单，在展示页不希望自动填充密码，想使用autocomplete=off关闭自动填充，调试的时候无效。查询了相关的支持情况：

# 浏览器支持情况
Can I Use上描述的对autocomplete on / off值的支持：

浏览器 | 支持情况
---|---
IE 6-10 | 完全支持
Chrome 27-40, IE 11 | password字段不支持off设置
Chrome 41 ~ | 当用户开启自动填充时忽略off设置
Firefox | login form忽略off设置
Safari | username, email, password 会忽略 off
移动端浏览器 | on的时候不会显示上一次提交的值

# 解决方法
1. 在name上添加随机字符串
2. 将字段设置为只读，当触发focus事件的时候去掉只读属性

```
<input id="email" readonly type="email" onfocus="if (this.hasAttribute('readonly')) {
    this.removeAttribute('readonly');
    // fix for mobile safari to show virtual keyboard
    this.blur();    this.focus();  }" />
```

3. chrome的解决办法：
```
<form name="myForm"" method="post">
   <input name="user" type="text" />
   <input name="pass" type="password" autocomplete="new-password" />
   <input type="submit">
</form>
```
# 附 autocomplete chrome 语义

- username:用于记住用户名的场合
- current-password:用于记住密码的场合
- new-password:用于修改密码的场合

# 参考文献
- [Password Form Styles that Chromium Understands](https://www.chromium.org/developers/design-documents/form-styles-that-chromium-understands)
- [Can I Use](https://caniuse.com/#search=autocomplete)
- [Stack Overflow](https://stackoverflow.com/questions/2530/how-do-you-disable-browser-autocomplete-on-web-form-field-input-tag)
