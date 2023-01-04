



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

