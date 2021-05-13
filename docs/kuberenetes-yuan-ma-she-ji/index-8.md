# Core Cacheable Object

## Overview

从命名来看，CacheableObject 用于保存 Object 实例，在 Kubernetes 版本 1.18 中，只要一个结构 cachingObject 实现了该接口，关系大致如下图  
![cacheable-object-overview.svg](../.gitbook/assets/1%20%284%29.jpeg)  
再结合 CacheableObject 定义，会发现：向 CacheableObject 存入 Object 时，指定了 Identifier，那是否意味着可以存入多个 Object 呢？而 GetObject 方法，确只返回了一个 Object 实例，虽然不能排除 slice、map 等容器类结构实现 Object 接口的可能性，但也是一个需要深入了解并解决的问题。让我们继续，并从尝试解决这两个问题开始，看看能有什么收获。  
![image.png](../.gitbook/assets/2%20%282%29.jpeg)  


## cachingObject

### Overview

每个 cachingObject 实际存储一个 Object，即 metaRuntimeInterface 实例。根据不同的 Identifier，将这个对象编码为不同格式，并缓存在 map 中。  
![cacheable-object-caching-object.svg](../.gitbook/assets/3%20%282%29.jpeg) 

### Implementation

#### Definition

![image.png](../.gitbook/assets/4%20%282%29.jpeg)  
metaRuntimeInterface 同时实现了 runtime.Object 与 metav1.Object 接口  
![image.png](../.gitbook/assets/5%20%281%29.jpeg)  
metav1.Object 接口定义如下，用于描述 Kubernetes 核心对象  
![image.png](../.gitbook/assets/6%20%281%29.jpeg)  


#### Creation

从 new 方法，可以看到，一个 cachingObject 存入的确实是一个实例，并且存入实例时，存储的并不是原始对象，而是一份儿深度拷贝。  
![image.png](../.gitbook/assets/7%20%281%29.jpeg)  
GetObject 方法获取存入的 metaRuntimeInterface 实例的一份儿深度拷贝。  
![image.png](../.gitbook/assets/8%20%281%29.jpeg) 

#### Implementation for runtime.Object

cachingObject 本身也满足 runtime.Object 接口，实现如下。这里需要注意，在 DeepCopyObject 方法中，创建了新的 serializationCache，且没有复制旧内容。  
![image.png](../.gitbook/assets/9%20%281%29.jpeg)  


#### Implementation For CacheableObject

![image.png](../.gitbook/assets/10%20%281%29.jpeg)  
重点是更换 atomic.Value 操作过程，如下所示  
![image.png](../.gitbook/assets/11%20%281%29.jpeg)  


