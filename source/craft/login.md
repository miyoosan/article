
<!-- 进入登录界面
  -本地有凭证Token
    -发送到服务端验证Token
      -网络错误，提示错误信息
      -网络正常
        -Token有效，直接进入游戏
        -Token无效，清空本地Token，进行【账号密码登录】
  -没有凭证Token
    -输入账号密码，点击登录
      -服务端验证账号密码
        -网络错误，提示错误信息
        -网络正常
          -账号存在
            -密码正确，进入游戏|返回凭证，本地保存
            -密码错误，提示输入正确密码
          -账号不存在
            -密码不为空，进入游戏|创建账号，返回凭证，本地保存
            -密码为空，提示输入密码 -->

<!-- 
```flow
st=>start: 登录界面
token=>condition: 本地有凭证
Token吗？
network1=>condition: 网络正常？
network2=>condition: 网络正常？
op1=>operation: 发送Token到服务端
authorized=>condition: Token有效？
login=>subroutine: 使用账号密码登录
account=>condition: 账号存在？
password=>condition: 密码正确？
pwd=>condition: 密码不
为空？
user=>subroutine: 创建账号
update=>subroutine: 生成新Token
e=>end: 进入游戏
e1=>end: 提示网络错误信息
e2=>end: 提示网络错误信息
e3=>end: 重新登录
e4=>end: 提示输入密码
e5=>end: 提示输入正确密码

st->token
token(no)->login
token(yes)->authorized
authorized(no)->login
authorized(yes)->e
login->account
account(no)->pwd
pwd(no)->e4
pwd(yes)->user->update
account(yes)->password
password(no)->e5
password(yes)->update
update(left)->e
``` -->