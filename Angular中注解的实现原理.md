在试用Angular进行开发的时候，我们很多时间都是在设计Component、Module。在定义一个Component的时候我们使用的是@Component进行注解声明的，如下代码：
```
@Component({
    selector: "my-app",
    template: "<div></div>"
})
export class AppComponent {}
```
这里以@开始的语句和java中的注解的使用方式和作用都非常的类似。@Component的作用就是把相应的元数据添加到AppComponent中去。那么Angular里的注解是怎么实现的呢？
# TypeScript中的装饰器
要知道Angular中注解实现的原理，首先就必须分析TypeScript中装饰器的原理，下面的代码就实现了一个简单的装饰器：
```
function f() {  //当调用C类method方法时其执行顺序是
    console.log("f(): evaluated");
    return function (target, propertyKey: string, descriptor: PropertyDescriptor) {
        console.log("f(): called");
    }
}

function g() {
    console.log("g(): evaluated");
    return function (target, propertyKey: string, descriptor: PropertyDescriptor) {
        console.log("g(): called");
    }
}

class C {
    @f()
    @g()
    method() { }
}
```
这段代码在调用C类的method方法的时候，输出的结果为：
```
f(): evaluated
g(): evaluated
g(): called
f(): called
```
其中f，g函数为装饰器工厂，他们主要作用是根据参数生成特定的装饰器（示例中没有参数，返回的也是特定的装饰器）。f，g函数内部定义的就是具体的装饰器方法，在这些方法中就可以对目标进行各种操作了。  
# Angular注解的原理
我们选用Component来分析Angular注解的原理，其它类如：Module，Input也都是类似的。  
首先我们找到Component注解的实现的源码：
```
export const Component: ComponentDecorator = <ComponentDecorator>makeDecorator(
    'Component', {
      selector: undefined,
      inputs: undefined,
      outputs: undefined,
      host: undefined,
      exportAs: undefined,
      moduleId: undefined,
      providers: undefined,
      viewProviders: undefined,
      changeDetection: ChangeDetectionStrategy.Default,
      queries: undefined,
      templateUrl: undefined,
      template: undefined,
      styleUrls: undefined,
      styles: undefined,
      animations: undefined,
      encapsulation: undefined,
      interpolation: undefined,
      entryComponents: undefined
    },
    Directive);
```
makeDecorator的源码精简后如下：
```
export function makeDecorator(
    name: string, props: {[name: string]: any}, parentClass?: any,
    chainFn: (fn: Function) => void = null): (...args: any[]) => (cls: any) => any {

    // ...

    function DecoratorFactory(objOrType: any): (cls: any) => any {

        // ...

        const TypeDecorator: TypeDecorator = <TypeDecorator>function TypeDecorator(cls: Type<any>) {

            // ...

        };

        // ...

       return TypeDecorator;
    }

    // ...

    return DecoratorFactory;
}
```
从代码我们可以发现makeDecorator返回的是一个装饰器工厂，这个装饰器工厂会返回一个装饰器TypeDecorator，从而可以知道TypeDecorator中干的就是实现注解的工作：把元数据写入Component修饰的类中去（比如：AppComponent类）。
接下来我们进一步分析TypeDecorator的代码：
```
const TypeDecorator: TypeDecorator = <TypeDecorator>function TypeDecorator(cls: Type<any>) {
  const annotations = Reflect.getOwnMetadata('annotations', cls) || [];
  annotations.push(annotationInstance);
  Reflect.defineMetadata('annotations', annotations, cls);
  return cls;
};
```
这里试用到了一个第三方的库Reflect，其具体作用就是操作目标类上的元数据。TypeDecorator方法内容非常的简单，把@Component传入的参数作为元数据添加到目标类中去。  
最后我们可以总结整个注解的实现过程：  
1、试用makeDecorator生成装饰器工厂。
2、目标类中使用装饰器工厂，并传入相应的参数。
3、自动调用装饰器方法，把对应的元数据写入annotations属性中去。
# 动态修改注解
根据源码我们可以知道注解的元数据都是存放在annotations属性中的，我们可以动态的获取并修改他们。具体事例如下：
```
@Component({
    selector: "my-app",
    template: "<div></div>"
})
export class AppComponent {}

const annotations = Reflect.getOwnMetadata('annotations', AppComponent);
annotations[0].template = "<div>This is test!</div>"
```
这段代码就可以修改AppComponent中的template元数据的内容，达到动态修改的效果。但是要注意任何元数据的修改，改变的只是元数据本身，任何以元数据为基础产生的内容不会修改，比如事例中的代码如果在AppComponent实例化后的页面内容是没有作用的。这种动态修改的Component只能影响修改后动态创建的组件。
