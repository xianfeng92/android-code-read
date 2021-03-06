# @Component

Component 管理着依赖实例，根据依赖实例之间的关系就能确定 Component 的关系。这些关系可以用 object graph描述，我称之为依赖关系图。

在 Dagger 中可以用 dependencies 来表明一个 Component 依赖 __其他 Compoent 公开的依赖__。

```
public class Son {
    @Inject
    Car car;
    @Inject
    Airplane airplane;

    public void goDongBei(){
        car.run();
        airplane.fly();
    }
}
```

用 dependencies 声明 SonComponent 依赖于 FatherComponent：

```
@Component(modules = SonModule.class,dependencies = FatherComponent.class)
public interface SonComponent {
    void inject(Son son);
}
```

在 FatherComponent 中定义可以暴露给 SonComponent 来依赖的对象：

```
@Component(modules = FatherModule.class)
public interface FatherComponent {
    void inject(Person person);
    Car car();// 暴露给 SonComponent 来依赖的对象,表明所有 dependencies = FatherComponent.class 的 Component 都可以得到 car 这个依赖
}
```

SonModule 中提供了 Son 所需的 Airplane 依赖：

```
@Module
public class SonModule {
    @Provides
    Airplane provideAirplane(){
        return new Airplane();
    }
}
```


```
Son son = new Son();
FatherComponent fatherComponent = DaggerFatherComponent.builder().build();
DaggerSonComponent.builder().fatherComponent(fatherComponent).build().inject(son);
son.goDongBei();
```

## 注入过程分析

### DaggerSonComponent 对象的构建

####1 SonComponent 对象采用 Builder 模式构建

```
  private DaggerSonComponent(Builder builder) {
    assert builder != null;
    initialize(builder);
  }
```


####2 构建 SonComponent 对象过程中，Dagger 会调用 initialize(builder) 来初始化 SonComponent 对象的一些变量

	```
	  private void initialize(final Builder builder) {

	    this.carProvider =
		new Factory<Car>() {
		  private final FatherComponent fatherComponent = builder.fatherComponent;

		  @Override
		  public Car get() {
		    return Preconditions.checkNotNull(
		        fatherComponent.car(), "Cannot return null from a non-@Nullable component method");
		  }
		};

	    this.provideAirplaneProvider = SonModule_ProvideAirplaneFactory.create(builder.sonModule);

	    this.sonMembersInjector = Son_MembersInjector.create(carProvider, provideAirplaneProvider);
	  }
	```
   
   1. 将 carProvider 初始化指向一个工厂对象（Factory<Car>），调用其 get 方法时会从  fatherComponent 中去获取 car（fatherComponent.car()）
      
      * 在构建 SonComponent 对象前，要事先构建一个 fatherComponent 对象，并传给 SonComponent。如何不传，是会报错的：

	```
	Preconditions.checkNotNull(fatherComponent.car(), "Cannot return null from a non-@Nullable component method");

	```

      * Dagger 会利用在 FatherComponent 中显示暴露的接口（Car car()），在 DaggerSonComponent 中创建对应的工厂类，用来提供 car 依赖

   2. 将 provideAirplaneProvider 变量初始化指向一个工厂对象（SonModule_ProvideAirplaneFactory），该工厂类主要提供
       sonModule，而 Airplane 依赖就在 sonModule 中提供的。

   3. 将 sonMembersInjector 变量初始化指向一个工厂对象（Son_MembersInjector），该工厂对象会存储 carProvider 和 provideAirplaneProvider 引用。
      
      * sonMembersInjector 中保存在 SonComponent 对象所需的所以依赖的引用，此处为 carProvider 和 provideAirplaneProvider


#### 注入依赖 sonComnent.inject(son)

DaggerSonComponent#inject
```
  @Override
  public void inject(Son son) {
    sonMembersInjector.injectMembers(son);
  }
```

在inject方法中，其实就是调用 sonMembersInjector.injectMembers(son)，来完成 car 和 airplane 对象的注入。

## 源码如下

#### DaggerSonComponent

```
// Generated by dagger.internal.codegen.ComponentProcessor (https://google.github.io/dagger).
package com.example.di.component;

public final class DaggerSonComponent implements SonComponent {
  private Provider<Car> carProvider;

  private Provider<Airplane> provideAirplaneProvider;

  private MembersInjector<Son> sonMembersInjector;

  private DaggerSonComponent(Builder builder) {
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {

    this.carProvider =
        new Factory<Car>() {
          private final FatherComponent fatherComponent = builder.fatherComponent;

          @Override
          public Car get() {
            return Preconditions.checkNotNull(
                fatherComponent.car(), "Cannot return null from a non-@Nullable component method");
          }
        };

    this.provideAirplaneProvider = SonModule_ProvideAirplaneFactory.create(builder.sonModule);

    this.sonMembersInjector = Son_MembersInjector.create(carProvider, provideAirplaneProvider);
  }

  @Override
  public void inject(Son son) {
    sonMembersInjector.injectMembers(son);
  }

  public static final class Builder {
    private SonModule sonModule;

    private FatherComponent fatherComponent;

    private Builder() {}

    public SonComponent build() {
      if (sonModule == null) {
        this.sonModule = new SonModule();
      }
      if (fatherComponent == null) {
        throw new IllegalStateException(FatherComponent.class.getCanonicalName() + " must be set");
      }
      return new DaggerSonComponent(this);
    }

    public Builder sonModule(SonModule sonModule) {
      this.sonModule = Preconditions.checkNotNull(sonModule);
      return this;
    }

    public Builder fatherComponent(FatherComponent fatherComponent) {
      this.fatherComponent = Preconditions.checkNotNull(fatherComponent);
      return this;
    }
  }
}

```


















