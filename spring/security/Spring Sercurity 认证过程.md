## Spring Sercurity 认证过程

### 认证时序图



![](../../images/security/2.png)

1. attemptAuthentication。

   ```java
   //1.判断请求方式，只能是post请求
   if (this.postOnly && !request.getMethod().equals("POST")) {
   			throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
   		}
   //2.构造UsernamePasswordAuthenticationToken对象
   		String username = obtainUsername(request);
   		username = (username != null) ? username : "";
   		username = username.trim();
   		String password = obtainPassword(request);
   		password = (password != null) ? password : "";
   		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);
   //3.调用authenticate方法
   this.getAuthenticationManager().authenticate(authRequest);
   ```

   

   



