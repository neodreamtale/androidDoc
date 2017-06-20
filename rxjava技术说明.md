Rxjava2.0使用小结
===========

RxJava正在Android开发者中变的越来越流行。唯一的问题就是上手不容易，尤其是大部分人之前都是使用命令式编程语言。但是一旦你弄明白了，你就会发现RxJava真是太棒了。

##开始使用Rxjava2.0

### 导入项目 ####

gradle:

	<compile 'io.reactivex.rxjava2:rxjava:2.0.3'>

maven:

	<dependency>
    <groupId>io.reactivex.rxjava2</groupId>
    <artifactId>rxjava</artifactId>
    <version>2.0.3</version>
</dependency>

####基础

RxJava最核心的两个东西是Observables（被观察者，事件源）和Subscribers（观察者）。Observables发出一系列事件，Subscribers处理这些事件。这里的事件可以是任何你感兴趣的东西（触摸事件，web接口调用返回的数据。。。）

一个Observable可以发出零个或者多个事件，知道结束或者出错。每发出一个事件，就会调用它的Subscriber的onNext方法，最后调用Subscriber.onNext()或者Subscriber.onError()结束。


Rxjava的看起来很想设计模式中的观察者模式，但是有一点明显不同，那就是如果一个Observerble没有任何的的Subscriber，那么这个Observable是不会发出任何事件的。

* 简单示例

我们知道一个简单的RxJava的应用，需要一个观察者或者订阅者Observer，一个被观察者Observable，最后调用subscribe()方法将两者绑定起来！


	//观察者模式,这里产生事件,事件产生后发送给接受者,但是一定要记得将事件的产生者和接收者捆绑在一起,否则会出现错误
	Observable.create(new ObservableOnSubscribe<String>() {
    	@Override
    	public void subscribe(ObservableEmitter<String> e) throws Exception {
        	//这里调用的方法会在产生事件之后会发送给接收者,接收者对应方法会收到
        	e.onNext("hahaha");
        	e.onError(new Exception("wulala"));
        	e.onComplete();
    	}/
	}).subscribe(new Observer<String>() {
    //接受者,根据事件产生者产生的事件调用不同方法
	    	@Override
	    	public void onSubscribe(Disposable d) {
	    	    Log.e(TAG, "onSubscribe: ");
	    	}
	
		    @Override
		    public void onNext(String value) {
		        Log.e(TAG, "onNext: " + value);
		    }
	
		    @Override
		    public void onError(Throwable e) {
		        Log.e(TAG, "onError: ", e);
		    }
		
		    @Override
		    public void onComplete() {
		        Log.e(TAG, "onComplete: ");
		    }
		});


![如图](http://upload-images.jianshu.io/upload_images/3026506-457d59a1b77325f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/740)

ObservableEmitter,这是个啥东西?Emitter:顾名思义,即Rxjava的发射器,通过这个发射器,即可发送事件-----通过调用onNext,onError,onComplete方法发送不同事件.

注意:
虽然RxJava可以进行事件发送,但这并不意味着你可以随便发送,这其中需要遵循一些规则.


1. onNext:你可以发送无数个onNext,发送的每个onNext接受者都会接收到.


2. onError:当发送了onError事件之后,发送者onError之后的事件依旧会继续发送,但是接收者当接收到onError之后就会停止接收事件了.


3. onComplete:当发送了onComplete事件之后,发送者的onComplete之后的事件依旧会继续发送,但是接收者接收到onComplete之后就停止接收事件了.


4. onError事件和onComplete事件是互斥的,但是这并不代表你配置了多个onError和onComplete一定会崩溃,多个onComplete是可以正常运行的,但是只会接收到第一个,之后的就不会再接收到了,多个onError时,只会接收到第一个,第二个会直接造成程序崩溃.

5. subscribe会返回一个Disposable对象，这个对象会控制整体的观察-消费行为。百度告诉我这东西叫做一次性的,是用来控制发送者和接受者之间的纽带的,默认为false,表示发送者和接受者直接的通信阀门关闭,可以正常通信,在调用dispose()方法之后,阀门开启,会阻断发送者和接收者之间的通信,从而断开连接.

6. 默认情况下,发送者和接收者都运行在主线程,但是这显然是不符合实际需求的,我们在日常使用中,通常用的最多的就是在子线程进行各种耗时操作,然后发送到主线程进行,难道我们就没有办法继续用这个优秀的库了?想多了你,一个优秀的库如果连这都想不到,怎么能被称为优秀呢,RxJava中有线程调度器,通过线程调度器,我们可以很简单的实现这种效果,下面放代码.

__示例__

	Observable.create(new ObservableOnSubscribe<String>() {
	    @Override
	    public void subscribe(ObservableEmitter<String> e) throws Exception {
	        e.onNext("hahaha");
	        e.onNext("hahaha");
	        e.onNext("hahaha");
	        Log.e(TAG,"运行在什么线程" + Thread.currentThread().getName());
	        e.onComplete();
	    }
	}).subscribeOn(Schedulers.newThread())               //线程调度器,将发送者运行在子线程
	  .observeOn(AndroidSchedulers.mainThread())          //接受者运行在主线程
	  .subscribe(new Observer<String>() {
	    @Override
	    public void onSubscribe(Disposable d) {
	        Log.e(TAG, "onSubscribe: ");
	        Log.e(TAG,"接收在什么线程" + Thread.currentThread().getName());
	    }
	
	    @Override
	    public void onNext(String value) {
	        Log.e(TAG, "onNext: " + value);
	    }
	
	    @Override
	    public void onError(Throwable e) {
	        Log.e(TAG, "onError: ", e);
	    }
	
	    @Override
	    public void onComplete() {
	        Log.e(TAG, "onComplete: ");
	    }
	});

RxJava线程池中的几个线程选项

	- Schedulers.io()      io操作的线程, 通常io操作,如文件读写.
	- Schedulers.computation()      计算线程,适合高计算,数据量高的操作.
	- Schedulers.newThread()      创建一个新线程,适合子线程操作.
	- AndroidSchedulers.mainThread()      Android的主线程,主线程


#### Rxjava 进阶-----操作符

RxJava的强大之处，在于它提供了非常丰富且功能强悍的操作符，通过使用和组合这些操作符，你几乎能完成所有你想要完成的任务

举个例子页面跳转原先这么写：
	
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_welcome);

        ImageView view = (ImageView) findViewById(R.id.iv_welcome);
        view.setImageResource(R.drawable.welcome);
        Handler handler = new Handler();
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                startActivity(new Intent(WelcomeActivity.this, MainActivity.class));
                finish();
            }
        },2000);
    }


举个例子页面跳转现在可以这么写：

	protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_welcome);

        ImageView view = (ImageView) findViewById(R.id.iv_welcome);
        view.setImageResource(R.drawable.welcome);
        Observable.timer(2, TimeUnit.SECONDS, AndroidSchedulers.mainThread()).map(l->{
            startActivity(new Intent(this, MainActivity.class));
            finish();
            return null;
        }).subscribe();
    }

带来这种极大语言丰富性的元素，就是操作符。比如上面这个例子里的`timer`

----------------

#### 操作符分类
#####Create one Observerable

如果你要__创建__一个observerable，有如下的操作符可以创建

* Create — create an Observable from scratch by calling observer methods programmatically
>最手动的方式创建一个Observable，需要自己写一些代码的方式

* Defer — do not create the Observable until the observer subscribes, and create a fresh Observable for each observer
>推迟一个Observable的创建到被订阅的时刻。有什么意义呢？看下下面的对比

	public class SomeType {  
		private String value;
	
		public void setValue(String value) {
		  this.value = value;
		}
	
		public Observable<String> valueObservable() {
		  return Observable.just(value);
		}
	}
这段代码在运行后会打印出什么呢？

	SomeType instance = new SomeType();  
	Observable<String> value = instance.valueObservable();  
	instance.setValue("Some Value");  
	value.subscribe(System.out::println);

答案是null 因为valueObservable在给Value赋值之前就生成好了,如何改变呢？使用defer

	public Observable<String> valueObservable() {
	  return Observable.defer(new Func0<Observable<String>>() {
	    @Override 
		public Observable<String> call() {
	      return Observable.just(value);
	    }
	  });
	}

* Empty/Never/Throw — create Observables that have very precise and limited behavior
>三个特殊的Observable 可以当做参数传递，他们本身不发射数据


* From — convert some other object or data structure into an Observable
>把参数转为Observable对象。RxJava中，from操作符可以转换Future、Iterable和数组。对于Iterable和数组，产生的Observable会发射Iterable或数组的每一项数据。对于Future，它会发射Future.get()方法返回的单个数据。from方法有一个可接受两个可选参数的版本，分别指定超时时长和时间单位。如果过了指定的时长Future还没有返回一个值，这个Observable会发射错误通知并终止


* Interval — create an Observable that emits a sequence of integers spaced by a particular time interval
>这个方法接受一个表示时间间隔参数和一个表示时间的参数，返回的Observable 按照固定的时间间隔发射一个无限递增的整数序列。


* Just — convert an object or a set of objects into an Observable that emits that or those objects
>这个方法将单个数据转换为发射那个数据的Observable，From操作符会将数组中的每个项单独发射，而Just等于将这个数组作为一个事件发射出去。Just方法接受一至9个参数，返回一个按参数列表顺序发射这些数据的Obserable.


* Range — create an Observable that emits a range of sequential integers
>创建一个发射指定范围内的有序整数序列的Obserable


* Repeat — create an Observable that emits a particular item or sequence of items repeatedly
>RxJava将这个操作符实现为repeat方法。它不是创建一个Observable，而是重复发射原始Observable的数据序列，这个序列或者是无限的，或者通过repeat(n)指定重复次数。
repeat操作符默认在trampoline调度器上执行。有一个变体可以通过可选参数指定Scheduler。


* Start — create an Observable that emits the return value of a function
>编程语言有很多种方法可以从运算结果中获取值，它们的名字一般叫functions, futures, actions, callables, runnables等等。在Start目录下的这组操作符可以让它们表现得像Observable，因此它们可以在Observables调用链中与其它Observable搭配使用。
Start操作符的多种RxJava实现都属于可选的rxjava-async模块。
rxjava-async模块包含start操作符，它接受一个函数作为参数，调用这个函数获取一个值，然后返回一个会发射这个值给后续观察者的Observable。

注意：这个函数只会被执行一次，即使多个观察者订阅这个返回的Observable。
* Timer — create an Observable that emits a single item after a given delay
>操作符创建一个在给定的时间段之后返回一个特殊值的Observable。


#####Transforming one Observerable

* __map( )__ — 对序列的每一项都应用一个函数来变换Observable发射的数据序列.
Map操作符对原始Observable发射的每一项数据应用一个你选择的函数，然后返回一个发射这些结果的Observable。

![map](http://reactivex.io/documentation/operators/images/map.png)

例如

	Observable<Integer> observable = Observable.just("hello")
				.map(new Function<String, Integer>() {
		            @Override
		            public Integer apply(String s) throws Exception {
		                return s.length();
				}});
<br><br><br>
* __flatMap( ), concatMap( ), and flatMapIterable( )__ — 将Observable发射的数据集合变换为Observables集合，然后将这些Observable发射的数据平坦化的放进一个单独的Observable。操作符使用一个指定的函数对原始Observable发射的每一项数据执行变换操作，这个函数返回一个本身也发射数据的Observable，然后FlatMap合并这些Observables发射的数据，最后将合并后的结果当做它自己的数据序列发射。

![](http://reactivex.io/documentation/operators/images/flatMap.c.png)

__注意:__FlatMap对这些Observables发射的数据做的是合并(merge)操作，因此它们可能是交错的。

在许多语言特定的实现中，还有一个操作符不会让变换后的Observables发射的数据交错，它按照严格的顺序发射这些数据，这个操作符通常被叫作ConcatMap或者类似的名字。

例如

	ArrayList<String[]> list=new ArrayList<>();
	String[] words1={"Hello,","I am","China!"};
	String[] words2={"Hello,","I am","Beijing!"};
	list.add(words1);
	list.add(words2);
	Flowable.fromIterable(list)
    .flatMap(new Function<String[], Publisher<String>>() {
        @Override
        public Publisher<String> apply(String[] strings) throws Exception {
            return Flowable.fromArray(strings);
        }
    })
    .subscribe(new Consumer<String>() {
        @Override
        public void accept(String s) throws Exception {
            Log.e("consumer", s);
        }
    });

<br><br><br>
* __switchMap( )__ — 类似于flatmap那样生成observable，并将Observable发射的数据集合变换为Observables集合，然后只发射这些Observables最近发射的数据
>简单来说就是在生成Observable如果是多线程情况下存在并发问题，那么在订阅之前最后一次生成的Observable才会被订阅和发射。

<br><br><br>
* __scan( )__ — 对Observable发射的每一项数据应用一个函数，然后按顺序依次发射每一个值。操作符对原始Observable发射的第一项数据应用一个函数，然后将那个函数的结果作为自己的第一项数据发射。它将函数的结果同第二项数据一起填充给这个函数来产生它自己的第二项数据。它持续进行这个过程来产生剩余的数据序列。这个操作符在某些情况下被叫做accumulator。
![](http://reactivex.io/documentation/operators/images/scan.png)

<br><br><br>
* __groupBy( )__ — 将Observable分拆为Observable集合，将原始Observable发射的数据按Key分组，每一个Observable发射一组不同的数据。

GroupBy操作符将原始Observable分拆为一些Observables集合，它们中的每一个发射原始Observable数据序列的一个子序列。哪个数据项由哪一个Observable发射是由一个函数判定的，这个函数给每一项指定一个Key，Key相同的数据会被同一个Observable发射。

RxJava实现了groupBy操作符。它返回Observable的一个特殊子类GroupedObservable，实现了GroupedObservable接口的对象有一个额外的方法getKey，这个Key用于将数据分组到指定的Observable。

有一个版本的groupBy允许你传递一个变换函数，这样它可以在发射结果GroupedObservable之前改变数据项。
![](http://reactivex.io/documentation/operators/images/groupBy.c.png)

<br><br><br>
* __buffer( )__ — 它定期从Observable收集数据到一个集合，然后把这些数据集合打包发射，而不是一次发射一个。
>Buffer操作符将一个Observable变换为另一个，原来的Observable正常发射数据，变换产生的Observable发射这些数据的缓存集合。Buffer操作符在很多语言特定的实现中有很多种变体，它们在如何缓存这个问题上存在区别。
![](http://reactivex.io/documentation/operators/images/Buffer.png)

>注意：如果原来的Observable发射了一个onError通知，Buffer会立即传递这个通知，而不是首先发射缓存的数据，即使在这之前缓存中包含了原始Observable发射的数据。

>Window操作符与Buffer类似，但是它在发射之前把收集到的数据放进单独的Observable，而不是放进一个数据结构。


* __window( )__ — 定期将来自Observable的数据分拆成一些Observable窗口，然后发射这些窗口，而不是每次发射一项

![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/window.C.png)

>Window和Buffer类似，但不是发射来自原始Observable的数据包，它发射的是Observables，这些Observables中的每一个都发射原始Observable数据的一个子集，最后发射一个onCompleted通知。
>
和Buffer一样，Window有很多变体，每一种都以自己的方式将原始Observable分解为多个作为结果的Observable，每一个都包含一个映射原始数据的window。用Window操作符的术语描述就是，当一个窗口打开(when a window "opens")意味着一个新的Observable已经发射（产生）了，而且这个Observable开始发射来自原始Observable的数据；当一个窗口关闭(when a window "closes")意味着发射(产生)的Observable停止发射原始Observable的数据，并且发射终止通知onCompleted给它的观察者们。

>在RxJava中有许多种Window操作符的变体。

<br><br><br>
* __cast( )__ — 在发射之前强制将Observable发射的所有数据转换为指定类型.

>翻阅代码发现，实际上看到内部实现就是map做的。

![](http://reactivex.io/documentation/operators/images/map.png)


<br><br><br>
#####Filtering Observables

* __filter( )__ — 过滤数据。__ofType( )__ — 只发射指定类型的数据
>filter是很简单的概念，给出规则，按照规则过滤出符合的数据。
>
>ofType是filter操作符的一个特殊形式。它过滤一个Observable只返回指定类型的数据。
例子：

	Observable.just(1, 2, 3, 4, 5)
          .filter(new Func1<Integer, Boolean>() {
              @Override
              public Boolean call(Integer item) {
                return( item < 4 );
              }
          }).subscribe(new Subscriber<Integer>() {
        @Override
        public void onNext(Integer item) {
            System.out.println("Next: " + item);
        }

        @Override
        public void onError(Throwable error) {
            System.err.println("Error: " + error.getMessage());
        }

        @Override
        public void onCompleted() {
            System.out.println("Sequence complete.");
        }
    });
输出

	Next: 1
	Next: 2
	Next: 3
	Sequence complete.

<br><br><br>

* __takeLast( )__ — 只发射最后的N项数据
> 使用TakeLast操作符修改原始Observable，你可以只发射Observable'发射的后N项数据，忽略前面的数据。

* __last( )__ — 只发射最后的一项数据

* __lastOrDefault( )__ — 只发射最后的一项数据，如果Observable为空就发射默认值
* __takeLastBuffer( )__ — 它和takeLast类似，，唯一的不同是它把所有的数据项收集到一个List再发射，而不是依次发射一个。
>如果你只对Observable发射的最后一项数据，或者满足某个条件的最后一项数据感兴趣，你可以使用Last操作符。

>在某些实现中，Last没有实现为一个返回Observable的过滤操作符，而是实现为一个在当时就发射原始Observable指定数据项的阻塞函数。在这些实现中，如果你想要的是一个过滤操作符，最好使用TakeLast(1)。

<br><br><br>

* __first( ) and takeFirst( )__ — 只发射第一项数据，或者满足某种条件的第一项数据
* __firstOrDefault( )__ — 只发射第一项数据，如果Observable为空就发射默认值
>如果你只对Observable发射的第一项数据，或者满足某个条件的第一项数据感兴趣，你可以使用First操作符。

>在某些实现中，First没有实现为一个返回Observable的过滤操作符，而是实现为一个在当时就发射原始Observable指定数据项的阻塞函数。在这些实现中，如果你想要的是一个过滤操作符，最好使用Take(1)或者ElementAt(0)。
>
>firstOrDefault与first类似，但是在Observagle没有发射任何数据时发射一个你在参数中指定的默认值。

<br><br><br>

* __elementAt( )__ — 发射第N项数据
* __elementAtOrDefault( )__ — 发射第N项数据，如果Observable数据少于N项就发射默认值
>如果你只对Observable发射的第一项数据，或者满足某个条件的第一项数据感兴趣，你可以使用First操作符。

>在某些实现中，First没有实现为一个返回Observable的过滤操作符，而是实现为一个在当时就发射原始Observable指定数据项的阻塞函数。在这些实现中，如果你想要的是一个过滤操作符，最好使用Take(1)或者ElementAt(0)。
>
>firstOrDefault与first类似，但是在Observagle没有发射任何数据时发射一个你在参数中指定的默认值。

<br><br><br>

* __skip( )__ — 使用Skip操作符，你可以忽略Observable'发射的前N项数据，只保留之后的数据。
* __skipLast( )__ — 使用SkipLast操作符修改原始Observable，你可以忽略Observable'发射的后N项数据，只保留前面的数据。
>这两个操作符都支持用逻辑上的时间来做筛选单位。分别是2个复写方法


<br><br><br>


* __take( )__ — 使用Take操作符让你可以修改Observable的行为，只返回前面的N项数据，然后发射完成通知，忽略剩余的数据。


* __sample( )__ or __throttleLast( )__ — 定期发射Observable最近的数据
>sample(别名throttleLast)的一个变体按照你参数中指定的时间间隔定时采样（TimeUnit指定时间单位）。
>sample的另外一种重载形式是传入一个observerable对象，这个对象开始发射数据的时候，我们的主observable才开始发射，个人理解这类似于一种触发器。
>
>注意：如果自上次采样以来，原始Observable没有发射任何数据，这个操作返回的Observable在那段时间内也不会发射任何数据。
	
	调用例子如下：
	Observable.create(new Observable.OnSubscribe<Integer>() {
	            @Override
	            public void call(Subscriber<? super Integer> subscriber) {
	                if(subscriber.isUnsubscribed()) return;
	                try {
	                    //前8个数字产生的时间间隔为1秒，后一个间隔为3秒
	                    for (int i = 1; i < 9; i++) {
	                        subscriber.onNext(i);
	                        Thread.sleep(1000);
	                    }
	                    Thread.sleep(2000);
	                    subscriber.onNext(9);
	                    subscriber.onCompleted();
	                } catch(Exception e){
	                    subscriber.onError(e);
	                }
	            }
	        }).subscribeOn(Schedulers.newThread())
	          .sample(2200, TimeUnit.MILLISECONDS)  //采样间隔时间为2200毫秒
	          .subscribe(new Subscriber<Integer>() {
	              @Override
	              public void onNext(Integer item) {
	                  System.out.println("Next: " + item);
	              }
	
	              @Override
	              public void onError(Throwable error) {
	                  System.err.println("Error: " + error.getMessage());
	              }
	
	              @Override
	              public void onCompleted() {
	                  System.out.println("Sequence complete.");
	              }
	          });
	运行结果如下： 
	Next: 3 
	Next: 5 
	Next: 7 
	Next: 8 
	Sequence complete.

* __throttleFirst( )__ — 定期发射Observable发射的第一项数据
>throttleFirst与throttleLast/sample不同，在每个采样周期内，它总是发射原始Observable的第一项数据，而不是最近的一项。


* __throttleWithTimeout( )__ or __debounce( )__ — 只有当Observable在指定的时间后还没有发射数据时，才发射一个数据.
>Debounce操作符会过滤掉发射速率过快的数据项。

>RxJava将这个操作符实现为throttleWithTimeout和debounce。

>注意：这个操作符会会接着最后一项数据发射原始Observable的onCompleted通知，即使这个通知发生在你指定的时间窗口内（从最后一项数据的发射算起）。也就是说，onCompleted通知不会触发限流。


* __timeout( )__ — 如果在一个指定的时间段后还没发射数据，就发射一个异常。
>如果原始Observable过了指定的一段时长没有发射任何数据，Timeout操作符会以一个onError通知终止这个Observable。

>有些版本的timeout在超时时会切换到使用一个你指定的备用的Observable，而不是发错误通知。它也默认在computation调度器上执行。

<br><br>
* __distinct( )__ — 过滤掉重复数据

>Distinct的过滤规则是：只允许还没有发射过的数据项通过。

>在某些实现中，有一些变体允许你调整判定两个数据不同(distinct)的标准。还有一些实现只比较一项数据和它的直接前驱，因此只会从序列中过滤掉连续重复的数据。


示例

	Observable.just(1, 2, 1, 1, 2, 3)
	          .distinct()
	          .subscribe(new Subscriber<Integer>() {
	        @Override
	        public void onNext(Integer item) {
	            System.out.println("Next: " + item);
	        }
	
	        @Override
	        public void onError(Throwable error) {
	            System.err.println("Error: " + error.getMessage());
	        }
	
	        @Override
	        public void onCompleted() {
	            System.out.println("Sequence complete.");
	        }
	    });
	输出
	
	Next: 1
	Next: 2
	Next: 3
	Sequence complete.


<br><br>
* __distinctUntilChanged( )__ — RxJava还是实现了一个distinctUntilChanged操作符。它只判定一个数据和它的直接前驱是否是不同的。

![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/distinctUntilChanged.png)

<br><br>

 * __ignoreElements( )__ — 丢弃所有的正常数据，只发射错误或完成通知
>IgnoreElements操作符抑制原始Observable发射的所有数据，只允许它的终止通知（onError或onCompleted）通过。

>如果你不关心一个Observable发射的数据，但是希望在它完成时或遇到错误终止时收到通知，你可以对Observable使用ignoreElements操作符，它会确保永远不会调用观察者的onNext()方法。
>
>RxJava将这个操作符实现为ignoreElements。


<br><br><br>
#####Combining Observables


* __startWith( )__ — 在数据序列的开头增加一项数据
>如果你想要一个Observable在发射数据之前先发射一个指定的数据序列，可以使用StartWith操作符。（如果你想一个Observable发射的数据末尾追加一个数据序列可以使用Concat操作符。

![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/startWith.png)

<br><br>

* __merge( )__ — 将多个Observable合并为一个
* __mergeDelayError( )__ — 合并多个Observables，让没有错误的Observable都完成后再发射错误通知

>使用Merge操作符你可以将多个Observables的输出合并，就好像它们是一个单个的Observable一样。
>
Merge可能会让合并的Observables发射的数据交错（有一个类似的操作符Concat不会让数据交错，它会按顺序一个接着一个发射多个Observables的发射物）。

>正如图例上展示的，任何一个原始Observable的onError通知会被立即传递给观察者，而且会终止合并后的Observable。

>*mergeDelayError*

>在很多ReactiveX实现中还有一个叫MergeDelayError的操作符，它的行为有一点不同，它会保留onError通知直到合并后的Observable所有的数据发射完成，在那时它才会把onError传递给观察者。
![](http://reactivex.io/documentation/operators/images/mergeDelayError.C.png)

<br><br>

* __zip( )__ — 使用一个函数组合多个Observable发射的数据集合，然后再发射这个结果

>Zip操作符返回一个Obversable，它使用这个函数按顺序结合两个或多个Observables发射的数据项，然后它发射这个函数返回的结果。它按照严格的顺序应用这个函数。它只发射与发射数据项最少的那个Observable一样多的数据。

>RxJava将这个操作符实现为zip和zipWith。

![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/zip.o.png)

<br><br>

* __combineLatest( )__ — 当两个Observables中的任何一个发射了一个数据时，通过一个指定的函数组合每个Observable发射的最新数据（一共两个数据），然后发射这个函数的结果

>CombineLatest操作符行为类似于zip，但是只有当原始的Observable中的每一个都发射了一条数据时zip才发射数据。CombineLatest则在原始的Observable中任意一个发射了数据时发射一条数据。当原始Observables的任何一个发射了一条数据时，CombineLatest使用一个函数结合它们最近发射的数据，然后发射这个函数的返回值。

![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/combineLatest.png)

<br><br>

* __join( )— 无论何时，如果一个Observable发射了一个数据项，只要在另一个Observable发射的数据项定义的时间窗口内，就将两个Observable发射的数据合并发射


![](http://reactivex.io/documentation/operators/images/join.c.png)
>merge按顺序合并两个Observable发出的数据
>
zip按顺序打包两个Obserable发出数据的合体； 那么join呢？ 
在阅读了RxJava Essential和Reactivex.io上的解释后，还是很难理解,官方的示例图也是基本没看明白：

When left produces a value, a window is opened. That value is also then passed to the leftDurationSelector function. The result of this function is an IObservable<TLeftDuration>. When that sequence produces a value or completes then the window for that value is closed. Note that it is irrelevant what the type of TLeftDuration is. This initially left me with the feeling that IObservable<TLeftDuration> was all a bit over kill as you effectively just need some sort of event to say 'Closed'. However by allowing you to use IObservable<T> you can do some clever stuff as we will see later.

So let us first imagine a scenario where we have the left sequence producing values twice as fast as the right sequence. Imagine that we also never close the windows. We could do this by always returning Observable.Never<Unit>() from the leftDurationSelector function. This would result in the following pairs being produced.	
	
	Left Sequence
	
	L 0-1-2-3-4-5-
	Right Sequence
	
	R --A---B---C-
	0, A
	1, A
	0, B
	1, B
	2, B
	3, B
	0, C
	1, C
	2, C
	3, C
	4, C
	5, C
	As you can see the left values are cached and replayed each time the right produces a value.

>说白了join操作我们不能把两个Observable等价看待，join的效果类似于排列组合，把第一个数据源A作为基座窗口，他根据自己的节奏不断发射数据元素，第二个数据源B，每发射一个数据，我们都把它和第一个数据源A中已经发射的数据进行一对一匹配；
>
>举例来说，如果某一时刻B发射了一个数据“B”,此时A已经发射了0，1，2，3共四个数据，那么我们的合并操作就会把“B”依次与0,1,2,3配对，得到四组数据： 0, B 1, B 2, B 3, B

<br><br>

* __switchOnNext( )__ — 将一个发射Observables的Observable转换成另一个Observable，后者发射这些Observables最近发射的数据.

源码：

	public static <T> Flowable<T> switchOnNext(Publisher<? extends Publisher<? extends T>> sources) {
	        return fromPublisher(sources).switchMap((Function)Functions.identity());
	}

实例代码：

    private Observable<String> createSwitchObserver(int index) {
        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                for (int i = 1; i < 5; i++) {
                    subscriber.onNext(index + "-" + i);
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).subscribeOn(Schedulers.newThread());
    }

    public void switchObserver(){
        Observable.switchOnNext(Observable.create(
                new Observable.OnSubscribe<Observable<String>>() {
                    @Override
                    public void call(Subscriber<? super Observable<String>> subscriber) {
                        for (int i = 1; i < 3; i++) {
                            subscriber.onNext(createSwitchObserver(i));
                            try {
                                Thread.sleep(2000);
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                }
        ))
                .subscribe(new Action1<String>() {
                    @Override
                    public void call(String s) {
                        logger("switch:" + s);
                    }
                });
    }
打印结果：

	switch:1-1 
	switch:1-2 
	switch:2-1 
	switch:2-2 
	switch:2-3 
	switch:2-4
从打印结果看，第一个小Observable只发射出了两个数据， 
第二个小Observable就被源Observable发射出来了，所以 第一个 接下来的两个数据被丢弃。


