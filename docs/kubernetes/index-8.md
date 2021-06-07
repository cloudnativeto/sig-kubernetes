# Core Cacheable Object

## Overview

Know the information from the name of `CacheableObject`. It could store the Object instance. In the 1.18 version of Kubernetes,  `cachingObject` is the only implementation of this interface - `CacheableObject`. Its relation as this picture.   


![cacheable-object-overview.svg](../.gitbook/assets/1%20%2814%29.jpeg)

  
Combined with `CacheableObject` definitions, it will be found that when the `CacheableObject` is stored in Object, Identifier is specified, whether does it mean to save\(cache\) multiple Object? 

The `GetObject()` method is indeed returned to an Object instance, although the container class structure of Slice, Map cannot be eliminated to implement the possibility of Object interface, it is also a problem that needs to be deeply understood and resolved. 

Let us continue and start by trying to solve these two questions and see what can have any gains.  


![image.png](../.gitbook/assets/2%20%289%29.jpeg)

  


## cachingObject

### Overview

Each `cachingObject` actually stores an Object, which is the `metaRuntimeInterface` instance. According to different Identifiers, this object is encoded into different formats and is cached in the map.  
 

![cacheable-object-caching-object.svg](../.gitbook/assets/3%20%288%29.jpeg)



### Implementation

#### Definition

![image.png](../.gitbook/assets/4%20%288%29.jpeg)

`metaRuntimeInterface` simultaneously implemented `runtime.Object` interface and `metav1.Object` interface.  


![image.png](../.gitbook/assets/5%20%283%29.jpeg)

  
`metav1.Object` interface as this defined is using to describe Kuberentes core Object.  


![image.png](../.gitbook/assets/6%20%282%29.jpeg)

  


#### Creation

From the new method, we can see that a `cachingObject` stores an instance, and when it stores an instance, it does not store the original object, but a deep copy

![image.png](../.gitbook/assets/7%20%282%29.jpeg)

 GetObject method Gets a deep copy of the metaRuntimeInterface instance.

![image.png](../.gitbook/assets/8%20%283%29.jpeg)



#### Implementation for runtime.Object

CachingObject itself also implement the `runtime.Object` interface, implemented as follows. It is important to note that in the `DeepCopyObject()` method, a new SerializationCache is created, and the old content is **not copied**.  


![image.png](../.gitbook/assets/9%20%281%29.jpeg)

  


#### Implementation For CacheableObject

![image.png](../.gitbook/assets/10%20%281%29.jpeg)

The focus is to replace the `atomic.Value` , the operation as shown below.

![image.png](../.gitbook/assets/11%20%281%29.jpeg)

  


