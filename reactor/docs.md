



내맘대로 번역

https://projectreactor.io/docs/core/release/reference/#intro-reactive





# #3 Introduction to Reactive Programming



리액터가 세상에 등장한 이유

1. Callback은 보일러플레이트 코드가 너무 많고 중첩될 경우 Callback 지옥
2. Callback 보다는 낫지만 그래도 여전히 높은 난이도의 Future



```java
// callback
userService.getFavorites(userId, new Callback<List<String>>() { 
  public void onSuccess(List<String> list) { 
    if (list.isEmpty()) { 
      suggestionService.getSuggestions(new Callback<List<Favorite>>() {
        public void onSuccess(List<Favorite> list) { 
          UiUtils.submitOnUiThread(() -> { 
            list.stream()
                .limit(5)
                .forEach(uiList::show); 
            });
        }

        public void onError(Throwable error) { 
          UiUtils.errorPopup(error);
        }
      });
    } else {
      list.stream() 
          .limit(5)
          .forEach(favId -> favoriteService.getDetails(favId, 
            new Callback<Favorite>() {
              public void onSuccess(Favorite details) {
                UiUtils.submitOnUiThread(() -> uiList.show(details));
              }

              public void onError(Throwable error) {
                UiUtils.errorPopup(error);
              }
            }
          ));
    }
  }

  public void onError(Throwable error) {
    UiUtils.errorPopup(error);
  }
});

// reactor

userService.getFavorites(userId) 
           .flatMap(favoriteService::getDetails) 
           .switchIfEmpty(suggestionService.getSuggestions()) 
           .take(5) 
           .publishOn(UiUtils.uiThreadScheduler()) 
           .subscribe(uiList::show, UiUtils::errorPopup); 
           
           
// future

CompletableFuture<List<String>> ids = ifhIds(); 

CompletableFuture<List<String>> result = ids.thenComposeAsync(l -> { 
	Stream<CompletableFuture<String>> zip =
			l.stream().map(i -> { 
				CompletableFuture<String> nameTask = ifhName(i); 
				CompletableFuture<Integer> statTask = ifhStat(i); 

				return nameTask.thenCombineAsync(statTask, (name, stat) -> "Name " + name + " has stats " + stat); 
			});
	List<CompletableFuture<String>> combinationList = zip.collect(Collectors.toList()); 
	CompletableFuture<String>[] combinationArray = combinationList.toArray(new CompletableFuture[combinationList.size()]);

	CompletableFuture<Void> allDone = CompletableFuture.allOf(combinationArray); 
	return allDone.thenApply(v -> combinationList.stream()
			.map(CompletableFuture::join) 
			.collect(Collectors.toList()));
});

List<String> results = result.join(); 
assertThat(results).contains(
		"Name NameJoe has stats 103",
		"Name NameBart has stats 104",
		"Name NameHenry has stats 105",
		"Name NameNicole has stats 106",
		"Name NameABSLAJNFOAJNFOANFANSF has stats 121");
		
// reactor

Flux<String> ids = ifhrIds(); 

Flux<String> combinations =
		ids.flatMap(id -> { 
			Mono<String> nameTask = ifhrName(id); 
			Mono<Integer> statTask = ifhrStat(id); 

			return nameTask.zipWith(statTask, 
					(name, stat) -> "Name " + name + " has stats " + stat);
		});

Mono<List<String>> result = combinations.collectList(); 

List<String> results = result.block(); 
assertThat(results).containsExactly( 
		"Name NameJoe has stats 103",
		"Name NameBart has stats 104",
		"Name NameHenry has stats 105",
		"Name NameNicole has stats 106",
		"Name NameABSLAJNFOAJNFOANFANSF has stats 121"
);
```





리액터가 전통적인 비동기 프로그래밍 방식을 개선하는것 외에도 갖고 있는 측면들

- **Composability** and **readability**
- Data as a **flow** manipulated with a rich vocabulary of **operators**
- Nothing happens until you **subscribe**
- **Backpressure** or *the ability for the consumer to signal the producer that the rate of emission is too high*
- **High level** but **high value** abstraction that is *concurrency-agnostic*





cold 시퀀스: `Subscriber` 마다 구독을 새로하는 것.

hot 시퀀스: `Subscriber` 별로 구독을 하지 않고 뭔가 상태를 유지하는 느낌인듯?



# #4 Reactor Core Features



Flux: 0...N 개의 리액티브 시퀀스를 표현

![Flux](https://projectreactor.io/docs/core/release/reference/images/flux.svg)

Mono: 0..1개의 값을 표현

![Mono](https://projectreactor.io/docs/core/release/reference/images/mono.svg)

Flux와 Mono중 어떤것을 사용할지 결정하는 방법은 실제 요소가 얼마나 있는지로 판단하면됨 예를 들어 HTTP Request 같은 경우 한번 호출하고 그 결과도 하나이므로 Mono를 사용하는게 더 좋음



Mono와  Flux를 사용하는 가장 쉬운 방법 => 팩토리 메서드를 사용하기

```
Flux<String> seq1 = Flux.just("foo", "bar", "foobar");

List<String> iterable = Arrays.asList("foo", "bar", "foobar");
Flux<String> seq2 = Flux.fromIterable(iterable);
```

```
Mono<String> noData = Mono.empty(); 

Mono<String> data = Mono.just("foo");

Flux<Integer> numbersFromFiveToSeven = Flux.range(5, 3); 
```



다양한 subscribe 변형 메서드를 통해서 구독할 수 있음

```
subscribe(); 

subscribe(Consumer<? super T> consumer); 

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer); 

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer); 

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer,
          Consumer<? super Subscription> subscriptionConsumer); 
```



람다를 기반으로한 모든 subscribe는 리턴으로 **Disposable**을 리턴한다. **Disposable**은 dispose() 메서드를 통해 구독을 취소할 수 있다. 

**Disposables** 의 **composite** 메서드 => 여러 disposable을 composite해서 한번에 종료시킬 수 있다.



----

BaseSubscriber를 직접 구현해서 제공해주는 훅 메서드를 통해 구독을 커스텀할 수 있다.

```
Flux.range(1, 10)
    .doOnRequest(r -> System.out.println("request of " + r))
    .subscribe(new BaseSubscriber<Integer>() {

      @Override
      public void hookOnSubscribe(Subscription subscription) {
        request(1);
      }

      @Override
      public void hookOnNext(Integer integer) {
        System.out.println("Cancelling after having received " + integer);
        cancel();
      }
    });
```

BaseSubscriber를 직접 구현할 때는 얼마만큼의 용량을 처리할 수 있는지 고민해야하고 hookOnNext()에서는 최소한개의 request를 요청해야한다.



limitRate와 limitRequset로 upstream, downstream 간의 요청을 직접 조정할 수 있다.

일반적으로 downstream에서 request는 Long.MAX_VALUE 만큼 요청을 한다.

```
Flux.range(1, 100)
    .log()
    .subscribe { println(it) }
    
[ INFO] (main) | request(unbounded)
[ INFO] (main) | onNext(1)
1
[ INFO] (main) | onNext(2)
2
[ INFO] (main) | onNext(3)
3
[ INFO] (main) | onNext(4)
...

```

이때 upstream에서 데이터를 방출하는게 downstream이 소비하는 속도보다 빠르거나 upstream의 데이터소스가 한번에 우르르 몰리면 순간적으로 downstream에서 대기하는? 데이터요소가 많아지게 되고 이는 비효율적인 상황이 된다.

이때 limitRate를 걸면 된다.

```
Flux.range(1, 100)
    .log()
    .delaySequence(Duration.ofMillis(100))
    .limitRate(10)
    .subscribe(System.out::println);
    
[ INFO] (main) | request(10)
[ INFO] (main) | onNext(1)
[ INFO] (main) | onNext(2)
[ INFO] (main) | onNext(3)
[ INFO] (main) | onNext(4)
[ INFO] (main) | onNext(5)
[ INFO] (main) | onNext(6)
[ INFO] (main) | onNext(7)
[ INFO] (main) | onNext(8)
[ INFO] (main) | onNext(9)
[ INFO] (main) | onNext(10)
1
2
3
4
5
6
7
8
[ INFO] (parallel-1) | request(8)
[ INFO] (parallel-1) | onNext(11)
[ INFO] (parallel-1) | onNext(12)
[ INFO] (parallel-1) | onNext(13)
[ INFO] (parallel-1) | onNext(14)
[ INFO] (parallel-1) | onNext(15)
[ INFO] (parallel-1) | onNext(16)
[ INFO] (parallel-1) | onNext(17)
[ INFO] (parallel-1) | onNext(18)
```

똑똑한게 limitRate의 75%만큼의 데이터를 소비하면  upstream한테 다시 75%만큼의 데이터를 요청한다. 

limitRequest는 downstream 요청을 n개만큼 제한하고 onComplete를 실행하는데 limitRequest는 deprecated되었고 take 쓰면 된다.

----



Flux.generate() 사용 예제

```
fun main() {
    Flux.generate(
        { 0 }
    ) { state, sink ->
        sink.next("3 x $state = ${state.times(3)}")
        if (state == 10) sink.complete()
        state + 1
    }.subscribe {
        println(it)
    }
}
3 x 0 = 0
3 x 1 = 3
3 x 2 = 6
3 x 3 = 9
3 x 4 = 12
3 x 5 = 15
3 x 6 = 18
3 x 7 = 21
3 x 8 = 24
3 x 9 = 27
3 x 10 = 30
```

뭔가 마지막에 리소스를 release 시키거나 .. 커넥션을 종료시키고 싶을땐?



```
fun main() {
    Flux.generate(
        { 0 },
        { state, sink ->
            sink.next("3 x $state = ${state.times(3)}")
            if (state == 10) sink.complete()
            state + 1
        }
    ) {
        println("handle... $it")
    }.subscribe {
        println(it)
    }
}
3 x 0 = 0
3 x 1 = 3
3 x 2 = 6
3 x 3 = 9
3 x 4 = 12
3 x 5 = 15
3 x 6 = 18
3 x 7 = 21
3 x 8 = 24
3 x 9 = 27
3 x 10 = 30
```





---



Flux.create는 멀티스레드에서 넘어오는 데이터를 방출할때 사용한다.

create는 이벤트 리스너 기반의 비동기 API를 bridge할때 유용하게 사용할 수 있다.

create는 코드 자체를 비동기화하는것은 아니기 때문에 long-blocking create lambda는 pipeline에 락을 잡을 수 있다. (테스트를 못해보겠어서 뭔지 모르겠음)

create 로 비동기 API를 연결할때 backpressure를 사용할 수 있으므로 **OverflowStrategy** 로 backpressure 방식을 구체화할 수 있다

- **IGNORE**: downstream backpressure requests 를 무시. downstream 큐가 가득찼을때 **IllegalStateException**가 발생할 수 있다.
- **ERROR** downstream이 upstream의 속도를 따라갈 수 없을 때 **IllegalStateException**를 발생
- **DROP**: downstream이 수신할 준비가 되지 않은 경우 신호를 drop
- **LATEST**: downstream이 upstream의 최신 신호만 받음
- **BUFFER**: (기본값) downstream이 신호를 받을 수 없어도 일단 모든 신호를 버퍼함 **OutOfMemoryError** 발생할 수도 있음



create vs generate (https://www.baeldung.com/java-flux-create-generate)





| Flux Create                                                  | Flux Generate                                         |
| ------------------------------------------------------------ | ----------------------------------------------------- |
| *Consumer<FluxSink\>*                                        | *Consumer<SynchronousSink\>* 를 사용한다              |
| consumer call을 한번만 한다                                  | downstream에서 필요에따라  cosumer 콜을 여러번한다    |
| consumer는 0..N 요소를 방출할 수 있다                        | 오직 한개의 요소만 방출할 수 있다.                    |
| publisher는 downstream의 상태를 모르기때문에 Overflow strategy 로 흐름 제어를 허용한다. | publisher는 downstream 의 필요에따라 요소를 생성한다. |
| *FluxSink*는 멀티스레드를 사용하여 요소를 방출할 수 있다.    | 한번에 하나의 요소만 방출한다.                        |



그냥 간단하게 뭔가 backpressure를 사용해야하고 요소가 끊임없이 쏟아지는 경우 create 사용. 하지만 이때 OverflowStrategy를 잘 짜야할듯? 이거 아니면 그냥  generate 사용





---



Flux.push는 generate와 create의 중간 지점. 단일 생성자의 이벤트를 프로세싱할때 효과적이다. (create는 멀티)



