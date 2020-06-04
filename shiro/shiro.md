## 前言

本文旨在介绍shiro源码，使用的shiro版本是1.4.0，使用springboot整合shiro的方式

需掌握的内容：

1. tomcat和java中servlet和filter相关的知识，本文只会涉及到一些作用和标准，filter的实现若要细究请看tomcat源码

## 实现基础类

1. **Filter**，拦截器，第一道工序，把请求接进来
2. **Authentication**，认证器，也就是认证登陆，通过调用**Realm**实现
3. **Authorization**，授权器，也就是校验权限，这个接口有，被**AuthorizationRealm**实现了
4. **Subject**，用户主体，内部封装自定义用户类，
5. **Realm**，领域内认证授权处理器，真正实现认证授权的大佬，通过support与token匹配
6. **SecurityManager**，管理器，组装和合成其他部件的配合工作
7. **AuthorizationAttributeSourceAdvisor**,注解AOP拦截器
8. **AbstractCacheManager**，缓存管理器，缓存token-AuthenticationInfo等信息，shiro提供的实现是MemoryConstrainedCacheManager，使用本地的MapCache缓存，可以自己实现基于redis等任意存储形式的缓存
9. **ThreadContext**，Subject本地JVM缓存，ThreadLocal实现，目的是在任意位置使用ThreadContext.getSubject()能获取到用户，与**SecurityManager**一对一绑定
10. **Token**，请求中附带的用户信息，由filter.createToken()方法提供封装对象，token的重点在于附带的信息和Realm的support与token匹配

## 原理

1. shiro如何实现拦截？

   底层通过java拦截器实现，shiro自定义了很多抽象类和可以直接使用的拦截器，能够实现各种功能，详见拦截器章节

2. 登录之后的请求，shiro之后怎么存储用户信息？（如何知道该用户已登录）

   1. 通过请求上下文创建用户主体，绑定到线程上下文
   2. shiro鉴权拦截器非允许通过的，会走onAccessDenied逻辑，此处调用登录，完成subject.login
   3. shiro登录时，会尝试拿缓存的验证和鉴权信息，之后会set到当前subject中

3. 如何实现token的自定义实现？

   1. 实现自定义Filter，createToken方法
   2. 实现自定义Token，实现org.apache.shiro.authc.AuthenticationToken

4. 授权逻辑

   TODO



## 自定义拦截器

拦截器种类，一个开发自定义的ShiroFIlter继承关系如下图：

![image-20200602112614521](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200602112614521.png)

介绍一下这些拦截器

1. javax.servlet.Filter，tomcat提供的java servlet filter接口，用于配置到web.xml实现拦截功能，主要实现doFilter()

2. org.apache.shiro.web.servlet.AbstractFilter，shiro封装的第一层Filter接口

   1. 持有了一个javax.servlet.FilterConfig类型的filterConfig对象，该对象有Filter名称等信息，并在init()方法中设置进来，具体调用init方法的应该是org.apache.shiro.web.filter.mgt.DefaultFilterChainManager#initFilter，而filterConfig的终极来源没有细究，应该是读取Filter配置（web.xml）的时候就可以知道
   2. 继承了org.apache.shiro.web.servlet.ServletContextSupport类，该类实现了对web上下文的获取

3. org.apache.shiro.web.servlet.NameableFilter，可命名Filter

   1. 持有一个name，该name可以是通过setName()方法设置的
   2. getName()方法中，name对象为空，则获取filterConfig对象获取web配置中的拦截器名称、

4. org.apache.shiro.web.servlet.OncePerRequestFilter，保证一个request只会执行一次的filter

   1. 持有了一个布尔值enabled，是否启用开关
   2. 实现了doFilter()方法，判断请求中是否存在已执行标识，以attribute形式存储，若存在则跳过该Filter，否则调用doFilterInternal方法，并在执行完成后移除已执行标识
   3. 定义doFilterInternal()抽象方法，子类拦截器的处理逻辑放在该方法中

5. org.apache.shiro.web.servlet.AdviceFilter，实现了AOP的拦截器

   1. 定义了三个抽象方法，子类可重写该方法，达到AOP处理的效果，入参都带有请求头和响应头

      ```java
      /**
       * 前置处理
       */
      protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
      	return true;
      }
      /**
       * 后置处理
       */
      protected void postHandle(ServletRequest request, ServletResponse response) throws Exception {
      }
      /**
       * 处理完成
       * 即使抛异常，也能执行，同时还能获取到异常
       */
      public void afterCompletion(ServletRequest request, ServletResponse response, Exception exception) throws Exception {
      }
      ```

   2. executeChain方法，调用Filter链中下一个Filter

      ```java
      protected void executeChain(ServletRequest request, ServletResponse response, FilterChain chain) throws Exception {
          chain.doFilter(request, response);
      }
      ```

   3. 实现了doFilterInternal方法，即OncePerRequestFilter中doFilter调用的，在该方法中，又将处理分为几部分

      preHandle -> executeChain -> postHandle -> afterCompletion

      继承该Filter的子类，则需要将处理逻辑放在上述三个方法中

      ```java
      public void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException {
      
      	Exception exception = null;
      
      	try {
      
      		boolean continueChain = preHandle(request, response);
      		if (log.isTraceEnabled()) {
      			log.trace("Invoked preHandle method.  Continuing chain?: [" + continueChain + "]");
      		}
      
      		if (continueChain) {
      			executeChain(request, response, chain);
      		}
      
      		postHandle(request, response);
      		if (log.isTraceEnabled()) {
      			log.trace("Successfully invoked postHandle method");
      		}
      
      	} catch (Exception e) {
      		exception = e;
      	} finally {
              //afterCompletion在此处调用，如果有异常，还会抛出ServletException或IOException，如果是其他异常，则包装成ServletException
      		cleanup(request, response, exception);
      	}
      }
      ```

6. org.apache.shiro.web.filter.PathMatchingFilter，可通过Url路径匹配的Filter

   1. 实现org.apache.shiro.web.filter.PathConfigProcessor，实现processPathConfig方法，该方法解析PathConfig，将需处理的path（URL前缀）以及对应的value放到appliedPaths里
   2. 实现preHandle方法，判断URL是否需要进行拦截，如果需要并且拦截器可用，调用onPreHandle方法，因此子类的拦截实现需要放在onPreHandle里

7. org.apache.shiro.web.filter.AccessControlFilter，访问控制过滤器，若用户未登录，将跳转到登录页

   1. 提供loginUrl对象，用于存放登录页面URL

   2. 定义两个抽象方法，isAccessAllowed用于判断是否运行通行，用登录的例子看就是判断是否登录；onAccessDenied则为不允许通行时的逻辑

      ```java
      protected abstract boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception;
      protected abstract boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception;
      ```

   3. 实现onPrehandle方法

      ```java
      public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
          return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
      }
      ```

   4. 实现了几个方法，提供给子类调用

      1. isLoginRequest，判断是否登录请求
      2. saveRequestAndRedirectToLogin，保存请求信息并重定向
      3. getSubject，获取登录实体

8. org.apache.shiro.web.filter.authc.AuthenticationFilter，需要登录认证的Filter

   该类的注释，Base class for all Filters that require the current user to be authenticated.

   1. 实现了isAccessAllowed方法，通过getSubject方法获取当前登录主体，并且将用户是否登录的判断交给Subject

      ```java
      protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
          Subject subject = getSubject(request, response);
          return subject.isAuthenticated();
      }
      ```

   2. 实现issueSuccessRedirect方法和持有successUrl字段，实现重定向到登录成功页方法

9. org.apache.shiro.web.filter.authc.AuthenticatingFilter，注释为可用的鉴权Filter

   1. 重写了isAccessAllowed方法，调用父类的isAccessAllowed，校验是否登录；增加了对非登录url并且mappedValue包含放行标识的请求的放行

      ```java
      @Override
      protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
          return super.isAccessAllowed(request, response, mappedValue) ||
                  (!isLoginRequest(request, response) && isPermissive(mappedValue));
      }
      ```

   2. 重写了cleanup方法，如果抛出UnauthenticatedException异常（未认证异常），执行onAccessDenied处理，并且不算异常，此处算是对OncePerRequestFilter拦截器的补充

      ```java
      @Override
      protected void cleanup(ServletRequest request, ServletResponse response, Exception existing) throws ServletException, IOException {
          if (existing instanceof UnauthenticatedException || (existing instanceof ServletException && existing.getCause() instanceof UnauthenticatedException))
          {
              try {
                  onAccessDenied(request, response);
                  existing = null;
              } catch (Exception e) {
                  existing = e;
              }
          }
          super.cleanup(request, response, existing);
      
      }
      ```

   3. 定义createToken抽象方法，为子类提供创建token的标准

      ```java
      protected abstract AuthenticationToken createToken(ServletRequest request, ServletResponse response) throws Exception;
      ```

      并且提供了Shiro默认的Token创建逻辑，org.apache.shiro.authc.UsernamePasswordToken

   4. 抽象了executeLogin方法，该方法实现登录，并且根据登录成功与否执行登录成功方法或登录失败方法，使用者可通过重写这两个方法达到登录后重定向，登录失败报错提示等

      ```java
      protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
          AuthenticationToken token = createToken(request, response);
          if (token == null) {
              String msg = "createToken method implementation returned null. A valid non-null AuthenticationToken " +
                      "must be created in order to execute a login attempt.";
              throw new IllegalStateException(msg);
          }
          try {
              Subject subject = getSubject(request, response);
              subject.login(token);
              return onLoginSuccess(token, subject, request, response);
          } catch (AuthenticationException e) {
              return onLoginFailure(token, e, request, response);
          }
      }
      /**
       * 登录成功，默认返回true，该Filter结束
       */
      protected boolean onLoginSuccess(AuthenticationToken token, Subject subject,
      	ServletRequest request, ServletResponse response) throws Exception {
      	return true;
      }
      
      /**
       * 登录失败，默认返回false，该Filter未通过
       */
      protected boolean onLoginFailure(AuthenticationToken token, AuthenticationException e,
      	ServletRequest request, ServletResponse response) {
      	return false;
      }
      ```

      通常该方法会放在onAccessDenied里执行，参考shiro的一个直接可用Filter，org.apache.shiro.web.filter.authc.FormAuthenticationFilter

      ```java
      protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
          if (isLoginRequest(request, response)) {
              if (isLoginSubmission(request, response)) {
                  if (log.isTraceEnabled()) {
                      log.trace("Login submission detected.  Attempting to execute login.");
                  }
                  return executeLogin(request, response);
              } else {
                  if (log.isTraceEnabled()) {
                      log.trace("Login page view.");
                  }
                  //allow them to see the login page ;)
                  return true;
              }
          } else {
              if (log.isTraceEnabled()) {
                  log.trace("Attempting to access a path which requires authentication.  Forwarding to the " +
                          "Authentication url [" + getLoginUrl() + "]");
              }
      
              saveRequestAndRedirectToLogin(request, response);
              return false;
          }
      }
      ```

      因为AuthenticatingFilter的isAccessAllowed中的逻辑，未对登录URL放行，所以会走onAccessDenied方法，因此在此判断如果是登录方法，则调用executeLogin()方法进行登录。

      ps：此处我觉得有点奇怪，为什么不是对登录URL进行放行？在网上搜了下，用Shiro实现登录的方式有两种，一种是直接通过Filter，即上述的executeLogin()方法进行登录，登录后重定向至登录接口，此时即跳过了Filter链，登录接口中再自实现登录校验；第二种是，自定义Filter，重写isAccessAllowed，对登录接口放行，在登录接口中，需自己通过调用subject.login()方法进行shiro的登录。第一种方法的好处则是只需简单配置，无需了解过多的shiro知识，即可使用shiro已经写好的form表单登录格式，但太过死板，不利于拓展；第二种则是需要自己对shiro有一定的了解，同时executeLogin方法也成了一个shiro登录方法的一个模板代码。

10. 自定义拦截器

    开发自定义拦截器时，要根据自身需求去选择继承shiro哪个抽象类。我使用shiro的项目中，只是需要自定义token，因此需要自定义Realm和Filter，而鉴权验证的逻辑是一样的，所以直接继承AuthenticatingFilter，继承该类，会发现需要重写两个方法

    ![image-20200602161542016](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200602161542016.png)

    1. createToken方法，自定义token的生成
    2. onAccessDenied方法，实现访问拒绝的逻辑

    我们在开发中，还重写了isAccessAllowed和onLoginFailure方法，对登录url进行了放行，封装了登录失败返回的格式

#### 小结

作者分析Filter的时候，是通过从最底层Filter开始一层一层往下延伸，此种方式是因为我已经知道拦截的原理大概是依赖底层的拦截器，而自定义Fitler的继承类图也刚好验证了这一点

## 获取当前用户

#### 绑定到线程上下文

调用的是org.apache.shiro.SecurityUtils#getSubject方法，接下来的源码分析将在代码注释中体现

```java
//org.apache.shiro.SecurityUtils#getSubject
public static Subject getSubject() {
    //从线程上下文中获取
	Subject subject = ThreadContext.getSubject();
	if (subject == null) {
        //否则新生成一个用户主体，并且绑定到线程上下文
		subject = (new Subject.Builder()).buildSubject();
		ThreadContext.bind(subject);
	}
	return subject;
}

//org.apache.shiro.util.ThreadContext#getSubject，先看看线程上下文的实现
public static Subject getSubject() {
	return (Subject) get(SUBJECT_KEY);
}

//org.apache.shiro.util.ThreadContext#get
public static Object get(Object key) {
	if (log.isTraceEnabled()) {
		String msg = "get() - in thread [" + Thread.currentThread().getName() + "]";
		log.trace(msg);
	}

	Object value = getValue(key);
	if ((value != null) && log.isTraceEnabled()) {
		String msg = "Retrieved value of type [" + value.getClass().getName() + "] for key [" +
		key + "] " + "bound to thread [" + Thread.currentThread().getName() + "]";
		log.trace(msg);
	}
	return value;
}

//org.apache.shiro.util.ThreadContext#getValue
private static Object getValue(Object key) {
	Map<Object, Object> perThreadResources = resources.get();
	return perThreadResources != null ? perThreadResources.get(key) : null;
}

//org.apache.shiro.util.ThreadContext#resources，resources对象定义
private static final ThreadLocal<Map<Object, Object>> resources = new InheritableThreadLocalMap<Map<Object, Object>>();
public static final String SUBJECT_KEY = ThreadContext.class.getName() + "_SUBJECT_KEY";
```

由此可以看出，线程上下文其实是通过特定的Key值去存储在ThreadLocal的Map对象里的，那这个对象到底是什么时候set进去的呢？我们看一下ThreadContext的源码，发现声明了put方法对resources进行put操作，在此处打个断点，并重新调用登录

ps：为什么不在在新生成用户对象并且绑定的地方打断点？亲测无效，说明不是在那地方set值得，先往下看

![image-20200602180242077](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200602180242077.png)

![image-20200602185453688](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200602185453688.png)

可以通过debug的调用栈看到，是在org.apache.shiro.subject.support.SubjectThreadState#bind的方法里绑定的

```java
/**
 * Binds a {@link Subject} and {@link org.apache.shiro.mgt.SecurityManager SecurityManager} to the
 * {@link ThreadContext} so they can be retrieved later by any
 * {@code SecurityUtils.}{@link org.apache.shiro.SecurityUtils#getSubject() getSubject()} calls that might occur
 * during the thread's execution.
 * <p/>
 * Prior to binding, the {@code ThreadContext}'s existing {@link ThreadContext#getResources() resources} are
 * retained so they can be restored later via the {@link #restore restore} call.
 * 通过注释可以看到，的确是在该方法将Subject和SecurityManager绑定到线程上下文中
 */
public void bind() {
    SecurityManager securityManager = this.securityManager;
    if ( securityManager == null ) {
        //try just in case the constructor didn't find one at the time:
        securityManager = ThreadContext.getSecurityManager();
    }
    this.originalResources = ThreadContext.getResources();
    ThreadContext.remove();

    ThreadContext.bind(this.subject);
    if (securityManager != null) {
        ThreadContext.bind(securityManager);
    }
}
```

继续往上看

```java
//org.apache.shiro.subject.support.SubjectCallable#call
public V call() throws Exception {
	try {
		threadState.bind();
		return doCall(this.callable);
	} finally {
		threadState.restore();
	}
}

//org.apache.shiro.subject.support.DelegatingSubject#execute(java.util.concurrent.Callable<V>)
public <V> V execute(Callable<V> callable) throws ExecutionException {
	Callable<V> associated = associateWith(callable);
	try {
		return associated.call();
	} catch (Throwable t) {
		throw new ExecutionException(t);
	}
}

//org.apache.shiro.web.servlet.AbstractShiroFilter#doFilterInternal
protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)
throws ServletException, IOException {
	// ...省略部分代码，可以看到是在这个类中调用了subject.execute
    	final Subject subject = createSubject(request, response);
		subject.execute(new Callable() {
			public Object call() throws Exception {
				updateSessionLastAccessTime(request, response);
				executeChain(request, response, chain);
				return null;
			}
		});
	}
	// ...
}
```

SubjectCallable继承java.util.concurrent.Callable，该类持有一个Callable对象和ThreadState对象，在call的时候，会先调用`threadState.bind()`，再调用callable的call方法，所以通过该类注册的任务，都会先经过绑定。

org.apache.shiro.subject.support.DelegatingSubject是Subject的一个实现类，而org.apache.shiro.web.servlet.AbstractShiroFilter是继承了OncePerRequestFilter的一个拦截器，是org.apache.shiro.web.servlet.ShiroFilter的父类，是shiro很重要的一个类，下面我们再具体分析这个类。

#### Subject的创建

```java
//org.apache.shiro.web.servlet.AbstractShiroFilter#createSubject
protected WebSubject createSubject(ServletRequest request, ServletResponse response) {
    //实例化了一个WebSubject的Builder，并赋值securityManager和请求头响应头，然后调用buildWebSubject
    return new WebSubject.Builder(getSecurityManager(), request, response).buildWebSubject();
}
//org.apache.shiro.web.subject.WebSubject.Builder#buildWebSubject
public WebSubject buildWebSubject() {
    //主要还是通过父类org.apache.shiro.subject.Subject.Builder#buildSubject方法实例subject，增加了类型检测
    Subject subject = super.buildSubject();
    if (!(subject instanceof WebSubject)) {
        String msg = "Subject implementation returned from the SecurityManager was not a " +
                WebSubject.class.getName() + " implementation.  Please ensure a Web-enabled SecurityManager " +
                "has been configured and made available to this builder.";
        throw new IllegalStateException(msg);
    }
    return (WebSubject) subject;
}
//org.apache.shiro.subject.Subject.Builder#buildSubject，委托给securityManager创建subject
public Subject buildSubject() {
    return this.securityManager.createSubject(this.subjectContext);
}
```

主要的创建逻辑还是在org.apache.shiro.mgt.DefaultSecurityManager#createSubject(org.apache.shiro.subject.SubjectContext)

```java
public Subject createSubject(SubjectContext subjectContext) {
    //create a copy so we don't modify the argument's backing map:
    SubjectContext context = copy(subjectContext);

    //确保设置SecurityManager 顺序为1、当前subject上下文 2、线程上下文 3、全局
    context = ensureSecurityManager(context);

    //解析session信息
    context = resolveSession(context);

    //解析权限信息，可适用缓存和rememberMe
    context = resolvePrincipals(context);
    
    //创建subject
    Subject subject = doCreateSubject(context);

    //save this subject for future reference if necessary:
    //(this is needed here in case rememberMe principals were resolved and they need to be stored in the
    //session, so we don't constantly rehydrate the rememberMe PrincipalCollection on every operation).
    //Added in 1.2:
    save(subject);

    return subject;
}
//实际是调用subjectFactory创建
protected Subject doCreateSubject(SubjectContext context) {
	return getSubjectFactory().createSubject(context);
}
//保存subject信息
protected void save(Subject subject) {
	this.subjectDAO.save(subject);
}
```

```java
//org.apache.shiro.web.mgt.DefaultWebSubjectFactory#createSubject
public Subject createSubject(SubjectContext context) {
    if (!(context instanceof WebSubjectContext)) {
        return super.createSubject(context);
    }
    WebSubjectContext wsc = (WebSubjectContext) context;
    SecurityManager securityManager = wsc.resolveSecurityManager();
    Session session = wsc.resolveSession();
    boolean sessionEnabled = wsc.isSessionCreationEnabled();
    PrincipalCollection principals = wsc.resolvePrincipals();
    boolean authenticated = wsc.resolveAuthenticated();
    String host = wsc.resolveHost();
    ServletRequest request = wsc.resolveServletRequest();
    ServletResponse response = wsc.resolveServletResponse();
    //实例化一个WebDelegatingSubject
    return new WebDelegatingSubject(principals, authenticated, host, session, sessionEnabled,
            request, response, securityManager);
}
```

可以看到subject是通过请求信息和上下文的信息创建的，并且如果session不为空或者sessionStorageEnabled为真时，会存储到session里。

综上所述，用户主体是通过请求信息和上下文创建后，绑定到线程上下文中，为了确保所有的Filter都能获得该信息，所以AbstractShiroFilter会放在最早执行，详见AbstractShiroFilter章节。

## AbstractShiroFilter

1. AbstractShiroFilter是Shiro的入口，是Shiro执行过程中的基类

2. AbstractShiroFilter另外个作用就是创建Subject实例，并绑定到当前线程ThreadLocal变量中

3. AbstractShiroFilter最后一个作用就是，根据URL配置的filter，选择并执行相应的filter chain、

   这个Filter继承了OncePerRequestFilter，所以需要实现doFilterInternal方法，我们来看看这个方法

   ```java
   protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)
           throws ServletException, IOException {
       Throwable t = null;
       try {
           final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
           final ServletResponse response = prepareServletResponse(request, servletResponse, chain);
           final Subject subject = createSubject(request, response);
   
           //noinspection unchecked
           subject.execute(new Callable() {
               public Object call() throws Exception {
                   //更新非HTTP session的lastAccessTime
                   updateSessionLastAccessTime(request, response);
                   //调用下一个Filter
                   executeChain(request, response, chain);
                   return null;
               }
           });
       } catch (ExecutionException ex) {
           t = ex.getCause();
       } catch (Throwable throwable) {
           t = throwable;
       }
   
       if (t != null) {
           if (t instanceof ServletException) {
               throw (ServletException) t;
           }
           if (t instanceof IOException) {
               throw (IOException) t;
           }
           //otherwise it's not one of the two exceptions expected by the filter method signature - wrap it in one:
           String msg = "Filtered request failed.";
           throw new ServletException(msg, t);
       }
   }
   ```

   这里看到，除了上文说过的调用subject.execute，绑定当前线程用户主体之外，在调用完成后，还会执行updateSessionLastAccessTime和executeChain方法，其中executeChain方法为当前URL选择了下一Filter。

   ```java
   protected void executeChain(ServletRequest request, ServletResponse response, FilterChain origChain)
           throws IOException, ServletException {
       //根据请求解析出需要执行的FilterChain(拦截器链)
       FilterChain chain = getExecutionChain(request, response, origChain);
       //拦截器处理
       chain.doFilter(request, response);
   }
   
       protected FilterChain getExecutionChain(ServletRequest request, ServletResponse response, FilterChain origChain) {
           FilterChain chain = origChain;
           //获取FilterChainResolver
           FilterChainResolver resolver = getFilterChainResolver();
           if (resolver == null) {
               log.debug("No FilterChainResolver configured.  Returning original FilterChain.");
               return origChain;
           }
   
           FilterChain resolved = resolver.getChain(request, response, origChain);
           if (resolved != null) {
               log.trace("Resolved a configured FilterChain for the current request.");
               chain = resolved;
           } else {
               log.trace("No FilterChain configured for the current request.  Using the default.");
           }
   
           return chain;
       }
   ```

![image-20200603104245843](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200603104245843.png)

通过断点可以看到，这里用的resolver是PathMatchingFilterChainResolver（目前shiro也只提供了该解析器）

```java
//org.apache.shiro.web.filter.mgt.PathMatchingFilterChainResolver#getChain
public FilterChain getChain(ServletRequest request, ServletResponse response, FilterChain originalChain) {
    FilterChainManager filterChainManager = getFilterChainManager();
    if (!filterChainManager.hasChains()) {
        return null;
    }

    String requestURI = getPathWithinApplication(request);
    //the 'chain names' in this implementation are actually path patterns defined by the user.  We just use them
    //as the chain name for the FilterChainManager's requirements
    for (String pathPattern : filterChainManager.getChainNames()) {

        // If the path does match, then pass on to the subclass implementation for specific checks:
        if (pathMatches(pathPattern, requestURI)) {
            if (log.isTraceEnabled()) {
                log.trace("Matched path pattern [" + pathPattern + "] for requestURI [" + requestURI + "].  " +
                        "Utilizing corresponding filter chain...");
            }
            return filterChainManager.proxy(originalChain, pathPattern);
        }
    }

    return null;
}
```

这个方法主要做了以下事情

1. 获取FilterChainManager
2. 获取当前接口url，只保留controller和方法
3. 将接口url和filterChainManager中的chians一一进行匹配，若匹配成功则返回一个代理的FilterChain

可以看到，resolver通过拿到当前url和filterChainManager中的配置进行匹配，我们先看看这个filterChainManager里放了什么东西

![image-20200603104931705](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200603104931705.png)

- filters：配置的shiroFilter，可以看到此处我自己配置的authFilter也在其中
- filterChains：chainName -> chain的Map，在该方法中 name是url pattern，value是对应的FilterChain。猜测此处是读取配置文件，有兴趣的可以深究。

```java
public FilterChain proxy(FilterChain original, String chainName) {
    NamedFilterList configured = getChain(chainName);
    if (configured == null) {
        String msg = "There is no configured chain under the name/key [" + chainName + "].";
        throw new IllegalArgumentException(msg);
    }
    return configured.proxy(original);
}
```

如果匹配，则获取到该chainName对应的FilterChain，这个FilterChain其实是一个org.apache.shiro.web.filter.mgt.NamedFilterList对象，继承List类，并指定Filter为泛型，这里的实现类是org.apache.shiro.web.filter.mgt.SimpleNamedFilterList，里面持有了一个Filter的List，重写了add、get等List接口方法。

proxy方法其实是实例化了一个org.apache.shiro.web.servlet.ProxiedFilterChain对象，并将请求原始filterChain和通过匹配得到的FilterList赋值给这个对象。ProxiedFilterChain继承了javax.servlet.FilterChain

```java
public class ProxiedFilterChain implements FilterChain {
    private static final Logger log = LoggerFactory.getLogger(ProxiedFilterChain.class);
    //原始FilterChain
    private FilterChain orig;
    //匹配filters
    private List<Filter> filters;
    private int index = 0;

    public ProxiedFilterChain(FilterChain orig, List<Filter> filters) {
        if (orig == null) {
            throw new NullPointerException("original FilterChain cannot be null.");
        }
        this.orig = orig;
        this.filters = filters;
        this.index = 0;
    }

    public void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {
        //重写doFilter方法，如果拓展filters已经执行完，或者为空，则执行原filterChain的doFilter方法，此处相当于是把控制权归还给原filterChain，否则走拓展filters的拦截处理
        if (this.filters == null || this.filters.size() == this.index) {
            //we've reached the end of the wrapped chain, so invoke the original one:
            if (log.isTraceEnabled()) {
                log.trace("Invoking original filter chain.");
            }
            this.orig.doFilter(request, response);
        } else {
            if (log.isTraceEnabled()) {
                log.trace("Invoking wrapped filter at index [" + this.index + "]");
            }
            this.filters.get(this.index++).doFilter(request, response, this);
        }
    }
}
```

可以看到，shiro是提供了通过配置，将特定url和特定拦截器（可多个）匹配，其实这些拦截器并没有注册到全局的配置中，即web.xml，而是通过AbstractShiroFilter做了一层处理，将特定拦截器的处理调用放在了AbstractShiroFilter，而不是全局的FilterChain。

这里再往前看一点，看一下这个拦截器是谁调用的。

![image-20200603145634430](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200603145634430.png)

其实可以看到调用doFilter的是org.springframework.web.filter.DelegatingFilterProxy，这个是spring提供的一个代理拦截器，可以看到这个拦截器代理的是AbstractShiroFilter，同时从filterChain中也可以看出

![image-20200603145944591](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200603145944591.png)

此处其实我的应用配置的filterChain还是有点问题的，多配置了AuthFilter（自定义拦截器）和ShiroFilterFilter，这样会重复执行同样的Filter，不过shiro的Filter由于大多继承了OncePerRequestFilter，所以不会重复执行

## 登录

作者整合shiro的方式是自定义Realm和自定义Filter，登录时自己调用subject.login()，代码如下

```java
public String login(LoginParam loginParam) {
   	//数据库登录校验逻辑（自实现部分，不贴了）
    ...
   	//校验通过后，调用shiro的登录，首先需要获取当前线程的登录用户主体
    Subject subject = SecurityUtils.getSubject();
    subject.login(realmToken);
    return token;
}
```

看一下subject的登录逻辑，login逻辑在org.apache.shiro.subject.support.DelegatingSubject中实现，这个类是shiro最重要的一个subject，实现了subject的大部分功能

![image-20200603165352810](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200603165352810.png)

```java
//org.apache.shiro.subject.support.DelegatingSubject#login
public void login(AuthenticationToken token) throws AuthenticationException {
    clearRunAsIdentitiesInternal();
    //实际登录交给securityManager实现，返回一个登录后新建的subject
    Subject subject = securityManager.login(this, token);

    PrincipalCollection principals;

    String host = null;

    if (subject instanceof DelegatingSubject) {
        DelegatingSubject delegating = (DelegatingSubject) subject;
        principals = delegating.principals;
        host = delegating.host;
    } else {
        principals = subject.getPrincipals();
    }

    if (principals == null || principals.isEmpty()) {
        String msg = "Principals returned from securityManager.login( token ) returned a null or " +
                "empty value.  This value must be non null and populated with one or more elements.";
        throw new IllegalStateException(msg);
    }
    //将获取到的权限信息等赋值给自己
    this.principals = principals;
    this.authenticated = true;
    if (token instanceof HostAuthenticationToken) {
        host = ((HostAuthenticationToken) token).getHost();
    }
    if (host != null) {
        this.host = host;
    }
    Session session = subject.getSession(false);
    if (session != null) {
        this.session = decorate(session);
    } else {
        this.session = null;
    }
}
```

通过断点可以看到，实际使用的securityManager是DefaultSecurityManager

```java
//org.apache.shiro.mgt.DefaultSecurityManager#login
public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
    AuthenticationInfo info;
    try {
        info = authenticate(token);
    } catch (AuthenticationException ae) {
        try {
            onFailedLogin(token, ae, subject);
        } catch (Exception e) {
            if (log.isInfoEnabled()) {
                log.info("onFailedLogin method threw an " +
                        "exception.  Logging and propagating original AuthenticationException.", e);
            }
        }
        throw ae; //propagate
    }

    Subject loggedIn = createSubject(token, info, subject);

    onSuccessfulLogin(token, info, loggedIn);

    return loggedIn;
}

public AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
    //验证委托给了authenticator
    return this.authenticator.authenticate(token);
}
```

#### Authenticator

org.apache.shiro.authc.AbstractAuthenticator，封装了监听器，验证成功/失败/登出可通知监听器调用对应的方法；声明抽象方法doAuthenticate，处理具体的验证逻辑

```java
public final AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
	//...省略部分代码
    AuthenticationInfo info;
    try {
        info = doAuthenticate(token);
        //...省略部分代码
    } catch (Throwable t) {
        AuthenticationException ae = null;
        if (t instanceof AuthenticationException) {
            ae = (AuthenticationException) t;
        }
        if (ae == null) {
            //...省略部分代码
        }
        try {
            notifyFailure(token, ae);
        } catch (Throwable t2) {
            //...省略部分代码
        }
        throw ae;
    }
	//...省略部分代码
    notifySuccess(token, info);
    return info;
}
```

再看看子类org.apache.shiro.authc.pam.ModularRealmAuthenticator的doAuthenticate方法做了啥

```java
protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
    //校验是否有Realms
    assertRealmsConfigured();
    Collection<Realm> realms = getRealms();
    if (realms.size() == 1) {
        //单个Realm验证
        return doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);
    } else {
        //多Realm验证
        return doMultiRealmAuthentication(realms, authenticationToken);
    }
}

//org.apache.shiro.authc.pam.ModularRealmAuthenticator#doSingleRealmAuthentication
protected AuthenticationInfo doSingleRealmAuthentication(Realm realm, AuthenticationToken token) {
    //1、校验该token是否支持当前realm
    if (!realm.supports(token)) {
        String msg = "Realm [" + realm + "] does not support authentication token [" +
                token + "].  Please ensure that the appropriate Realm implementation is " +
                "configured correctly or that the realm accepts AuthenticationTokens of this type.";
        throw new UnsupportedTokenException(msg);
    }
    //2、获取验证信息
    AuthenticationInfo info = realm.getAuthenticationInfo(token);
    if (info == null) {
        String msg = "Realm [" + realm + "] was unable to find account data for the " +
                "submitted AuthenticationToken [" + token + "].";
        throw new UnknownAccountException(msg);
    }
    return info;
}
```

现在我们知道，securityManager将获取验证信息委托给验证器Authenticator，而ModularRealmAuthenticator则调用Realm，对Realm的分析放到后文，我们此处知道他需要实现能够获取验证信息即可，这里通过断点可以看到持有的realm是我们自定义的Realm。

```java
//org.apache.shiro.realm.AuthenticatingRealm#getAuthenticationInfo
public final AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
	//1、先获取缓存的验证信息
    AuthenticationInfo info = getCachedAuthenticationInfo(token);
    if (info == null) {
        //2、缓存取不到，则调用子类
        info = doGetAuthenticationInfo(token);
        log.debug("Looked up AuthenticationInfo [{}] from doGetAuthenticationInfo", info);
        if (token != null && info != null) {
            //3、如果允许缓存，则缓存起来
            cacheAuthenticationInfoIfPossible(token, info);
        }
    } else {
        log.debug("Using cached authentication info [{}] to perform credentials matching.", info);
    }

    if (info != null) {
        //检查之前生成的验证信息（登录入参）和获取的是否相同
        assertCredentialsMatch(token, info);
    } else {
        log.debug("No AuthenticationInfo found for submitted AuthenticationToken [{}].  Returning null.", token);
    }

    return info;
}

/**
 * 获取缓存的验证信息
 */
private AuthenticationInfo getCachedAuthenticationInfo(AuthenticationToken token) {
    AuthenticationInfo info = null;

    Cache<Object, AuthenticationInfo> cache = getAvailableAuthenticationCache();
    if (cache != null && token != null) {
        log.trace("Attempting to retrieve the AuthenticationInfo from cache.");
        Object key = getAuthenticationCacheKey(token);
        info = cache.get(key);
        if (info == null) {
            log.trace("No AuthorizationInfo found in cache for key [{}]", key);
        } else {
            log.trace("Found cached AuthorizationInfo for key [{}]", key);
        }
    }

    return info;
}

/**
 * 获取缓存器（其实就是一个存放用户信息缓存的地方，暂且这么叫他）
 */
private Cache<Object, AuthenticationInfo> getAvailableAuthenticationCache() {
    //直接获取该Realm持有的
    Cache<Object, AuthenticationInfo> cache = getAuthenticationCache();
    boolean authcCachingEnabled = isAuthenticationCachingEnabled();
    if (cache == null && authcCachingEnabled) {
        //如果为空且可以缓存，则懒加载该缓存器
        cache = getAuthenticationCacheLazy();
    }
    return cache;
}
/**
 * 懒加载缓存器
 */
private Cache<Object, AuthenticationInfo> getAuthenticationCacheLazy() {

    if (this.authenticationCache == null) {

        log.trace("No authenticationCache instance set.  Checking for a cacheManager...");
		//通过CacheManager获取
        CacheManager cacheManager = getCacheManager();

        if (cacheManager != null) {
            String cacheName = getAuthenticationCacheName();
            log.debug("CacheManager [{}] configured.  Building authentication cache '{}'", cacheManager, cacheName);
            this.authenticationCache = cacheManager.getCache(cacheName);
        }
    }

    return this.authenticationCache;
}
```

org.apache.shiro.realm.AuthenticatingRealm#getAuthenticationInfo中第二步提到的，调用子类获取验证信息的，会发现调用的是自定义Realm重写的方法。

验证完成后的逻辑：

1. 获取成功后，就会一直返回到securityManager，然后会再创建一个subject，此处可以进行RememberMe操作，就不细究了
2. 拿到securityManager中返回的新建的subject信息，并将其中一些字段赋值给自己（调用登录的subject）

至此，登录则完成了，可见shiro登录主要是获取用户信息（可缓存），然后设置到线程上下文中。

#### 验证信息缓存

刚刚上面说到，验证信息是可以缓存的，shiro提供了内存的缓存，我们也可以自己实现，此处举个例子，使用redis做缓存。

1. 需要设置一个CacheManager，`securityManager.setCacheManager(cacheManager(redisTemplate));`

   这里的CacheManager其实是自实现的一个Manager，我们看一下代码

   ```java
   public class ShiroRedisCacheManager extends AbstractCacheManager {
   
       private RedisTemplate<String, Object> redisTemplate;
   
       public ShiroRedisCacheManager(RedisTemplate<String, Object> redisTemplate) {
           this.redisTemplate = redisTemplate;
       }
   
       @Override
       protected Cache createCache(String name) throws CacheException {
           return new ShiroRedisCache(redisTemplate);
       }
   }
   ```

   可以看到，实现了一个createCache方法，并返回了一个ShiroRedisCache

2. 实现自己的shiro缓存，ShiroRedisCache，这里需要实现shiro的cache接口，该接口类似map，本来缓存也类似Map结构。

   ```java
   public class ShiroRedisCache<K,V> implements Cache<K,V> {
       private RedisTemplate<String, Object> redisTemplate;
   
       private static final String prefix = "shiro_redis";
   
       public ShiroRedisCache(RedisTemplate<String, Object> redisTemplate) {
           this.redisTemplate = redisTemplate;
       }
   
       @Override
       public V get(K k) throws CacheException {
           if (k == null) {
               return null;
           }
           String bytes = getStringKey(k);
           return (V)redisTemplate.opsForValue().get(bytes);
       }
   
       @Override
       public V put(K k, V v) throws CacheException {
           if (k== null || v == null) {
               return null;
           }
   
           String bytes = getStringKey(k);
           redisTemplate.opsForValue().set(bytes, v, Duration.ofSeconds(300));
           return v;
       }
   
       @Override
       public V remove(K k) throws CacheException {
           if (k == null) {
               return null;
           }
           String bytes = getStringKey(k);
           V v = (V) redisTemplate.opsForValue().get(bytes);
           redisTemplate.delete(bytes);
           return v;
       }
   
       @Override
       public void clear() throws CacheException {
       }
   
       @Override
       public int size() {
           return 0;
       }
   
       @Override
       public Set<K> keys() {
           return null;
       }
   
       @Override
       public Collection<V> values() {
           return null;
       }
   
       private String getStringKey(K k) {
           String key;
           if (k instanceof String) {
               key = prefix + k;
           } else {
               String s = JSON.toJSONString(k);
               key = prefix + s;
           }
           return key;
       }
   ```

查看断点信息，发现cache是自定义的类，继续进入get方法，发现进入到自定义缓存的get实现

![image-20200603190103157](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200603190103157.png)![image-20200603190220880](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200603190220880.png)



## 授权

## 会话管理

## 总结

1. shiro的强大在于将用户体系抽象成了四大模块
2. shiro提供了各种可拓展接口和抽象类，使用者可根据自己需求自行拓展，但由于中文文档过少，开发者通常需要阅读源码和英文文档去了解shiro的一些类，才能在合适的地方拓展，本文旨在以作者所使用的例子，去介绍shiro的一些核心类，以及阅读shiro源码的一些方法