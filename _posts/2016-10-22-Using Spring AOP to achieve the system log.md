---
layout: post
title:  "利用Spring AOP实现系统日志"
date:   2016-10-22 14:00:00
categories: java web
tags: featured
image: /assets/article_images/2016-10-22/title_img.jpg
image2: /assets/article_images/2016-10-22/title_img.jpg
---
### *摘要*
>　　我们可能都非常熟悉大名鼎鼎的OOP（面向对象编程），但却不一定清楚与它同为软件设计思想的AOP。从名称上来看，它们很相似，但两种设计思想在目标上却有着本质的区别。当前广泛被采用的开源框架Spring拥有两个重要的特征，一个是控制反转，另一个就是支持AOP编程，那么AOP到底是什么，它又能为我们的程序带来什么？
<br>
>　　在本文中，我将首先对AOP这一关键的设计思想进行介绍，之后我会使用Spring AOP去实现一个web系统的日志功能，这也是AOP非常重要的应用场景，通过这个例子可以加深我们对AOP的理解。如果你对该方面内容感兴趣，可以继续阅读下文。

# *1. AOP思想* #

* **AOP的概念**
<br>
　　AOP的全称是Aspect Oriented Programming，即**面向切面编程**，与OOP一样是一种软件设计的思想。AOP可以将程序中业务逻辑的各个部分隔离开来，将某些系统功能统一维护，从而达到降低业务逻辑耦合度的目的。
* **为什么需要AOP**
<br>
　　可能从上面简单的概念描述上，我们很难真正理解AOP的含义和特点。其实，当我们知道为什么需要AOP的时候，它的本质也就清楚了。我们知道，面向对象编程（OOP）将数据和对数据的操作封装成了统一的类以及具体的对象，这降低了代码的复杂程度，提高了重用性，让程序的结构变得更为清晰。但是OOP针对的是具有类似属性特征的对象，而不是行为特征，也就是说OOP会将属性类似的对象统一成类，而不是将行为类似的对象统一。我们可以设想，如果多个类都需要实现一个完全一样的方法，但这些类在属性上又不具有互相之间的继承关系，那么每个类就只能各自去实现这个方法了，结论就是OOP无法解决行为上的重用。可能你会说为什么不把这个特别的方法独立成另外一个类，其他类维护它的对象即可，但这样做实际上又会加深代码的耦合，这个类的任何变动都会影响到其他类的使用。
<br>
　　是的，这就是AOP设计出现的背景。OOP处理的是实体，封装了属性和操作，而AOP处理的是切面(切面这个概念将在后面一节说明)，封装了**行为**，它将几个类中共同的行为抽取出来，统一管理，在这些类需要的时候又将行为切入到指定的位置，跟之前直接在各个类中管理行为的结果是一样的。下图展示了AOP将公共行为抽象成切面后的场景，图中出现的切面、连接点等概念都会在下节中说明。

![AOP工作的场景](/assets/article_images/2016-10-22/2016-10-22_1.jpg "AOP工作的场景")

　　**总结一下，OOP和AOP就像是两个相互垂直的设计，OOP横向封装对象的属性和独有行为，而AOP纵向封装对象之间公有的行为，AOP是对OOP有力的补充**，让我们的程序结构变得更加清晰和易于管理。

* **AOP的适用场景**
<br>
　　既然知道了为什么需要AOP，自然它的适用场景就很容易想到了。在我们的程序中，有很多系统级的业务，它们穿插于各项普通业务的逻辑里，比如日志记录、异常管理、权限管理等等。这些系统业务当然可以直接写到各项普通业务的逻辑里，但这样就会有大量的代码重复出现，且过于混乱，不宜后期维护。这时候，就可以使用AOP将系统级业务抽出，统一管理，在适当的接入点将这些系统级业务重新插入普通业务中就可以了。

# *2. AOP的运用* #

　　接下来，我将说明几个AOP中的重要概念，在后面的示例程序中也会涉及到。

* **切面（Aspect）**　　切面就是上图中纵向切入到业务中的部分，每个切面都是一个统一出来的功能模块（往往是系统级功能），比如日志管理、权限管理等，它们可以横切多个业务，从而发挥作用。
* **连接点（Joinpoint）**　　连接点代表了切面切入不同业务的“时机”，即应该什么时候切入。这个时机可以是某个方法被调用执行的时候，也可以是某个方法抛出异常的时候，连接点需要我们为切面定义。
* **切入点（Pointcut）**　　切入点可以和连接点一起理解，它代表了切面切入不同业务的“地点”，即应该在什么位置切入。这个位置可以是某个类中的所有方法，也可以是其中的某个方法，切入点同样需要我们为切面定义。
* **通知**　　通知是在连接点的基础上继续细化切入的“时机”，也定义了切面切入后具体要完成的工作。根据不同的时间点可以分为五种通知类型，它们分别是前置通知（Before，在连接点之前执行的通知）、后置通知（After，在连接点之后执行的通知，无论是正常返回还是异常退出）、返回后通知（AfterReturning，在连接点之后执行的通知，仅正常返回时）、环绕通知（Around，环绕一个连接点进行的通知，可以自定义连接点前后或者异常抛出时的行为）以及异常通知（AfterThrowing，在抛出异常之后执行的通知）。

　　回想一下，我们在之前是如何实现系统日志功能的？一般情况下，都是在需要记录日志的时候，直接使用语句logger.info或logger.error来完成，因此这些日志语句就穿插于各项业务的逻辑代码中。编写的时候可能觉得这并没有什么，但是这对于一名后期维护人员就很头疼了，因为这些日志语句可能毫无规律，它们不整块出现，使用时就一条语句，还藏在大量的无关代码中，要找到它们并更新它们就变得非常困难。因此，日志管理就非常适合使用AOP去实现，在接下来的示例程序中，我将简单展示如何在Spring MVC里使用AOP去实现系统的日志管理，这里我采用的是全注解的实现方式。

　　首先是在Spring的配置文件applicationContext.xml中开启自动代理，这样Spring容器才会为使用了@Aspect注解的切面类创建代理，代码如下：
	
	<!-- 激活自动代理 -->
	<aop:aspectj-autoproxy proxy-target-class="true"/>

　　如果是maven项目，还需要在pom.xml中添加切面编程的相关jar包依赖，代码如下:

	<!-- aspectj start -->
		<dependency>
    		<groupId>org.aspectj</groupId>
    		<artifactId>aspectjrt</artifactId>
    		<version>1.8.9</version>
		</dependency>
		<dependency>
    		<groupId>org.aspectj</groupId>
    		<artifactId>aspectjweaver</artifactId>
    		<version>1.8.9</version>
		</dependency>
	<!-- aspectj end -->

　　完成前面两步我们就可以定义切面类了，在这个类里我们将逐一完成切入点、连接点以及通知的定义，从而让切面可以正确切入到我们的业务代码中。下面的代码包含注释，我将不再详细描述，只在代码后对一些需要注意的问题进行补充说明。

> *SystemLogAspect.java*

	package com.ihuyang.aspect;
	
	import org.aspectj.lang.JoinPoint;
	import org.aspectj.lang.annotation.AfterReturning;
	import org.aspectj.lang.annotation.AfterThrowing;
	import org.aspectj.lang.annotation.Aspect;
	import org.aspectj.lang.annotation.Pointcut;
	import org.springframework.beans.factory.annotation.Autowired;
	import org.springframework.stereotype.Component;
	
	import com.ihuyang.service.SystemLogService;
	
	/** 
	 * 系统日志的切面类，也是我们实现日志功能的核心类，
	 * 在这个类里我们将定义连接点、切入点以及通知
	 * @author 作者: huyang
	 * @version 创建时间：2016-10-22
	 */
	
	@Aspect //声明这是一个切面
	@Component //实例化为bean，并交由Spring容器管理
	public class SystemLogAspect {
	
		@Autowired //声明这个变量将由Spring容器注入
		SystemLogService systemLogService;
		
		//定义切入点
		@Pointcut("execution(* com.ihuyang.sevice..*.*(..)) " +
				"&& !execution(* com.ihuyang.service.SystemLogService.*(..))")
		public void serviceAspect(){
		}
		
		/**
		 * 定义连接点和通知，@AfterReturning注解表示这是“返回后通知”，即正常返回后执行的通知
		 * @param joinPoint 当前连接点的信息，后面我会介绍它的相关API
		 */
		@AfterReturning("serviceAspect") // 这里将连接点和上面定义的切入点关联起来
		public void afterReturn(JoinPoint joinPoint){
			
			/*
			 * 这个方法内是通知的具体操作，包括你日志记录的相关代码，我不再举例
			 * 这段操作会根据上面定义的切入点（指定的方法），在这些方法执行完成正常返回后切入
			 */		
		}
		
		/**
		 * 定义连接点和通知，@AfterThrowing注解表示这是“异常通知”，即抛出异常时执行的通知
		 * @param joinPoint 当前连接点的信息，后面我会介绍它的相关API
		 */
		@AfterThrowing(pointcut = "serviceAspect", throwing = "e") // 这里将连接点和上面定义的切入点关联起来
		public void doAfterThrowing(JoinPoint joinPoint, Throwable e){
			
			/*
			 * 这个方法内是通知的具体操作，包括你日志记录的相关代码，我不再举例
			 * 这段操作会根据上面定义的切入点（指定的方法），在这些方法抛出异常后切入
			 */		
		}
	}

　　这段代码中有几点需要特殊说明。首先，@Component注解用来将这个切面类实例化为bean，这样的操作也可以在Spring的配置文件中通过`<bean id="" class=""/>`来实现，效果一致。但是注解实现和配置实现两者**不能共存**，因为共存后Spring容器会同时管理两个同样的切面bean，导致的结果将是所有的通知都会执行两次，因为会有两个一样的切面被切入到业务代码中。

　　其次就是切入点的定义，它拥有一定的语法要求。首先execution()是表达式的主体，括号内是具体的范围定义，第一个“\*”的位置定义返回类型，“\*”匹配所有的返回类型。接下来的空格后就是需要匹配的包路径，后面的两个“..”代表匹配该路径下所有的子包和更深层级的包。第二个“\*”的位置定义类名，“\*”匹配当前包路径下的所有类。接下来“\*(..)”的位置定义方法名，“\*”匹配当前类下的所有方法，而括号里的“..”匹配当前方法的任意参数。现在我们可以重新看看代码中的切入点定义，“execution(* com.ihuyang.sevice..\*.\*(..))”代表切入点匹配的是com.ihuyang.sevice路径下所有类中的所有方法，而后面的“!execution(* com.ihuyang.service.SystemLogService.*(..))”代表切入点不匹配SystemLogService类下的所有方法。这里之所以需要排除掉这些方法，是因为这些方法会在我们的通知中被调用，如果它本身还可以被切入的话，就会造成切面切入的无限循环，无法退出。

　　最后，在上面的代码中，我只展示了@AfterThrowing和@AfterReturning两类通知，除此之外还有@Before（前置通知），@After（后置通知）以及@Around（环绕通知），它们的用法与@AfterReturning类似，有关定义也在上文中有所介绍。

　　通过以上代码，我们就已经实现了日志管理的功能，但在实际的开发中，我们还可以通过自定义注解的方式为我们切入的方法添加更多信息，这些信息可以在通知中获取到并记入日志。下面的ServiceLog就是我自定义的一个注解，它通过修饰方法为其添加自定义信息。

> *ServiceLog.java*

	package com.ihuyang.annoation;
	
	import java.lang.annotation.Documented;
	import java.lang.annotation.ElementType;
	import java.lang.annotation.Retention;
	import java.lang.annotation.RetentionPolicy;
	import java.lang.annotation.Target;
	import com.ihuyang.entity.MethodDescEnum;
	
	/** 
	 * 自定义注解，用于对切面切入的方法附加日志信息
	 * @author huyang
	 * @version 创建时间：2016-10-22
	 */
	@Target(ElementType.METHOD) //表示自定义注解可以用于标注方法
	@Retention(RetentionPolicy.RUNTIME) //表示自定义注解的生命周期是JVM运行期间
	@Documented //表示自定义注解会被javadoc记录，出现在文档中
	public @interface ServiceLog {
	
		/**
		 * 方法描述
		 */
		public MethodDescEnum description();
	}

　　在里面我使用到了一个叫做MethodDescEnum的枚举类型，它帮我定义了不同方法的描述信息，这些描述信息我将在自定义注解中作为参数使用。

> *MethodDescEnum.java*

	package com.ihuyang.entity;
	/** 
	 * 用于定义方法的中文描述（这些描述作为附加信息出现在日志中）
	 * @author huyang
	 * @version 创建时间：2016-10-22
	 */
	public enum MethodDescEnum {
	
		/*
		 * 这里定义了我们需要切入的三个方法的描述
		 * 我以比较常见的用户管理为例 
		 */
		CREATE_USER("创建用户"),
		UPDATE_USER("更新用户"),
		DELETE_USER("删除用户");
		
		private String description; //方法描述
		
		private MethodDescEnum(String description){
			this.description = description;
		}
		
		public String getDescription() {
			return description;
		}
		
		public void setDescription(String description) {
			this.description = description;
		}
	}

　　定义好注解过后，将注解使用到我们需要标注的方法上，这样，枚举中的方法描述会与特定的方法绑定（下面的这段代码就将CREATE_USER的信息与creatUser()方法绑定了起来），在这些方法被切入的时候我们就能够获取到这些描述信息了。

	@ServiceLog(description = MethodDescEnum.CREATE_USER)
	public void creatUser(User user){
		// 代码略
	}

　　最后，我再介绍一些通知定义中常用的API，这些API可以帮助我们获取到本次切入的详细信息。

* **获取当前切入的类和方法**

　　可以通过JoinPoint的getTarget()和getSignature()获取：

	Class targetClass = joinPoint.getTarget().getClass();
	Method method = ((MethodSignature)joinPoint.getSignature()).getMethod();

　　这里顺便提一下，当我们通过反射获取到method后，就可以通过它拿到我们自定义的注解了，代码如下:

	ServiceLog serviceLog = method.getAnnotation(ServiceLog.class);

* **获取当前切入方法的调用参数**

　　可以通过JoinPoint的getArgs()获取,返回的Object[]就是方法的参数数组，通过数组下标能够逐一访问到各个参数。

	Object[] params = joinPoint.getArgs();

<br>
<br>
<br>
<br>

>若需转载此文，请注明作者和原文地址。未经本人同意，本文不允许任何人任何形式的商业使用。如有疑问，请联系本人，联系方式详见本页末尾。
<br>
>原文地址：[胡杨的个人博客-《利用Spring AOP实现系统日志》（2016.10.22）](http://ihuyang.me/java/web/Using-Spring-AOP-to-achieve-the-system-log.html)