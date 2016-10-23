---
layout: post
title:  "内存缓存的一种典型实现"
date:   2016-09-24 22:00:00
categories: java web
tags: featured
image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
image2: /assets/article_images/2014-11-30-mediator_features/night-track-mobile.JPG
---
### *摘要*
>　　缓存技术作为一种可以有效加快数据读取速度的机制，被广泛应用于各类应用场景中，以提高运行效率，比如CPU寄存器的指令缓存、数据库缓存等等。在我之前的一项工作里，我需要在程序中随时读取整个项目的数据字典，而数据字典与其他数据一样存储于数据库中，频繁读写消耗大，因此我选择了内存缓存的方式去实现这个需求。
<br>
>　　在本文中，我将针对这个应用场景对内存缓存的一种实现方式进行介绍，如果你对该方面内容感兴趣，可以继续阅读下文。

# *1. 缓存技术* #
　　缓存，其本质依然是一种存储数据的介质，以数据缓冲区的形式存在，但它往往具备比正常存储器更快的数据读写速度，这种关系就像是寄存器与内存，或者说是内存与磁盘。而缓存的特点也决定了我们使用它的原因，即提高程序运行的效率。在比较典型的缓存应用场景里，我们会在第一次读取数据的时候，将数据同时存放在缓存中，之后如果读取相同的数据就可以直接从缓存获取，而不是访问原本的介质。下面我总结了缓存的几个关键特点，了解这些特点有助于我们判断什么时候该去使用缓存，需要时又该怎样去实现缓存。

* **更快的数据读写速度**　　这点在前文中已经提到，也是我们使用缓存的最主要原因，它的这个特点可以为我们的程序带来更快的运行速度，减小原有存储介质的负担。
* **单位容量的成本更高**　　这也说明了为什么我们不用更快的缓存去完全替换原有的存储介质，也告诉我们不是所有的数据都适合存放在缓存中，因为缓存的容量往往是有限的。
* **非永久存储**　　缓存是数据存储的缓冲区，仅仅起到临时缓冲数据的作用，有效期一般是程序运行期间或者更短，而不像文件或者磁盘可以长久的保留数据。
* **存在时效性问题**　　如果存放在缓存中的数据对应于原存储介质中的部分发生了改变，那么我们读取缓存时可能拿到的就不再是最新的数据，这一点是我们使用缓存时的重要考量，时效性要求高的数据并不适合使用缓存。而常见的应对策略是定期从原介质更新缓存中的数据，周期可以由程序的需求来决定。

　　在接下来介绍的应用场景里，我使用内存作为缓存去存储程序中的数据字典信息。简单说明一下原因。我所使用到的数据字典是对我项目中涉及到的元素进行的定义和说明，起到目录的作用，是程序运行过程中不会发生改变（若需要改变往往是项目版本更新的时候）但又频繁访问的数据，而使用内存缓存数据字典将比每次都访问数据库（即磁盘）快上不少，也减轻了数据库的压力。这种需求的实现方式很多，我将在下面贴出我所采用的比较典型的一种，可以根据项目的不同需求做调整。

> *Cache.java*

	package com.ihuyang.cache;
	
	/** 
	 * 缓存接口
	 * @author huyang
	 * @version 创建时间：2016-10-21
	 */
	public interface Cache {
		
		/**
		 * 存放缓存数据，同key的数据会被更新
		 * @param key 数据键值
		 * @param data 数据对象
		 */
		public void put(String key, Object data);
		
		/**
		 * 获取缓存数据
		 * @param key 数据键值
		 * @return 数据对象
		 */
		public Object get(String key);
		
		/**
		 * 判断缓存是否包含指定数据
		 * @param 数据键值
		 * @return 判断结果
		 */
		public boolean containsKey(String key);
		
		/**
		 * 清空缓存
		 */
		public void clear();
		
		/**
		 * 移除缓存中的指定数据
		 * @param key 数据键值
		 */
		public void remove(String key);
	
	}

> *CacheImpl.java*

	package com.ihuyang.cache;
	
	import java.util.Map;
	import java.util.concurrent.ConcurrentHashMap;
	
	/**
	 * 缓存接口实现类
	 * @author huyang
	 * @version 创建时间：2016-10-21
	 */
	public class CacheImpl implements Cache {
	
		// 使用线程安全的ConcurrentHashMap作为缓存数据的容器
		private static Map<String, Object> CAS = new ConcurrentHashMap<String, Object>();
	
		@Override
		public void put(String key, Object data) {
			// TODO Auto-generated method stub
			CAS.put(key, data);
		}
	
		@Override
		public Object get(String key) {
			// TODO Auto-generated method stub
			return CAS.get(key);
		}
	
		@Override
		public boolean containsKey(String key) {
			// TODO Auto-generated method stub
			return CAS.containsKey(key);
		}
	
		@Override
		public void clear() {
			// TODO Auto-generated method stub
			CAS.clear();
		}
	
		@Override
		public void remove(String key) {
			// TODO Auto-generated method stub
			if (CAS.containsKey(key)) {
				CAS.remove(key);
			}
		}
	
	}

> *CacheUtils.java*

	package com.ihuyang.utils;
	
	import com.ihuyang.cache.Cache;
	import com.ihuyang.cache.CacheImpl;
	
	/**
	 * 缓存工具类
	 * @author huyang
	 * @version 创建时间：2016-10-21
	 */
	public class CacheUtils {
		private static Cache cache;
	
		static {
			cache = new CacheImpl();
		}
	
		/**
		 * 存放缓存数据
		 * @param key 数据键值
		 * @param data 数据对象
		 */
		public static void put(String key, Object data) {
			cache.put(key, data);
		}
	
		/**
		 * 判断缓存是否包含指定数据
		 * @param key 数据键值
		 * @return 判断结果
		 */
		public static boolean containsKey(String key) {
			return cache.containsKey(key);
		}
	
		/**
		 * 获取指定缓存数据
		 * @param key 数据键值
		 * @return 数据对象
		 */
		public static Object get(String key) {
			return cache.get(key);
		}
	
		/**
		 * 清空缓存数据
		 */
		public static void clearCache() {
			cache.clear();
		}
	}
　　这里之所以创建工具类CacheUtil，将Cache类封装起来，而不是直接在程序中使用Cache类，是因为可以在CacheUtils类中声明更多复杂的缓存操作逻辑，以满足应用特别的需求变化。

　　到这里，内存缓存的基础代码已完成，可以直接套用到各类项目中。接下来就是根据应用的实际情况去初始化缓存、读取缓存以及更新缓存了。下面的DataCache.java是我在实现数据字典缓存时的代码，可供参考。

> *DataCache.java*

	package com.ihuyang.cache;
	
	import java.util.ArrayList;
	import java.util.List;
	import org.apache.commons.lang.StringUtils;
	import com.ihuyang.dao.DictInfMapper;
	import com.ihuyang.entity.DictInf;
	import com.ihuyang.utils.CacheUtils;
	import com.ihuyang.utils.SpringUtils;
	
	/**
	 * 向系统提供数据缓存的服务
	 * @author huyang
	 * @version 创建时间：2016-10-21
	 */
	public class DataCache {
	
		/**
		 * 缓存初始化
		 */
		public static void init() {
			initDataDict();
		}
	
		/**
		 * 数据字典缓存的初始化
		 */
		@SuppressWarnings("unchecked")
		private static void initDataDict() {
			/*
			 * 从数据库中查询出所有数据字典项，
			 * 数据源除了数据库，也可以是文件，
			 * 如果是数据库，这里的代码由持久化框架决定，我这里对应的是MyBatis
			 */
			DictInfMapper dictInfMapper = (DictInfMapper) SpringUtils
					.getBean("DictInfMapper");
			List<DictInf> dictInfs = dictInfMapper.selectAll();
	
			for (DictInf dictInf : dictInfs) {
				if (CacheUtils.containsKey(dictInf.getDataTp())) {
					// 若缓存中已存在该类别的数据字典项，则在该类中添加新字典项
					((List<DictInf>) CacheUtils.get(dictInf.getDataTp()))
							.add(dictInf);
				} else {
					// 若缓存中不存在该类别的数据字典项，则新创建该类别的List，然后在里面添加新字典项
					List<DictInf> dicts = new ArrayList<DictInf>();
					dicts.add(dictInf);
					CacheUtils.put(dictInf.getDataTp(), dicts);
				}
			}
	
		}
	
		/**
		 * 根据数据字典项的类型获取到字典项列表
		 * @param dataTp 数据字典项类型
		 * @return 字典项列表
		 */
		@SuppressWarnings("unchecked")
		public static List<DictInf> getDictList(String dataTp) {
			return (List<DictInf>) CacheUtils.get(dataTp);
		}
	
		/**
		 * 根据数据字典项的类型和字典项key，获取到指定字典项
		 * @param dataTp 数据字典项类型
		 * @param dataKey 字典项的key
		 * @return 指定字典项
		 */
		@SuppressWarnings("unchecked")
		public static DictInf getDictInf(String dataTp, String dataKey) {
			List<DictInf> dictInfs = (List<DictInf>) CacheUtils.get(dataTp);
			for (DictInf dictInf : dictInfs) {
				if (StringUtils.equals(dictInf.getDataKey(), dataKey))
					return dictInf;
			}
			return null;
		}
	}
　　我在程序的其他代码中对缓存的所有操作，都是通过DataCache类来完成的，比如我调用getDictList的两个重载方法使用缓存中的数据字典。但使用DataCache类之前首先需要初始化，可以是第一次使用的时候，也可以是服务器启动的时候，对于服务器启动的情况，如果使用了Spring框架，可以通过如下代码在加载bean的时候完成。

	<bean id="serverInit" class="com.ihuyang.cache.ServerInit" init-method="init"/>

　　ServerInit类中的init()方法会在bean加载的时候执行，只要在该init()方法中调用DataCache类中的静态方法init()就可以初始化缓存了。
<br>
<br>
<br>
<br>

>若需转载此文，请注明作者和原文地址。未经本人同意，本文不允许任何人任何形式的商业使用。如有疑问，请联系本人，联系方式详见本页末尾。
<br>
>原文地址：[胡杨的个人博客-《内存缓存的一种简单实现》（2016.09.24）](http://ihuyang.me/java/web/A-typical-implementation-of-memory-cache.html)