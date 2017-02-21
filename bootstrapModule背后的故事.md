bootstrapModule是PlatformRef的一个方法，这个方法中实现了Module的初始化、Component的初始化等，是Angular2的启动函数。  
阅读代码我们可以发现真正调用的是_bootstrapModuleWithZone这个方法。  
# 最开始干的事情
最开始会通过创建一个Compiler的实例，代码如下：  
```
const compilerFactory: CompilerFactory = this.injector.get(CompilerFactory);
const compiler = compilerFactory.createCompiler(
        Array.isArray(compilerOptions) ? compilerOptions : [compilerOptions]);
```
首先通过Injector得到一个得到一个Compiler的Factory（todo：具体过程这里不讲，以后单独讲）。然后通过Factory
和option创建一个compiler用于Module和Component的编译。  
Compiler的实际使用的是JitCompiler，通过JitCompilerFactory进行创建，创建的过程为：  
1、根据option创建一个Injector  
2、从创建的Injector中创建一个Compiler  
所以Compiler也是通过Injector进行创建的，具体内容将在Injector部分具体分析。  

# 编译过程
创建好Compiler后就可以对代码进行编译了：  
```
return compiler.compileModuleAsync(moduleType)
    .then((moduleFactory) => this._bootstrapModuleFactoryWithZone(moduleFactory, ngZone));
```
这段代码的第一部分就是编译我们的代码，阅读代码可以发现它的调用路径为：  
```
compileModuleAsync
         |
         +--->_compileModuleAndComponents
                       |
                       +--->_loadModules,_compileComponents,_compileModule
```
## 加载Module
在加载Module的时候，首先是通过getNgModuleMetadata方法获取Model的meta数据（[meta数据是怎么来的](./Angular中注解的实现原理.md)），在这个方法中
会得到class通过NgModule注解修饰的信息。  
```
if (meta.imports){
   ......
}
if (meta.exports){
   ......
}
if (meta.declarations){
   ......
}
if (meta.providers){
   ......
}
if (meta.entryComponents){
   ......
}
if (meta.bootstrap){
   ......
}
if (meta.schemas){
   ......
}
```
每一个判断语句内都是对各种信息的处理，最后把得到的信息组合成一个CompilerMeta。其中需要特别注意的是transitiveModule属性
这个属性记录了Module内使用到的各种信息，包括Module，Component，Provider，Pipe，Directive。  
在获取了Module的Compiler信息后就需要进行加载了，  
```
loadNgModuleDirectiveAndPipeMetadata(moduleType: any, isSync: boolean, throwIfNotFound = true):
    Promise<any> {
  const ngModule = this.getNgModuleMetadata(moduleType, throwIfNotFound);
  const loading: Promise<any>[] = [];
  if (ngModule) {
    ngModule.declaredDirectives.forEach((id) => {
      const promise = this._loadDirectiveMetadata(id.reference, isSync);
      if (promise) {
        loading.push(promise);
      }
    });
    ngModule.declaredPipes.forEach((id) => this._loadPipeMetadata(id.reference));
  }
  return Promise.all(loading);
}
```
通过代码可以看出：  
1、加载通过declared声明的directive（包括Component）  
2、加载pipe  
这两步都是对使用到的类的元数据进行处理。  
## 编译Component
加载完Module的全部信息后就要对Module中的Component进行编译，编译过程：  
1、收集Module中的全部Component  
2、收集每一个Component中使用到的entryComponents  
3、收集Module中使用到的entryComponents  
4、使用收集到的信息进行编译  
```
if (!this._compilerConfig.useJit) {
  viewClass = interpretStatements(statements, compileResult.viewClassVar);
} else {
  viewClass = jitStatements(
      `/${identifierName(template.ngModule.type)}/${identifierName(template.compType)}/${template.isHost?'host':'component'}.ngfactory.js`,
      statements, compileResult.viewClassVar);
}
```
这是调用编译的入口，具体编译过程不再这里分析，两种编译方式的区别：使用jit会在客户端(浏览器)中生成具体的代码文件，可以在浏览器查看本地文件看到，不适用jit的方式则会编译保存在内存中。  
## 编译Module
Module的编译过程和Component编译过程类似，首先进行数据收集和初始化，然后进行编译。  
```
if (!this._compilerConfig.useJit) {
  ngModuleFactory =
      interpretStatements(compileResult.statements, compileResult.ngModuleFactoryVar);
} else {
  ngModuleFactory = jitStatements(
      `/${identifierName(moduleMeta.type)}/module.ngfactory.js`, compileResult.statements,
      compileResult.ngModuleFactoryVar);
}
```
也有jit和非jit两种方式，两种方式都会返回一个ModuleFactory作为最终的编译结果。  
# 启动Module
编译完以后就会得到一个ModuleFactory，然后使用这个Factory来启动我们的程序。  
```
return compiler.compileModuleAsync(moduleType)
    .then((moduleFactory) => this._bootstrapModuleFactoryWithZone(moduleFactory, ngZone));
```
这段函数的后半部分就是进行启动，其中用到的moduleFactory就是前半部分编译后得到的。  
经过跳转后执行的代码：  
```
private _moduleDoBootstrap(moduleRef: NgModuleInjector<any>) {
  const appRef = moduleRef.injector.get(ApplicationRef);
  if (moduleRef.bootstrapFactories.length > 0) {
    moduleRef.bootstrapFactories.forEach((compFactory) => appRef.bootstrap(compFactory));
  } else if (moduleRef.instance.ngDoBootstrap) {
    moduleRef.instance.ngDoBootstrap(appRef);
  } else {
    throw new Error(
        `The module ${stringify(moduleRef.instance.constructor)} was bootstrapped, but it does not declare "@NgModule.bootstrap" components nor a "ngDoBootstrap" method. ` +
        `Please define one of these.`);
  }
}
```
有两种方式进行bootstrap：  
1、bootstrapFactories的长度大于0，使用appRef的bootstrap函数进行启动，当我们在NgModule中配置了bootstrap属性的时候，就会记录到这里来。  
2、使用moduleRef.instance的ngDoBootstrap方法进行启动，这个moduleRef.instance就是我们的NgModule修饰的类。
如果上述两种都没有的话就会报错，所以在实现作为启动Module的时候，要么注解中配置bootstrap属性，要么类中定义ngDoBootstrap方法。  
appRef实际是ApplicationRef_类的实例，bootstrap的代码为：  
```
bootstrap<C>(componentOrFactory: ComponentFactory<C>|Type<C>): ComponentRef<C> {
  if (!this._initStatus.done) {
    throw new Error(
        'Cannot bootstrap as there are still asynchronous initializers running. Bootstrap components in the `ngDoBootstrap` method of the root module.');
  }
  let componentFactory: ComponentFactory<C>;
  if (componentOrFactory instanceof ComponentFactory) {
    componentFactory = componentOrFactory;
  } else {
    componentFactory = this._componentFactoryResolver.resolveComponentFactory(componentOrFactory);
  }
  this._rootComponentTypes.push(componentFactory.componentType);
  const compRef = componentFactory.create(this._injector, [], componentFactory.selector);
  compRef.onDestroy(() => { this._unloadComponent(compRef); });
  const testability = compRef.injector.get(Testability, null);
  if (testability) {
    compRef.injector.get(TestabilityRegistry)
        .registerApplication(compRef.location.nativeElement, testability);
  }

  this._loadComponent(compRef);
  if (isDevMode()) {
    this._console.log(
        `Angular 2 is running in the development mode. Call enableProdMode() to enable the production mode.`);
  }
  return compRef;
}
```
在这段代码中主要干的事情有：  
1、获取Component Factory  
2、创建跟Component  
3、注册测测试组件  
4、加载Component  
5、显示调试信息，在开发模式下每次启动时console中显示的那段话就是在这里打印的  
加载Component的过程：  
```
private _loadComponent(componentRef: ComponentRef<any>): void {
  this.attachView(componentRef.hostView);
  this.tick();
  this._rootComponents.push(componentRef);
  // Get the listeners lazily to prevent DI cycles.
  const listeners =
      <((compRef: ComponentRef<any>) => void)[]>this._injector.get(APP_BOOTSTRAP_LISTENER, [])
          .concat(this._bootstrapListeners);
  listeners.forEach((listener) => listener(componentRef));
}
```
1、把Component的vie和application进行关联  
2、只是一个tick（todo：以后分析）  
3、添加到跟Component列表中去（可以看出Angular2是支持多个跟节点的）  
4、从Injector中获取APP_BOOTSTRAP_LISTENER名称的方法，然后调用。  

# 总结
至此Angular2的整个启动过程主线就全部分析完了。
