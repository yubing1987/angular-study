# Angular2 启动过程
在了解Angular2启动过程之前，我们先看一段代码：
```
import { platformBrowserDynamic } from "@angular/platform-browser-dynamic";

// The app module
import {
    NgModule,
    Component
}  from "@angular/core";
import { BrowserModule }  from "@angular/platform-browser";

@Component({
    selector: "my-app",
    template: "<div></div>"
})
export class AppComponent {}

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

platformBrowserDynamic().bootstrapModule(AppModule);
```
在这段代码中我们定义了一个AppComponent和一个AppModule类。
这段代码实现了一个最为简单的Angular2程序，它包含了Angular2最基本的几个组成部分：Module和Component。
最后我们把设计好的Module传给Angular，让它来帮我们启动整个程序。
从代码中我们可以看出，整个过程包括两大部分：
1、platformBrowserDynamic()
2、bootstrapModule(AppModule)([在这里](http://www.jianshu.com/p/dd085c38f238))
本文先研究第一步。
## platformBrowserDynamic都干了啥
要想知道platformBrowserDynamic都干了啥，我们可以查看他的代码：
```
export const platformBrowserDynamic = createPlatformFactory(
    platformCoreDynamic, 'browserDynamic', INTERNAL_BROWSER_DYNAMIC_PLATFORM_PROVIDERS);
```
可以看出这个方法是通过调用createPlatformFactory方法得到的，生成的时候会传入三个参数，其中有一个是字符串，我们看看其它两个参数都是什么：
```
export const platformCoreDynamic = createPlatformFactory(platformCore, 'coreDynamic', [
  {provide: COMPILER_OPTIONS, useValue: {}, multi: true},
  {provide: CompilerFactory, useClass: JitCompilerFactory},
  {provide: PLATFORM_INITIALIZER, useValue: _initReflector, multi: true},
]);
```
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
由之前的关系图可以知道，当我们调用platformBrowserDynamic，实际调用的是platformCore，Provider就是各个createPlatformFactory传入的第三个参数，通过这些Provider创建了一个Injector，并通过Injector创建了platform。
创建Injector的过程这里先不分析（todo:下次分析）。
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
在这段代码使用到了Injector的特性来创建的（todo:还是下次再分析）。创建完以后会获取所有名为PLATFORM_INITIALIZER的Provider，然后调用它。

至此platformBrowserDynamic()执行过程已经全部理清楚了，在这个过程中会创建Injector和platform，其中Injector是依赖注入的关键。
在分析的过程中，发现可以通过定义一个PLATFORM_INITIALIZER名称的Provider实现在初始化Platform后执行代码。示例代码：
```
function test(){
   console.log("This is a test!");
}

platformBrowserDynamic([{provide: PLATFORM_INITIALIZER, useValue: test, multi: true}]).bootstrapModule(AppModule);
```
