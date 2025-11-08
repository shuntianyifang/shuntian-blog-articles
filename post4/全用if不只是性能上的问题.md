---
title: 全用if不只是性能问题
published: 2025-11-10
description: '一个因为全用if导致的WA'
image: ''
tags: [Programming Pitfalls]
category: Programming Pitfalls
draft: false
---

## if与if-else的差异  

### 核心差异  

|特性|只使用if|if-else|
|--------|--------|--------|
|执行逻辑|所有条件都会检查|遇到第一个真条件后停止检查|
|互斥性|多个条件可能同时执行|最多执行一个分支|
|性能|可能较低（检查所有条件）|通常较高（短路执行）|
|逻辑清晰度|适用于独立条件|适用于互斥条件|

### 具体示例  

#### 只使用if

这是一段注册功能的java代码，每个if所对应的检查都有明确的目的和错误处理

```java
    /**
     * 用户注册功能
     * @param username 用户名
     * @param password 密码
     * @param email 邮箱
     * @param userType 用户类型
     * @return 注册成功返回用户信息，用户名已存在返回null
     */
    @Override
    public Map<String, Object> register(String username, String nickname, String password, String email, UserTypeEnum userType) {

        LambdaQueryWrapper<User> userQueryWrapper = new LambdaQueryWrapper<>();
        userQueryWrapper.eq(User::getUsername, username);
        User existingUser = userMapper.selectOne(userQueryWrapper);

        if (existingUser != null) {
            throw new ApiException(ExceptionEnum.USER_EXIST);
        }

        // 检查邮箱是否已存在
        if (email != null && !email.isEmpty()) {
            LambdaQueryWrapper<User> emailQueryWrapper = new LambdaQueryWrapper<>();
            emailQueryWrapper.eq(User::getEmail, email);
            User emailUser = userMapper.selectOne(emailQueryWrapper);
            if (emailUser != null) {
                throw new ApiException(ExceptionEnum.EMAIL_EXIST);
            }
        }

        // 创建新用户
        User user = User.builder()
                .username(username)
                .nickname(nickname)
                .password(passwordEncoder.encode(password))
                .email(email)
                .userType(userType != null ? userType : UserTypeEnum.STUDENT)
                .deleted(false)
                .createdAt(LocalDateTime.now())
                .updatedAt(LocalDateTime.now())
                .lastLoginAt(LocalDateTime.now())
                .build();

        userMapper.insert(user);

        UserSimpleVO userVO = userConverterUtils.toUserSimpleVO(user);

        // 生成JWT token
        UserDetails userDetails = userDetailsService.loadUserByUsername(username);
        String token = jwtTokenUtil.generateToken(userDetails);

        // 返回结果
        Map<String, Object> result = new HashMap<>(8);
        result.put("token", token);
        result.put("user", userVO);
        result.put("message", "注册成功");

        return result;
    }
```  

检查用户名和检查邮箱是两个独立的过程，即使用户名已存在，我们可能仍然想知道邮箱是否也被占用（或者反之），因此使用独立的if是合理的

#### if-else  

一个签到题的c++题解
原题是一道判定n变为3的倍数所需增加多少量的题

```c++
    #include <iostream>
    using namespace std;
    int main(){
        int t;
        cin >> t;
        for (int i = 0; i < t; i++){
            int n;
            cin >> n;
            if (n % 3 == 0){
                cout << 0 << endl;
            }
            else{
                cout << 3 - (n % 3) << endl;
            }
        }
        return 0;
    }
```

n除以3不余0的情况自然便是else  
不过此题全用if也不会WA，顶多慢一点  

## 我是怎么WA的  

原题（洛谷上的）：[P1563 [NOIP 2016 提高组] 玩具谜题](https://www.luogu.com.cn/problem/P1563)

预期逻辑：对于每条指令，根据当前小人x的朝向(p[x]) 和指令方向(a[i])，二选一地决定是顺时针还是逆时针移动  

WA的代码：

```c++
    #include <bits/stdc++.h>
    #define ll long long
    using namespace std;
    ll p[100010] = {0};
    string name[100010] = {};
    ll a[100010] = {0};
    ll s[100010] = {0};
    int main (){
        ll n,m;
        cin >> n >> m;
        for (ll i = 0; i < n; i++){
            cin >> p[i] >> name[i];
        }
        for (ll i = 0; i < m; i++){
            cin >> a[i] >> s[i];
        }
        int x = 0;
        for (ll i = 0; i < m; i++){
            if ((p[x] == 0 && a[i] == 1) || (p[x] == 1 && a[i] == 0)){
                x = (x + s[i]) % n;
            }
            if ((p[x] == 1 && a[i] == 1) || (p[x] == 0 && a[i] == 0)){
                x = (x - s[i] % n + n) % n;
            }
        }
        cout << name[x];
        return 0;
    }
```  

AC的代码:

```c++
    #include <bits/stdc++.h>
    #define ll long long
    using namespace std;
    ll p[100010] = {0};
    string name[100010] = {};
    ll a[100010] = {0};
    ll s[100010] = {0};
    int main (){
        ll n,m;
        cin >> n >> m;
        for (ll i = 0; i < n; i++){
            cin >> p[i] >> name[i];
        }
        for (ll i = 0; i < m; i++){
            cin >> a[i] >> s[i];
        }
        int x = 0;
        for (ll i = 0; i < m; i++){
            if ((p[x] == 0 && a[i] == 1) || (p[x] == 1 && a[i] == 0)){
                x = (x + s[i]) % n;
            }
            else {
                x = (x - s[i] % n + n) % n;
            }
        }
        cout << name[x];
        return 0;
    }
```  

可以看到，两者间只把if改成else，差距却一个WA一个AC  
乍看上去两者相差不大，都正确包含了所需情况  
然而只使用if会导致第二个if在意料之外被执行  

由于每个if语句执行后x会变，而x又是条件语句中的一部分  
于是就出现了这样的情况  
在第一个if更新当前位置x后，第二个if使用更新后的x来判断朝向  
这与我们程序的目的显然不符  

正确的做法是使用if-else结构，确保每条指令都基于指令执行前的位置x的朝向来决定移动方向  
