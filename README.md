@[TOC](.net core 下的简单二级缓存)

# 为什么写轮子

.net core 下的二级缓存做的已经足够好了，此次编写轮子是因为需要在 .net core 和 .net Framework 4.6 两个框架中共用该类，因此没办法直接使用**MemoryCache** 包

## 缓存思路

这里缓存采用静态变量存储，为了线程安全，采用如下代码定义：
```csharp
 private static  ConcurrentDictionary<string, object> _dict = new ConcurrentDictionary<string, object>();
```
有了词典就比较好办了，需要二级缓存时就保存在该词典类，如果访问过期就干掉它。如果没有在词典中查找到，就到Redis缓存中拉取，并放入词典缓存，过期时间半分钟。
```csharp
//从Redis拉取，这里没有封装redis键的过期时间，因此用**带过期时间的缓存类**简单处理，避免其他进程过于频繁的拉取Redis
var cache = Cache.CacheManager.GetCache();
var val = cache?.Get<CacheClass<T>>(key) ;
 if (val != null)
 {
     _dict.TryAdd(key, val);
 }
 return val?.Value;
```
## 带过期时间的缓存类

自己管理过期 

```csharp
	/// <summary>
    /// 带过期时间的缓存类
    /// </summary>
    /// <typeparam name="T"></typeparam>
    public class CacheClass<T> where T:class
    {
        /// <summary>
        /// 缓存值
        /// </summary>
        public T Value { private set; get; }

        /// <summary>
        /// 绝对过期时间
        /// </summary>
        public DateTime ExpireDate {private set;get;}

        /// <summary>
        /// 判断是否过期,true-过期
        /// </summary>
        /// <returns></returns>
        public bool IsExpire()
        {
            return (ExpireDate < DateTime.Now);
        }
       
        /// <summary>
        /// 构造
        /// </summary>
        /// <param name="obj"></param>
        /// <param name="expireDate">绝对过期时间</param>
        public CacheClass(T obj, DateTime? expireDate = null)
        {
            this.Value = obj;
            if (!expireDate.HasValue)
            {
                ExpireDate = DateTime.Now + new TimeSpan(0, 10, 0);
            }
            else
            {
                ExpireDate = expireDate.Value;
            }
        }
    }
```


## 完整的Get和Set方法


```csharp
public static T Get<T>(string key) where T : class
 {
     object obj ;
     if (_dict.TryGetValue(key, out obj))
     {
         var cc = (CacheClass<T>)obj;
         if (cc.IsExpire())
         {
             _dict.TryRemove(key, out obj);
         }
         else
         {
             return cc.Value;
         }
     }

     //从redis中取值
     var cache = Cache.CacheManager.GetCache();
     var val = cache?.Get<CacheClass<T>>(key) ;
     if (val != null)
     {
         _dict.TryAdd(key, val);
     }
     else
     {
         _dict.TryRemove(key,out obj);
     }
     return val?.Value;
 }

 /// <summary>
 /// 设置缓存
 /// </summary>
 /// <typeparam name="T"></typeparam>
 /// <param name="key"></param>
 /// <param name="obj"></param>
 /// <param name="expireDate"></param>
 public static void Set<T>(string key, T obj, DateTime? expireDate = null) where T : class
 {
     var cc = new CacheClass<T>(obj, expireDate);
     if (!cc.IsExpire())
     {
         _dict.TryAdd(key, cc);
         var cache = Cache.CacheManager.GetCache();
         cache?.Set<CacheClass<T>>(key, cc, cc.ExpireDate);
     }
 }
```



## 结尾

优点：简单易用
缺点：缺失很多缓存的特性，并且对于缓存的大小等都无限制。滥用很可能导致本地内存暴增。

### 引用链接
1. [口袋代码仓库](http://codeex.cn)
2. [在线计算器](http://jisuanqi.codeex.cn)
3. 本节源码：github

