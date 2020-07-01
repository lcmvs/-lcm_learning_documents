#  			 [Java通过BCrypt加密](https://www.cnblogs.com/xingzc/p/8624007.html)  		



## 一、概述

 

在用户模块，对于用户密码的保护，通常都会进行加密。我们通常对密码进行加密，然后存放在数据库中，在用户进行登录的时候，将其输入的密码进行加密然后与数据库中存放的密文进行比较，以验证用户密码是否正确。

目前，MD5和BCrypt比较流行。相对来说，BCrypt比MD5更安全，但加密更慢。

 

## 二、使用BCrypt

 

首先，可以在官网中取得源代码<http://www.mindrot.org/projects/jBCrypt/>

然后通过Ant进行编译。编译之后得到jbcrypt.jar。也可以不需要进行编译，而直接使用源码中的java文件（本身仅一个文件）。

下面是官网的一个Demo。

```java
public class BCryptDemo {
  public static void main(String[] args) {
 　　// Hash a password for the first time
 　　　　String password = "testpassword";
　　　　String hashed = BCrypt.hashpw(password, BCrypt.gensalt());
　　　　System.out.println(hashed);
　　// gensalt's log_rounds parameter determines the complexity
　　// the work factor is 2**log_rounds, and the default is 10
　　String hashed2 = BCrypt.hashpw(password, BCrypt.gensalt(12));
 
　　// Check that an unencrypted password matches one that has
　　// previously been hashed
　　String candidate = "testpassword";
　　//String candidate = "wrongtestpassword";
　　if (BCrypt.checkpw(candidate, hashed))
　　　　System.out.println("It matches");
　　else
　　System.out.println("It does not match");
　　}
}
```

 

在这个例子中，

```java
BCrypt.hashpw(password, BCrypt.gensalt())
```

是核心。通过调用BCrypt类的静态方法hashpw对password进行加密。第二个参数就是我们平时所说的加盐。

```java
BCrypt.checkpw(candidate, hashed)
```

该方法就是对用户后来输入的密码进行比较。如果能够匹配，返回true。

 

## 三、加盐

如果两个人或多个人的密码相同，加密后保存会得到相同的结果。破一个就可以破一片的密码。如果名为A的用户可以查看数据库，那么他可以观察到自己的密码和别人的密码加密后的结果都是一样，那么，别人用的和自己就是同一个密码，这样，就可以利用别人的身份登录了。

其实只要稍微混淆一下就能防范住了，这在加密术语中称为“加盐”。具体来说就是在原有材料（用户自定义密码）中加入其它成分（一般是用户自有且不变的因素），以此来增加系统复杂度。当这种盐和用户密码相结合后，再通过摘要处理，就能得到隐蔽性更强的摘要值。