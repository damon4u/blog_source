---
title: shiro开发（四）Realm及相关对象
date: 2018-11-21 16:41:50
tags: shiro
categories: shiro
---

### AuthenticationToken
AuthenticationToken用于收集用户提交的身份（如用户名）及凭据（如密码）：
```java
public interface AuthenticationToken extends Serializable {
    Object getPrincipal(); //身份
    Object getCredentials(); //凭据
}
```
扩展接口`RememberMeAuthenticationToken`提供了`boolean isRememberMe()`实现“记住我”的功能；
扩展接口`HostAuthenticationToken`提供了`String getHost()`方法用于获取用户“主机”的功能。

Shiro提供了一个直接拿来用的`UsernamePasswordToken`，用于实现用户名/密码`Token`组，另外其实现了`RememberMeAuthenticationToken`和`HostAuthenticationToken`，可以实现记住我及主机验证的支持。

<!-- more -->

我们可以实现自己的`AuthenticationToken`，例如想要统一账号登录，这个账号可能是平台唯一账号：
```java
public class MyAuthenticationToken implements AuthenticationToken {

	private String account; // 统一账号

	public MyAuthenticationToken(String account) {
		this.account = account;
	}

	@Override
	public Object getPrincipal() {
		return getAccount();
	}

	@Override
	public Object getCredentials() {
		return getAccount(); // 密码凭证也返回统一账号
	}

	public String getAccount() {
		return this.account;
	}

}
```

### AuthenticationInfo
`AuthenticationInfo`有两个作用：
1. 如果Realm是`AuthenticatingRealm`子类，则提供给`AuthenticatingRealm`内部使用的`CredentialsMatcher`进行凭据验证；
2. 提供给SecurityManager来创建Subject（提供身份信息）。

说一下第一个功能。
一般我们在实现Realm时，会实现`doGetAuthenticationInfo`方法，获取认证信息：
```java
protected abstract AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException;
```
然后`AuthenticatingRealm`内部会使用这个认证信息进行凭证校验：
```java
protected void assertCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) throws AuthenticationException {
    CredentialsMatcher cm = getCredentialsMatcher();
    if (cm != null) {
        if (!cm.doCredentialsMatch(token, info)) {
            //not successful - throw an exception to indicate this:
            String msg = "Submitted credentials for token [" + token + "] did not match the expected credentials.";
            throw new IncorrectCredentialsException(msg);
        }
    } else {
        throw new AuthenticationException("A CredentialsMatcher must be configured in order to verify " +
                "credentials during authentication.  If you do not wish for credentials to be examined, you " +
                "can configure an " + AllowAllCredentialsMatcher.class.getName() + " instance.");
    }
}
```
我们可以实现自己的`AuthenticationInfo`：
```java
public class MyAuthenticationInfo implements AuthenticationInfo {

	private PrincipalCollection principals;
	private String credentials;

	public MyAuthenticationInfo(Object principal, String realmName) {
		this.principals = new SimplePrincipalCollection(principal, credentials);
		this.credentials = credentials;
	}

	@Override
	public PrincipalCollection getPrincipals() {
		return this.principals;
	}

	@Override
	public Object getCredentials() {
		return this.credentials;
	}

}
```
其中`PrincipalCollection`用于聚合这些身份信息，一般用`SimplePrincipalCollection`就好。

当然，还要写一个对应的比对`CredentialsMatcher`，用来对`MyAuthenticationInfo`进行凭证校验：
```java
public class MintCredentialsMatcher extends PasswordMatcher {

	@Override
	public boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) {
		if(token instanceof MyAuthenticationToken) {
			return token.getCredentials().equals(info.getCredentials());
		}
		return super.doCredentialsMatch(token, info);
	}
}
```
在MyRealm的`doGetAuthenticationInfo`方法中，返回一个`MyAuthenticationInfo`，将`MyAuthenticationToken`的account赋值给`MyAuthenticationInfo`的credentials：
```java
return new MyAuthenticationInfo(user, myAuthenticationToken.getAccount());
```
这样在校验时就能成功了。


### AuthorizationInfo
AuthorizationInfo包含了授权信息，当我们使用`AuthorizingRealm`时，如果身份验证成功，在进行授权时就通过`doGetAuthorizationInfo`方法获取角色/权限信息用于授权验证。
Shiro提供了一个实现`SimpleAuthorizationInfo`，大多数时候使用这个即可。

### Subject
Subject是Shiro的核心对象，基本所有身份验证、授权都是通过Subject完成。
#### 身份信息获取
```java
Object getPrincipal(); //Primary Principal
PrincipalCollection getPrincipals(); // PrincipalCollection
```
#### 身份验证
```java
void login(AuthenticationToken token) throws AuthenticationException;
boolean isAuthenticated();
boolean isRemembered();
```
通过`login`登录，如果登录失败将抛出相应的`AuthenticationException`，如果登录成功调用`isAuthenticated`就会返回true，即已经通过身份验证；
如果`isRemembered`返回true，表示是通过记住我功能登录的而不是调用login方法登录的。
`isAuthenticated`/`isRemembered`是互斥的，即如果其中一个返回true，另一个返回false。
#### 角色授权验证
```java
boolean hasRole(String roleIdentifier);
boolean[] hasRoles(List<String> roleIdentifiers);
boolean hasAllRoles(Collection<String> roleIdentifiers);
void checkRole(String roleIdentifier) throws AuthorizationException;
void checkRoles(Collection<String> roleIdentifiers) throws AuthorizationException;
void checkRoles(String... roleIdentifiers) throws AuthorizationException;
```
`hasRole*`进行角色验证，验证后返回true/false；而`checkRole*`验证失败时抛出`AuthorizationException`异常。
#### 权限授权验证
```java
boolean isPermitted(String permission);
boolean isPermitted(Permission permission);
boolean[] isPermitted(String... permissions);
boolean[] isPermitted(List<Permission> permissions);
boolean isPermittedAll(String... permissions);
boolean isPermittedAll(Collection<Permission> permissions);
void checkPermission(String permission) throws AuthorizationException;
void checkPermission(Permission permission) throws AuthorizationException;
void checkPermissions(String... permissions) throws AuthorizationException;
void checkPermissions(Collection<Permission> permissions) throws AuthorizationException;
```
`isPermitted*`进行权限验证，验证后返回true/false；而`checkPermission*`验证失败时抛出`AuthorizationException`。
#### 会话
```java
Session getSession(); //相当于getSession(true)
Session getSession(boolean create);  
```
类似于Web中的会话。如果登录成功就相当于建立了会话，接着可以使用getSession获取；如果create=false如果没有会话将返回null，而create=true如果没有会话会强制创建一个。
#### 退出
```
void logout();
```

参考：
[第六章 Realm及相关对象——《跟我学Shiro》](http://jinnianshilongnian.iteye.com/blog/2022468)
[Apache Shiro Reference Documentation](http://shiro.apache.org/reference.html)
