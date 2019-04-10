## 1. Dagger2简介

## 2. Dagger2使用

如下方式添加Dagger2依赖：

```groovy
implementation 'com.jakewharton:butterknife:8.0.0'
annotationProcessor 'com.jakewharton:butterknife-compiler:8.0.0'
```

## 3. Dagger2注解说明

### 3.1 @Inject

@Inject注解，运用的地方有两处，

1.@Inject给一个类的属性做标记时，表明它是一个依赖需求方，需要一些依赖。

2.@Inject给一个类的构造方法进行注解时，表示它提供依赖的能力。

### 3.2 @Component

@Component相当于关系纽带，将@inject标记的需求和依赖绑定起来，建立联系。

### 3.3 @Provides和@Module

Dagger2 中规定，用 @Provides 注解的依赖必须存在一个用 @Module 注解的类中。

如果一个Component没有实现任何构造方法，那么Component中Dagger2会自动创建，如果Module实现了有参的构造方法，那么它需要在Component构建的时候手动传递进去。

Component可以通过create方法创建，但是前提是，Component中的Module中被@Provides注解的方法都必须是静态方法，也就是它们必须被static修饰。

### 3.4 @Inject 和 @Provides 的优先级

一个类被 @Inject 注解了构造方法，又在某个 Module 中的 @Provides 注解的方法中提供了依赖。那么会使用@Provides提供的依赖。Dagger2依赖查找的顺序是先查找Module内所有的@Provides提供的依赖，如果查找不到再去查找@Inject提供的依赖。

### 3.5 @Singleton

用@Singleton标记在目标单例上或者Provides提供依赖的方法上，然后用@Singleton标记Component对象上。目标类就实现了此作用域下（Compoent的范围内）的单例效果。

### 3.6 @Scope

@Scope是作用域的意思，@Scope是一个元注解，@Singleton只是@Scope的一个默认实现而已。

所以可以自定义一个域@Scope的注解

```java
@Target(ANNOTATION_TYPE)
@Retention(RUNTIME)
@Documented
public @interface Scope {}
```

```java
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton {}
```

### 3.7 @Qualifiers 和 @Name

如果存在两个返回值一样类型的Provides，怎么确定依赖关系呢，就是使用@Name注解进行区分，配合@Inject和@Provides一起使用。

@Qualufie是一个元注解，Qualifiers是修饰符的意思，而@Name是被@Qualufies注解的一个注解。所以我们也可以自己定义@Qualufies代替@Name，效果相同。

### 3.8 Dagger2的延迟加载

Dagger2提供了延迟加载的能力，即只有使用到需要的依赖时才会实例依赖对象。

方法是使用Lazy，Lazy是泛型类，接受任何类型的参数。

```java
@Inject
@Named("TestLazy")
Lazy<String> name;
```

### 3.9 Provider 强制重新加载

Provider 强制重新加载

```
@Inject
Provider<Integer> randomValue;
```

应用 @Singleton 的时候，我们希望每次都是获取同一个对象，但有的时候，我们希望每次都创建一个新的实例，这种情况显然与 @Singleton 完全相反。Dagger2 通过 Provider 就可以实现。它的使用方法和 Lazy 很类似，但是，需要注意的是 Provider 所表达的重新加载是说每次重新执行 Module 相应的 @Provides 方法，如果这个方法本身每次返回同一个对象，那么每次调用 get() 的时候，对象也会是同一个。

### 3.10 Dagger2 中 Component 之间的依赖

通过dependencies添加依赖的Component

```groovy
@Component(modules = CommonModule.class, dependencies = SecondComponent.class)
```

### 3.11 Dagger2 中的 SubComponent

在 Java 软件开发中，我们经常面临的就是“组合”和“继承”的概念。它们都是为了扩展某个类的功能。 前面的 Component 的依赖采用 @Component(dependecies=othercomponent.class) 就相当于组合。 

使用 Subcomponent 时，还是要先构造 ParentComponent 对象，然后通过它提供的 SubComponent 再去进行依赖注入。

可以细细观察下它与 depedency 方法的不同之处。但是，SubComponent 同时具备了 ParentComponent 和自身的 @Scope 作用域。所以，这经常会造成混乱的地方。需要特别注意。如果要比较SubComponent 和 dependency 形式哪种更好时，各有优点，基于组合优于继承一般倾向于 dependency，因为它更灵活。