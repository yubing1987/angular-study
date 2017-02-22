在分析bootstrapModule方法时，我们发现当需要一个类的事例的时候只需要调用injector.get方法就能够得到。这个Injector就是Angular依赖注入的基础，任何可以被注入的类都是都可以通过injector.get方法来获取。  
调用injector.get方法的时候会出现一下几种情况：  
1、如果以前没有创建过，就会new一个事例，并返回。  
2、如果以前创建过，就返回以前创建的那个事例，相当于单例模式。  

# Injector.get方法的调用过程
Injector是一个接口，实际实例化的类为ReflectiveInjector_。继承关系为  
```
ReflectiveInjector_ -> ReflectiveInjector -> Injector。  
```
查看ReflectiveInjector_.get方法的代码，我们可以发现其调用关系为：  
```
get -> _getByKey -> _getByKeyDefault -> _getObjByKeyId
```
如果调用自己的_getObjByKeyId没有找到实例，就会查找父Injector的_getObjByKeyId，一直向上查找。  
在_getObjByKeyId如果具有声明但是没有事例化，就会去实例化并返回，具体代码如下：  
```
private _getObjByKeyId(keyId: number): any {
  for (let i = 0; i < this.keyIds.length; i++) {
    if (this.keyIds[i] === keyId) {
      if (this.objs[i] === UNDEFINED) {
        this.objs[i] = this._new(this._providers[i]);
      }

      return this.objs[i];
    }
  }
  return UNDEFINED;
}
```
# 实例化的过程
实例化是通过调用_new方法来完成的，具体代码为：
```
_new(provider: ResolvedReflectiveProvider): any {
  if (this._constructionCounter++ > this._getMaxNumberOfObjects()) {
    throw cyclicDependencyError(this, provider.key);
  }
  return this._instantiateProvider(provider);
}
```
在这段代码中具有一个防止循环依赖的检验。  
接下来就是使用provider中的创建工厂来创建实例了，具体代码为：
```
private _instantiate(
    provider: ResolvedReflectiveProvider,
    ResolvedReflectiveFactory: ResolvedReflectiveFactory): any {
  const factory = ResolvedReflectiveFactory.factory;

  let deps: any[];
  try {
    deps =
        ResolvedReflectiveFactory.dependencies.map(dep => this._getByReflectiveDependency(dep));
  } catch (e) {
    if (e.addKey) {
      e.addKey(this, provider.key);
    }
    throw e;
  }

  let obj: any;
  try {
    obj = factory(...deps);
  } catch (e) {
    throw instantiationError(this, e, e.stack, provider.key);
  }

  return obj;
}
```
其中_getByReflectiveDependency方法会去查找依赖的事例，如果依赖的事例没有被创建，就会先去创建，这里就会有可能造成循环依赖。  
# 什么是Provider
Provider可以理解为是Angular对于依赖注入项的一个描述，一个对象可以被依赖注入，就必须有一个Provider对其进行描述。Injector根据这个Provider才能够创建实例。  
Provider源码：  
```
export interface TypeProvider extends Type<any> {}

export interface ValueProvider {

  provide: any;

  useValue: any;

  multi?: boolean;
}

export interface ClassProvider {

  provide: any;

  useClass: Type<any>;

  multi?: boolean;
}

export interface ExistingProvider {

  provide: any;

  useExisting: any;

  multi?: boolean;
}

export interface FactoryProvider {

  provide: any;

  useFactory: Function;

  deps?: any[];

  multi?: boolean;
}

export type Provider =
    TypeProvider | ValueProvider | ClassProvider | ExistingProvider | FactoryProvider | any[];
```
从源码我们可以知道总共可以有5中类型的Provider以及一个数组类型的Provider。对于不同类型的Provider也有各自的定义和用法。  
|类型|说明|  
|---|---|  
|TypeProvider|任意类型的类，任意一个类都可以成为一个Provider，Angular会对其进行包装，Factory就是new一个这个类型的事例，都是单例的|  
|ValueProvider|使用具体的值作为实例的|  
|ClassProvider|使用具体class来创建事例，和TypeProvider类似，但是可以设置为multi|  
|ExistingProvider|使用一个已有的Provider，相当于给已有的Provider设置一个别名|  
|FactoryProvider|使用自己的创建工厂来实例化Provider，可以设置创建的依赖项|  
Injector在创建一个事例的时候就是使用Provider的Factory和deps来实例化（具体如上面的代码所示）。
# 每一种Provider的Factory是怎么来的
从上面的分析我们可以发现，这么多种Provider中只有FactoryProvider是需要设置Factory和deps的，然而在Injector创建实例的时候是必须需要Factory的，那么其它类型的Provider的Factory又是怎么来的呢？  
通过跟踪代码，我们可以发现如下代码：
```
function resolveReflectiveFactory(provider: NormalizedProvider): ResolvedReflectiveFactory {
  let factoryFn: Function;
  let resolvedDeps: ReflectiveDependency[];
  if (provider.useClass) {
    const useClass = resolveForwardRef(provider.useClass);
    factoryFn = reflector.factory(useClass);
    resolvedDeps = _dependenciesFor(useClass);
  } else if (provider.useExisting) {
    factoryFn = (aliasInstance: any) => aliasInstance;
    resolvedDeps = [ReflectiveDependency.fromKey(ReflectiveKey.get(provider.useExisting))];
  } else if (provider.useFactory) {
    factoryFn = provider.useFactory;
    resolvedDeps = constructDependencies(provider.useFactory, provider.deps);
  } else {
    factoryFn = () => provider.useValue;
    resolvedDeps = _EMPTY_LIST;
  }
  return new ResolvedReflectiveFactory(factoryFn, resolvedDeps);
}
```
从这段代码可以发现，Angular我们定义的Provider从新计算了他们的Factory和deps。  
# Inectory自身的创建过程
通过调用ReflectiveInjector.resolveAndCreate方法可以创建一个Injector，这个方法的第一个参数就是我们需要使用到的Provider列表，第二个参数就是父Injector。  
```
static resolveAndCreate(providers: Provider[], parent: Injector = null): ReflectiveInjector {
  const ResolvedReflectiveProviders = ReflectiveInjector.resolve(providers);
  return ReflectiveInjector.fromResolvedProviders(ResolvedReflectiveProviders, parent);
}
```
这个方法里面主要完成两个动作：  
1、格式化Provider。  
2、new一个Injector事例。  
第二步比较清晰，下面主要分析第一步。  
第二步主要的代码：
```
export function resolveReflectiveProviders(providers: Provider[]): ResolvedReflectiveProvider[] {
  const normalized = _normalizeProviders(providers, []);
  const resolved = normalized.map(resolveReflectiveProvider);
  const resolvedProviderMap = mergeResolvedReflectiveProviders(resolved, new Map());
  return Array.from(resolvedProviderMap.values());
}
```
这里主要做的事情有：  
1、把provider列表理顺，如：[[p1,p2],p3,[p4,p5]]理顺为[p1,p2,p3,p4,p5]  
2、封装provider，就是上面添加Factory和deps的内容。
3、合并相同的Provider，如果不支持multi的Provider声明的了多个，就会抛出异常。
