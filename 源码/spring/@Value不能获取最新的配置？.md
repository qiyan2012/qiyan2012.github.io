

## 前言

公司使用Apollo做配置中心，也没怎么具体了解过。只知道有配置使用时用@Value就能获取到了，而且还能自动刷新。遂一直以为@Value本身有自动刷新功能。

直到有一天用nacos做配置中心。哎？我的配置怎么不能自动刷新了？@Value出问题了吗？

### 我的controller

![img](../../img/@Value不能获取最新的配置？.assets\1684221748950-6b8a8ef5-9895-4bd1-aa05-fae5572f8dc7.png)

### 我的nacos配置

![img](../../img/@Value不能获取最新的配置？.assets\1684221406116-3f55950a-6805-4911-9260-f0040b950c4c.png)

### 修改nacos配置

把account.name改为杜甫

![img](../../img/@Value不能获取最新的配置？.assets\1684221446571-cc80636f-d623-4eca-b158-ddf66a2a3a73.png)

可以看到控制台也打印了refresh keys changed

![img](../../img/@Value不能获取最新的配置？.assets\1684221490755-7bbf550f-c7ea-4748-bee6-56d57af8715b.png)

可是调用接口得到的值仍是更新前的

![img](../../img/@Value不能获取最新的配置？.assets\1684221672969-be4384ec-c5e6-4467-8d07-2b728159bbaf.png)



后面才知道nacos要配合@RefreshScope使用才能自动刷新。Apollo的@Value自动刷新功能算是一个机制了。

于是加上@RefreshScope试验一下

![img](../../img/@Value不能获取最新的配置？.assets\1684221741201-281a0a7e-65ba-401f-85fc-283fba92ccf6.png)

先把nacos中的account.name配置改为李白，启动项目之后，再改为杜甫，然后调用接口获取account.name

![img](../../img/@Value不能获取最新的配置？.assets\1684221866920-d2fe5182-4384-404a-a468-25b072579574.png)

这次获取到的配置就是最新的了。



为什么Apollo能实现@Value的自动刷新，而nacos需要配合@RefreshScope呢？

本文旨在分析Apollo和nacos为什么能实现配置自动，和他们的实现方式有什么不同。



先简单看一下Apollo为什么能实现@Value的自动更新

## Apollo的自动更新

Apollo的自动更新比较简单

AutoUpdateConfigChangeListener#onChange

AutoUpdateConfigChangeListener是一个监听器，监听ConfigChangeEvent事件。客户端会一直向远程拉取，当配置发生变化就会发布这个事件。

![img](../../img/@Value不能获取最新的配置？.assets\1683799360787-5ed2cc2d-0af0-4941-97b3-b787de94a456.png)

可以看到通过springValueRegistry获取到了SpringValue的集合。SpringValue是Apollo维护的来更新spring中@Value值变化后的类。里面维护了@Value的占位符，bean，field等信息。通过继承BeanPostProcessor实现postProcessBeforeInitialization来维护起来的，这里不管他，只要知道他维护了@Value注释的字段和bean信息就行了。

继续看updateSpringValue

```java
private void updateSpringValue(SpringValue springValue) {
    try {
        //解析出表达式的值
        Object value = resolvePropertyValue(springValue);
        //更新
        springValue.update(value);
    
        logger.info("Auto update apollo changed value successfully, new value: {}, {}", value,
                    springValue);
    } catch (Throwable ex) {
        logger.error("Auto update apollo changed value failed, {}", springValue.toString(), ex);
    }
}
```

继续走springValue.update()方法

![img](../../img/@Value不能获取最新的配置？.assets\1683859866026-da692ca4-0a7a-4f60-bf4f-c10ea0f6213b.png)

我们这里挑一个看

![img](../../img/@Value不能获取最新的配置？.assets\1683859892940-9aae4951-a239-40b5-944c-9d37b0270762.png)

![img](../../img/@Value不能获取最新的配置？.assets\1683859967797-a96c25f0-ee1d-4c85-8cff-eddcba88d566.png)

可以看到这里先获取bean(弱引用)，然后更新这个字段的值(Field.set()方法)。

### 总结：

Apollo自己维护了@Value的信息，当有变化时，Apollo直接更新使用@Value注解bean的字段值



## nacos的自动更新

nacos的自动更新使用了spring的机制，因此比较复杂，所以nacos的自动更新机制我们详细看看。



在nacos-config的spring.factoies中,注入了NacosConfigAutoConfiguration

![img](../../img/@Value不能获取最新的配置？.assets\1683876912882-5463a7f7-7458-4932-804b-d4c7e6f0a3a4.png)

在NacosConfigAutoConfiguration里面注入了一个NacosContextRefresher的Bean

![img](../../img/@Value不能获取最新的配置？.assets\1683876959446-72ec9d40-0c5c-426d-bfb4-6d8b26fbdbc9.png)

这个Bean实现了ApplicationListener，监听ApplicationReadyEvent事件，这个事件会在spring就绪后发布，也就是SpringApplication#run()的最后。

![img](../../img/@Value不能获取最新的配置？.assets\1683877462400-2d3fe1a8-4504-49a6-972f-c89a7419bd6d.png)

![img](../../img/@Value不能获取最新的配置？.assets\1683877547757-ef99a9b9-990c-457d-9b64-944dcd1ca913.png)

![img](../../img/@Value不能获取最新的配置？.assets\1683877522369-4ae968b9-27ea-4622-86db-6d9aa115a2bf.png)

继续到registerNacosListener里，重点看一下画红圈的部分

![img](../../img/@Value不能获取最新的配置？.assets\1683877586026-26858e29-682d-4b91-b12b-1437c80ba65a-171142960560450.png)

applicationContext.publishEvent，这里发布了一个RefreshEvent事件，这个事件会由RefreshEventListener来处理，这个事件的处理我们下面说。



先来看configService.addListener。configService是nacos具体实现配置发布、更新、获取、删除的实现类，具体就是拼接url和参数调用远程的nacos服务。addListener就是添加监听器

![img](../../img/@Value不能获取最新的配置？.assets\1683878691802-c24a7e43-44f4-4334-a570-a75c12583407.png)

![img](../../img/@Value不能获取最新的配置？.assets\1683878703231-87ca2643-364e-4261-a410-6ea4117ea6d2.png)

到这里nacos添加上了配置更新的监听器，那么这个监听器是在哪触发的呢？



还是ConfigService，它的实现类NacosConfigService中的构造方法中。

![img](../../img/@Value不能获取最新的配置？.assets\1683878921557-e00cc5d1-623a-487b-a868-5cdf38e85f92.png)

在构造方法中，创建了ClientWorker。

![img](../../img/@Value不能获取最新的配置？.assets\1683879041392-dfe21cb2-b588-4f19-ac99-d3a0212d098a.png)

在ClientWorker会启动定时任务调用checkConfigInfo()。还创建了一个线程池赋值给executorService。

在checkConfigInfo()中会使用这个线程池执行任务

![img](../../img/@Value不能获取最新的配置？.assets\1683879203119-ba61febf-0c65-4e44-9f35-439ce9fb51bd.png)

可以看到提交了LongPollingRunnable这个任务，继续看一下他的run方法

这个方法有点长，只截取一部分

![img](../../img/@Value不能获取最新的配置？.assets\1683879482124-db9780d9-e666-4e0d-9b23-5dbf58acdb69-171142960560557.png)

可以看到如果	远程发现了有配置更新，就会更新缓存为最新值，checkUpdateDataIds和getServerConfig就是http请求nacos服务端获取数据，这里不展开了，感兴趣的可以自己看一下。



到此为止，我们动态更新的配置已经可以被nacos检测到了。那为什么@Value返回的值还是旧的呢。

这就要回到我们刚才略过的applicationContext.publishEvent，发布的RefreshEvent事件了。

![img](../../img/@Value不能获取最新的配置？.assets\1683877586026-26858e29-682d-4b91-b12b-1437c80ba65a-171142960560450.png)

RefreshEventListener监听RefreshEvent事件，在RefreshEventListener的handle里对事件做处理

![img](../../img/@Value不能获取最新的配置？.assets\1683880039999-fe6b03d2-e20f-4b0e-915c-4ff7f7521bcf-171142960560559.png)![img](../../img/@Value不能获取最新的配置？.assets\1684133701639-b9dbadc3-dc1b-4c3f-946a-08a76413212e-171142960560561.png)

refresh方法就是能刷新配置的重点了。refresh方法里一共调用了两个方法我们一个一个来看



先看refreshEnvironment

![img](../../img/@Value不能获取最新的配置？.assets\1684136604485-fd21af3b-9ab4-4935-ade5-033b07bd239d-171142960560563.png)

refreshEnvironment是一个更新配置的方法，可以看到是先获取到before(之前的配置)，然后用之后的配置更新配置。同样都是使用this.context.getEnvironment().getPropertySources()来获取配置，在中间调用了addConfigFilesToEnvironment()，为什么调用addConfigFilesToEnvironment之后就能获取到新的了呢？



在addConfigFilesToEnvironment中会创建一个新的spring上下文，最终会调用到SpringApplication.run()里(是不是很熟悉，又到了spring经典的run方法)。

在run()方法中有个prepareContext()，再往里调用applyInitializers。在applyInitializers里会遍历所有实现了ApplicationContextInitializer的接口(在Spring容器刷新之前执行的一个回调函数，通常用于向Spring容器中注入属性)，调用其中的initialize方法。



啊啊啊？这都哪到哪啊。这个关我自动刷新什么事？别急，这就有关联了。



PropertySourceBootstrapConfiguration(也会由spring.factories注入)，这个类就实现了ApplicationContextInitializer(也就是向spring容器注入属性的接口)。所以在prepareContext的applyInitializers方法中就会调用它的initialize方法。



在initialize中会调用PropertySourceLocator接口的locateCollection。nacos的NacosPropertySourceLocator就实现了这个接口，会向nacos获取配置，这样nacos里的配置就是最新的了。再往下他会塞到environment里，这样this.context.getEnvironment().getPropertySources()就能获取到最新的配置了。



在addConfigFilesToEnvironment做完这些工作后就会把新创建的spring上下文关闭。



看到这好像大功告成了，但其实还有个问题，配置变了，bean里通过@Value获取到的值可没变。怎么把它的值给更新呢？spring选择了非常简便的方法，弄个新的出来。

继续看refresh里调用的refreshAll()方法

![img](../../img/@Value不能获取最新的配置？.assets\1684133701639-b9dbadc3-dc1b-4c3f-946a-08a76413212e-171142960560561.png)

![img](../../img/@Value不能获取最新的配置？.assets\1683880312125-a656fd61-7bbc-4d9f-8f58-73fd492897d4-171142960560565.png)![img](../../img/@Value不能获取最新的配置？.assets\1684119419249-d24dd94a-bf87-4107-9b55-de3c09443024-171142960560567.png)

这里的this.cache是BeanLifecycleWrapperCache，里边存放BeanLifecycleWrapper。BeanLifecycleWrapper里存放了bean的名称和ObjectFactory。ObjectFactory提供了一个getObject()方法，用于获取对象实例。

![img](../../img/@Value不能获取最新的配置？.assets\1684129135307-ea06595d-59c5-4962-8075-b81e07403bfd-171142960560569.png)

那代码把cache清空，把BeanLifecycleWrapper都销毁有什么用呢？我们先去看一下BeanLifecycleWrapper是怎么放进去的。



看一下Scope的实现类GenericScope的get方法

![img](../../img/@Value不能获取最新的配置？.assets\1684129781193-6be232f0-6fd0-47f5-a7a7-3248d0021585-171142960560571.png)

get方法创建了新的BeanLifecycleWrapper放入到cache里。注意这里的cache.put()最终调用的是ConcurrentHashMap的putIfAbsent，如果已有可就不放了。返回的即是BeanLifecycleWrapper的getBean()方法。

![img](../../img/@Value不能获取最新的配置？.assets\1684130081186-038294d8-8626-4ba4-856f-aecba258cb84-171142960560573.png)

getBean就会调用ObjectFactory的getObject方法来获取对象。



那这个get方法在哪里被调用的呢，就是我们最熟悉的doGetBean

![img](../../img/@Value不能获取最新的配置？.assets\1684131696519-c6f4d20a-5ccb-45cf-af2c-9aac625a3479-171142960560575.png)

在doGetBean里会检测bean的scope，scope不是单例和原型的会走这个else里的逻辑。然后回去scope里调用get。标注了@RefreshScope注解的bean的scope为refresh，就会走这个else里的逻辑。



那逻辑就清楚了：在有值更新后，销毁scope里cache里的BeanLifecycleWrapper，当有请求到达bean的时候，doGetBean获取bean时，由于找不到原来的BeanLifecycleWrapper，又新建一个BeanLifecycleWrapper然后调用getBean,返回一个全新的bean，在新的bean里就有刷新的配置了。如果没有值更新，下次当有请求到达bean的时候，doGetBean获取bean时，BeanLifecycleWrapper仍存在，返回的bean仍是上次那个。



到这里，自动刷新才算完成了。最后我们大白话总结一下。

### 总结

nacos有任务去检查远程的配置和本地的配置是否一样，不一样的话就发布spring的事件，spring监听到后会创建一个新的上下文环境获取最新配置然后刷新配置，并且删除掉scope的缓存，使下次获取bean时重新创建一个新的bean，在新的bean中就会有刷新后的配置。