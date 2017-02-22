# 一个最简单的Angular程序
在了解Angular2启动过程之前，我们先看一段代码：
```
import { platformBrowserDynamic } from "@angular/platform-browser-dynamic";
import { NgModule, Component}  from "@angular/core";
import { BrowserModule }  from "@angular/platform-browser";

// 定义Component
@Component({
    selector: "my-app",
    template: "<div></div>"
})
export class AppComponent {}

// 定义Module
@NgModule({
    imports: [
        BrowserModule
    ],
    declarations: [
        AppComponent
    ],
    bootstrap: [ AppComponent ]
})
export class AppModule { }

// 启动Angular程序
platformBrowserDynamic().bootstrapModule(AppModule);
```
这一段代码主要有四个部分组成：  
1、引用Angular相关的类  
2、定义Component（什么是Component（todo：下次分析），@Component是什么（[在这里](./Angular中注解的实现原理.md)））    
3、定义了一个Module（什么是Module（todo：下次分析），@Module是什么（[在这里](./Angular中注解的实现原理.md)））  
4、调用Angular的启动函数，实现启动功能。  
从代码我们可以发现，启动过程实际上是分成了两大步骤的。  
1、调用platformBrowserDynamic()方法  
2、调用bootstrapModule(AppModule)方法([在这里](./bootstrapModule背后的故事.md))
# platformBrowserDynamic都干了什么
要想知道platformBrowserDynamic都干了啥，我们可以查看他的代码：
```
export const platformBrowserDynamic = createPlatformFactory(
    platformCoreDynamic, 'browserDynamic', INTERNAL_BROWSER_DYNAMIC_PLATFORM_PROVIDERS);
```
可以看出这个方法是通过调用createPlatformFactory方法得到的，生成的时候会传入三个参数，其中有第二个参数是字符串，我们看看其它两个参数都是什么。  
第一个参数：
```
export const platformCoreDynamic = createPlatformFactory(platformCore, 'coreDynamic', [
  {provide: COMPILER_OPTIONS, useValue: {}, multi: true},
  {provide: CompilerFactory, useClass: JitCompilerFactory},
  {provide: PLATFORM_INITIALIZER, useValue: _initReflector, multi: true},
]);
```
第三个参数：
```
export const INTERNAL_BROWSER_DYNAMIC_PLATFORM_PROVIDERS: Provider[] = [
  INTERNAL_BROWSER_PLATFORM_PROVIDERS,
  {
    provide: COMPILER_OPTIONS,
    useValue: {providers: [{provide: ResourceLoader, useClass: ResourceLoaderImpl}]},
    multi: true
  },
];
```
发现platformCoreDynamic也是通过createPlatformFactory方法得到的，调用的时候又引入了一个新的参数：platformCore，它初始化的代码为：
```
function _reflector(): Reflector {
  return reflector;
}

const _CORE_PLATFORM_PROVIDERS: Provider[] = [
  PlatformRef_,
  {provide: PlatformRef, useExisting: PlatformRef_},
  {provide: Reflector, useFactory: _reflector, deps: []},
  {provide: ReflectorReader, useExisting: Reflector},
  TestabilityRegistry,
  Console,
];

/**
 * This platform has to be included in any other platform
 *
 * @experimental
 */
export const platformCore = createPlatformFactory(null, 'core', _CORE_PLATFORM_PROVIDERS);
```
可以看出platformCore初始化的时候也是调用的createPlatformFactory。
至此我们可以知道platformBrowserDynamic的关系图：
```
platformBrowserDynamic
     |
     +---> platformCoreDynamic
                |
                +---> platformCore
```
接下来我们分析最为关键的createPlatformFactory方法。
```
export function createPlatformFactory(
    parentPlaformFactory: (extraProviders?: Provider[]) => PlatformRef, name: string,
    providers: Provider[] = []): (extraProviders?: Provider[]) => PlatformRef {
  const marker = new OpaqueToken(`Platform: ${name}`);
  return (extraProviders: Provider[] = []) => {
    if (!getPlatform()) {
      if (parentPlaformFactory) {
        parentPlaformFactory(
            providers.concat(extraProviders).concat({provide: marker, useValue: true}));
      } else {
        createPlatform(ReflectiveInjector.resolveAndCreate(
            providers.concat(extraProviders).concat({provide: marker, useValue: true})));
      }
    }
    return assertPlatform(marker);
  };
}
```
可以看出在createPlatformFactory方法中，只是简单的返回了一个内部方法。当我们调用platformBrowserDynamic()时，实际就是调用这个内部方法。其实这个内部方法也比较简单，主要逻辑为：
1、判断是否已经创建过了。
2、判断是否有父Factory。
3、如果有父Factory就把调用Factory时传入的Provider和调用createPlatformFactory传入的Provider合并，然后调用父Factory。
4、如果没有父Factory，先创建一个Injector，然后去创建platform。
由之前的关系图可以知道，当我们调用platformBrowserDynamic，实际调用的是platformCore，Provider就是各个createPlatformFactory传入的第三个参数，通过这些Provider创建了一个Injector，并通过Injector创建了platform（创建过程[看这里](./Injector的实现原理.md)）。
我们在看看创建Platform的过程：
```
export function createPlatform(injector: Injector): PlatformRef {
  if (_platform && !_platform.destroyed) {
    throw new Error(
        'There can be only one platform. Destroy the previous one to create a new one.');
  }
  _platform = injector.get(PlatformRef);
  const inits: Function[] = <Function[]>injector.get(PLATFORM_INITIALIZER, null);
  if (inits) inits.forEach(init => init());
  return _platform;
}
```
在这段代码使用到了Injector的特性来创建的（[看这里](./Injector的实现原理.md)）。创建完以后会获取所有名为PLATFORM_INITIALIZER的Provider，然后调用它。

至此platformBrowserDynamic()执行过程已经全部理清楚了，在这个过程中会创建Injector和platform，其中Injector是依赖注入的关键(Injector依赖注入的关键[看这里](./Injector的实现原理.md))。
在分析的过程中，发现可以通过定义一个PLATFORM_INITIALIZER名称的Provider实现在初始化Platform后执行代码。示例代码：
```
function test(){
   console.log("This is a test!");
}

platformBrowserDynamic([{provide: PLATFORM_INITIALIZER, useValue: test, multi: true}]).bootstrapModule(AppModule);
```
