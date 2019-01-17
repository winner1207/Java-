# 1、什么是线程封闭？
它其实就是把对象封装到一个线程里，只有一个线程能看到这个对象，那么这个对象就算不是线程安全的，也不会出现任何线程安全方面的问题。

线程封闭技术有一个常见的应用：
> 数据库连接对应jdbc的Connection对象，Connection对象在实现的时候并没有对线程安全做太多的处理，jdbc的规范里也没有要求Connection对象必须是线程安全的。  
> 实际在服务器应用程序中，线程从连接池获取了一个Connection对象，使用完再把Connection对象返回给连接池，由于大多数请求都是由单线程采用同步的方式来处理的，并且在Connection对象返回之前，连接池不会将它分配给其他线程。因此这种连接管理模式处理请求时隐含的将Connection对象封闭在线程里面，这样我们使用的connection对象虽然本身不是线程安全的，但是它通过线程封闭也做到了线程安全。

# 2、线程封闭的种类：
- （1）Ad-hoc 线程封闭：  
Ad-hoc线程封闭是指，维护线程封闭性的职责完全由程序实现来承担。Ad-hoc线程封闭是非常脆弱的，因为没有任何一种语言特性，例如可见性修饰符或局部变量，能将对象封闭到目标线程上。事实上，对线程封闭对象（例如，GUI应用程序中的可视化组件或数据模型等）的引用通常保存在公有变量中。

- （2）堆栈封闭：  
堆栈封闭其实就是方法中定义局部变量。不存在并发问题。多个线程访问一个方法的时候，方法中的局部变量都会被拷贝一份到线程的栈中（Java内存模型），所以局部变量是不会被多个线程所共享的。

- （3）ThreadLocal线程封闭：  
它是一个特别好的封闭方法，其实ThreadLocal内部维护了一个map,map的key是每个线程的名称，而map的value就是我们要封闭的对象。ThreadLocal提供了get、set、remove方法，每个操作都是基于当前线程的，所以它是线程安全的。 

![](https://img-blog.csdn.net/20180411204833840?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

<pre>
    //ThreadLocal的get方法源码
    public T get() {
        Thread t = Thread.currentThread();//当前线程对象
        ThreadLocalMap map = getMap(t);//get操作基于当前线程
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
</pre>

# 3、Springboot框架中使用ThreadLocal
说明：  
1、这里不描述springboot框架的搭建过程，假定你已经有了一个可以正常运行的springboot简单项目。  
2、我们这里的例子使用的是springboot框架中的filter与Interceptor来使用threadLocal，对于Springboot的filter与Interceptor不做过多的讲解。  

coding：  
（1）创建一个包含ThreadLocal对象的类，并提供基础的添加、删除、获取操作。

<pre>
public class RequestHolder {

    private final static ThreadLocal<Long> requestHolder = new ThreadLocal<>();

    public static void add(Long id) {
        requestHolder.set(id);
    }

    public static Long getId() {
        return requestHolder.get();
    }

    public static void remove() {
        requestHolder.remove();
    }
}
</pre>

（2）创建Filter，在Filter中对ThreadLocal做添加操作。

<pre>
public class HttpFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest servletRequest,
                         ServletResponse servletResponse,
                         FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        //打印当前线程的ID与请求路径
        log.info("do filter, {}, {}", Thread.currentThread().getId(), request.getServletPath());
        //将当前线程ID添加到ThreadLocal中
        RequestHolder.add(Thread.currentThread().getId());
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {

    }
}
</pre>

（3)创建controller，在controller中获取到filter中存入的值

<pre>
@Controller
@RequestMapping("/threadLocal")
public class ThreadLocalController {

    @RequestMapping("/test")
    @ResponseBody
    public Long test() {
        return RequestHolder.getId();
    }
}
</pre>

（4）创建拦截器Interceptor，在拦截器中删除刚才添加的值

<pre>
public class HttpInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) throws Exception {
        log.info("preHandle");
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, 
                            Object handler, Exception ex) throws Exception {
        RequestHolder.remove();
        log.info("afterCompletion");
        return;
    }
}
</pre>

（5）在springboot的启动类Application中注册filter与Interceptor。要继承WebMvcConfigurerAdapter 类。（我这里的启动类名为：ConcurrencyApplication）

<pre>
@SpringBootApplication
public class ConcurrencyApplication extends WebMvcConfigurerAdapter {

    public static void main(String[] args) {
        SpringApplication.run(ConcurrencyApplication.class, args);
    }

    //注册过滤器
    @Bean
    public FilterRegistrationBean httpFilter() {
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        registrationBean.setFilter(new HttpFilter());
        //设置要过滤的url
        registrationBean.addUrlPatterns("/threadLocal/*");
        return registrationBean;
    }

    //注册拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new HttpInterceptor()).addPathPatterns("/**");
    }
}
</pre>

（6）运行程序，访问 http://localhost:8080/threadLocal/test 结果如下：

- 页面中打印出我们当前的线程ID： 

![](https://img-blog.csdn.net/20180411203503274?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- 查看控制台： 

![](https://img-blog.csdn.net/20180411203545536?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

从控制台的打印日志我们可以看出，首先filter过滤器先获取到我们当前的线程ID为40、我们当前的请求路径为/threadLocal/test ，紧接着进入了我们的Interceptor的preHandle方法中，打印了preHandle字样。最后进入了我们的Interceptor的afterCompletion方法，删除了我们之前存入的值，并打印了afterCompletion字样。
