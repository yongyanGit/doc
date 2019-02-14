### 负载均衡策略

IRule接口的各个实现如下：

![rule](../images/springcloud/ribbon/rule.png)

#### AbstractLoadBalancerRule

负载均衡策略的抽象类，在该抽象类中定义了负载均衡器ILoadBalancer对象，该对象能够在具体实现选择服务策略时，获取到一些负载均衡器中维护的信息来作为分配依据，并以此设计一些算法来实现针对特定场景的高效策略。

```java
public abstract class AbstractLoadBalancerRule implements IRule, IClientConfigAware {

    private ILoadBalancer lb;
        
    @Override
    public void setLoadBalancer(ILoadBalancer lb){
        this.lb = lb;
    }
    
    @Override
    public ILoadBalancer getLoadBalancer(){
        return lb;
    }      
}
```

### RandomRule

该策略实现了从服务实例清单中随机选择一个服务实例的功能。

```java
public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        Server server = null;

        while (server == null) {
            //如果线程被中断，返回null
            if (Thread.interrupted()) {
                return null;
            }
            //可用的服务
            List<Server> upList = lb.getReachableServers();
            //所有的服务
            List<Server> allList = lb.getAllServers();
			//所有实例服务的数量
            int serverCount = allList.size();
            //没有服务，则返回空
            if (serverCount == 0) {
                return null;
            }
			//获取一个随机数
            int index = rand.nextInt(serverCount);
            //根据下标获取服务
            server = upList.get(index);

            if (server == null) {
               
                Thread.yield();
                continue;
            }

            if (server.isAlive()) {
                return (server);
            }

            // Shouldn't actually happen.. but must be transient or a bug.
            server = null;
            Thread.yield();
        }

        return server;

    }
```

