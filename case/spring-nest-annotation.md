---

layout: single
title: Case Study（记一次SpringAOP实践过程-包扫描和嵌套注解）
permalink: /case/spring-nest-annotation.html

classes: wide

author: Bob Dong

---

# 前言

每一次实践得出结论，得出的对过往理论的印证，都是一次悟道，其收益远大于争论和抱怨。

技术是一件比较客观的事，正确与错误，其实就摆在哪里，意见不统一，写段代码试验一下就好了，一段代码印证不了的时候，就多写几段。



先同一个案例说起


挺简单的一个案例，通过SpringAOP和注解，使用Guava缓存。代码如下：

GuavaCache.java

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface GuavaCache {
	/**
	 * group : 一个group代表一个cache，不传或者""则使用 class+method 作为cache名字
	 * @return
	 */
	public String group() default "";
	/**
	 * key : 注意，所有参数必须实现GuavaCacheInterface接口，如果不实现，则会用toString()的MD5作为Key
	 * @return
	 */
    public String key() default "";
    /**
     * 过期时间，缺省30秒
     * @return
     */
    public long timeout() default 30;
    /**
     * 缓存最大条目，缺省10000
     * @return
     */
    public long size() default 10000;
    /**
     * 是否打印日志
     * @return
     */
    public boolean debug() default false;
}
```

**GuavaInterface.java**

```
/**
 * 使用GuavaCache注解时，如果传入参数是对象，则必须实现这个类
 *
 */
public interface GuavaCacheInterface {
    public String getCacheKey();
}
```

**GuavaCacheProcessor.java**

```
import java.lang.reflect.Method;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Map;
import java.util.concurrent.TimeUnit;
 
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
 
import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import com.google.common.collect.Maps;
 
/**
 * GuavaCache注解处理器
 *
 */
@Component
@Aspect
public class GuavaCacheProcessor {
    
    private static final Logger logger = LoggerFactory.getLogger(GuavaCacheProcessor.class);
    private static Map<String, Cache<String, Object>> cacheMap = Maps.newConcurrentMap();
    
    @Around("execution(* *(..)) && @annotation(guavaCache)")
    public Object aroundMethod(ProceedingJoinPoint pjd, GuavaCache guavaCache) throws Throwable {
        Cache<String, Object> cache = getCache(pjd, guavaCache);
        String key = getKey(pjd);
        boolean keyisnull = (null == key) || ("".equals(key));
        if(guavaCache.debug()) {
            logger.info("GuavaCache key : {} begin", key);
        }       
        Object result = null;
        if(!keyisnull) { 
            result = cache.getIfPresent(key);
            if(result != null) {
                return result;
            }
        }
        try {
            result = pjd.proceed();
            if(!keyisnull) {
                cache.put(key, result);
            }
        } catch (Exception e) {
            throw e;
        }
        if(guavaCache.debug()) {
            logger.info("GuavaCache key : {} end", key);
        }        
        return result;
    }
 
    /**
     * 获取Cache
     * @param pjd
     * @param guavaCache
     * @return
     */
    private Cache<String, Object> getCache(ProceedingJoinPoint pjd, GuavaCache guavaCache) {
        String group = guavaCache.group();
        if(group == null || "".equals(group)) {
            MethodSignature signature = (MethodSignature) pjd.getSignature();
            Method method = signature.getMethod();
            Class<?> clazz =  method.getDeclaringClass();
            group = clazz.getName();
        }
        Cache<String, Object> cache = cacheMap.get(group);
        if(cache == null) {
            cache = CacheBuilder.newBuilder()
                    .maximumSize(guavaCache.size())
                    .expireAfterWrite(guavaCache.timeout(), TimeUnit.SECONDS)
                    .build();
            cacheMap.put(group, cache);
        }
        return cache;
    }
    
    /**
     * 获取Key：方法名+getCacheKey方法(如果没有，则用toString())的MD5值
     * @param pjd
     * @return
     */
    private String getKey(ProceedingJoinPoint pjd) {
        StringBuilder sb = new StringBuilder();
        MethodSignature signature = (MethodSignature) pjd.getSignature();
        Method method = signature.getMethod();
        sb.append(method.getName());
        for(Object param : pjd.getArgs()) {
            if(GuavaCacheInterface.class.isAssignableFrom(param.getClass())) {
                sb.append(((GuavaCacheInterface)param).getCacheKey());
            } else {
                if(!param.getClass().isPrimitive()) {
                    return null;
                }
                sb.append(param.toString());
            }
        }
        String key = md5(sb.toString());
        return key;
    }
    
    /**
     * 进行MD5加密
     *
     * @param info
     *            要加密的信息
     * @return String 加密后的字符串
     */
    private static String md5(String info) {
        byte[] digesta = null;
        try {
            // 得到一个md5的消息摘要
            MessageDigest alga = MessageDigest.getInstance("MD5");
            // 添加要进行计算摘要的信息
            alga.update(info.getBytes());
            // 得到该摘要
            digesta = alga.digest();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        // 将摘要转为字符串
        String rs = byte2hex(digesta);
        return rs;
    }
    
    /**
     * 将二进制转化为16进制字符串
     *
     * @param b
     *            二进制字节数组
     * @return String
     */
    private static String byte2hex(byte[] b) {
        String hs = "";
        String stmp = "";
        for (int n = 0; n < b.length; n++) {
            stmp = (java.lang.Integer.toHexString(b[n] & 0XFF));
            if (stmp.length() == 1) {
                hs = hs + "0" + stmp;
            } else {
                hs = hs + stmp;
            }
        }
        return hs.toUpperCase();
    }
}
```

遇到的第一个问题


问题出现


当bean在applicationContext.xml中通过<bean...>定义时，注解正常；当通过@Service定义时，注解失效。



初步解决


把对于AOP的定义<aop:aspectj-autoproxy proxy-target-class="true" />移到servlet-context.xml（这是controller定义文件）中，OK了。



期间的胡思乱想
context:component-scan 会不会不扫依赖的jar中的bean（因为注解时在依赖的jar包中定义的）
会不会扫描有顺序，先扫自己的，再扫依赖的jar的，由于有先后顺序，导致Spring加载Service类时需要的注解类先没扫到
<aop:aspectj-autoproxy>会不会因为是Web应用，所以就需要在servlet-context.xml中定义呢？
找到真实的原因

类加载两次


监测Service类和注解处理类（通过在构造函数增加System.out），发现类被加载两次，疑惑了一下，加载两次？

看applicationContext.xml，配置正常；看servlet-context.xml，居然扫描的不仅仅是controller类，而是把所有的类都扫描了一遍，这就解释了，为什么<aop:aspectj-autoproxy proxy-target-class="true" />配置在servlet-context.xml中，aop才生效。

因为根据web.xml的加载顺序：context-param>listener>filter>servlet，servlet-context.xml是最后加载的，spring又扫描了一遍，如果不写<aop:...>，就会加载没有aop的类。



<aop:aspectj-autoproxy proxy-target-class="true" />在哪里生效


这个配置，配置在哪个文件中，就会对哪个文件scan的类生效。



解答胡思乱想


看似诡异的问题，更大的可能是我们了解的不够透彻，不够深入；如果规范一点，诡异的问题往往都不出现。

会扫描依赖的jar，根据base-package的定义
有先后顺序，但是扫描某个类的时候，如果有依赖的类还没扫，会马上扫，所以先后没有关系（Spring很强，不会这么弱）
<aop:aspectj-autoproxy>定义在哪个文件，就对哪个文件scan的类生效


第二个问题：嵌套注解


Spring AOP，在同一个类中，嵌套注解时，只对最外层的注解生效，这个有很多文章，解释的很清楚了，可以参考：

http://blog.csdn.net/ldwtill/article/details/22061421

https://www.oschina.net/question/504281_235673

当前SpringAOP的实现，在同一个类中，是不支持嵌套注解的，说不定在以后会实现。

对于嵌套注解，如果一定需要，可以通过aspectJ通过LTW解决，这个我试验过，确实可以解决，但是成本较高，首先jvm需要增加-javaagent，然后项目中的单元测试之类，都要加这个参数，所以如果不是一定需要，就不要这样做了，参考文章：

http://sexycoding.iteye.com/blog/1062372

http://blog.csdn.net/scorpio3k/article/details/6745443

http://blog.csdn.net/qyongkang/article/details/7799603

对于嵌套注解，还有一种解决办法，也是通过aspectJ，在编译期解决，这个连编译器都换了，成本太高，没有去实践。

self字段解决办法：http://strongant.iteye.com/blog/2146152

备注：

对于不同的类之间，嵌套是没有问题的，这个我们相信Spring
在同一个类中，是不支持嵌套注解的，说不定在以后Spring自己就会支持


总结


以上，一次知识总结，没有放过，没有看似解决了，就这样吧，找出了根本问题，很好。

# 后记

