
Spring Webflux 는 reactor 를 사용해 반응형 어플리케이션을 짤 수 있다. 함수형 프로그래밍 기반으로 non-blocking 구조를 가져서 IO 등에 의한 thread block 을 최소화 하는 것이 목적이다.
그치만 제대로 만드려면 몇가지 어려운 부분이 있는데, 그 중 하나가 에러 핸들링인 것 같다. `.onError*` 류의 메서드가 몇십개는 되는 것 같고 언제 실행될지, 코드에서 어디에 위치해야 하는지 직관적이지 않기 때문이다.

여러 케이스들을 실험해보며 Webflux 에서 에러핸들링을 어떻게 해야하는지 알아보자.

# 케이스 1 `.subscribe()`

스트림의 끝단인 `.subscribe()` 에선 마지막 결과를 컨슘할수도 있고 에러처리를 할수도 있다.
```java
List<Integer> id = List.of(1,2,3,4,5,6,7,8,9,0);  
Flux.fromStream(id.stream())  
		.map(i -> {  
			if (i%2 == 0) {  
				throw new RuntimeException("error 1");  
			}  
			return i;  
		})  
		.subscribe(  
				res -> log.debug("result " + res),  
				err -> log.error("error " + err.getMessage())  
		);  

>
2024-11-27T00:46:54.596+09:00 DEBUG 307803 --- [module-spring] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : result 1
2024-11-27T00:46:54.598+09:00 ERROR 307803 --- [module-spring] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : error error 0 : 2
```

## `.subscribe()` 에서 에러처리를 하지 않으면?

만약 에러 핸들러를 지정해두지 않으면 `default onErrorDropped` 라는 메세지와 에러 스택들이 로깅된다.

```java
List<Integer> id = List.of(1,2,3,4,5,6,7,8,9,0);  
try {  
    Flux.fromStream(id.stream())  
            .map(i -> {  
                if (i % 2 == 0) {  
                    throw new RuntimeException("error 1 : " + i);  
                }  
                return i;  
            })  
            .subscribe(  
                    res -> log.debug("result " + res)  
            );  
}catch (Exception err) {  
    log.error("outside Error : " + err);  
}  
  

> 
2024-11-26T23:15:45.074+09:00 ERROR 264594 --- [module-spring] [  restartedMain] reactor.core.publisher.Operators         : Operator called default onErrorDropped

reactor.core.Exceptions$ErrorCallbackNotImplemented: java.lang.RuntimeException: error 1 : 2
Caused by: java.lang.RuntimeException: error 1 : 2
	at ...

```

`default onErrorDropped` 라고 로깅된 후 에러를 다시 던진다. 스택 트레이스에 에러가 발생한 메서드 정돈 표시되지만 99% 는 쓸모없는 에러 메세지라서 말줄임표 해두었다. (프록시 구조에서 에러날 때를 생각하면 비슷하다.)

재밌는 점은 `try-catch` 블록 안에 코드들이 들어있기 때문에 **에러를 잡아야 할 것 같지만 그렇지 않다는 것**이다. 코드 블락 안에 있는 내용들 **자체**에선 아무 에러가 나지 않고, 내용을 실제 실행하는 다른 쓰레드에서 에러가 났기 때문에, main 쓰레드에서 catch 하지 않은 것이다.

try-catch 가 스트림 내부에 들어가면 상식적으로 동작한다.

```java
List<Integer> id = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 0);  
Flux.fromStream(id.stream())  
        .map(i -> {  
            try {  
                if (i % 2 == 0) {  
                    throw new RuntimeException("error 1 : " + i);  
                }  
            } catch (Exception err) {  
                log.error("inside Error : " + err);  
            }  
            return i;  
        })  
        .subscribe(  
                res -> log.debug("result " + res)  
        );

> 2024-11-26T23:25:09.740+09:00 ERROR 268509 --- [module-spring] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : inside Error : java.lang.RuntimeException: error 1 : 2
2024-11-26T23:25:09.741+09:00 ERROR 268509 --- [module-spring] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : inside Error : java.lang.RuntimeException: error 1 : 4
2024-11-26T23:25:09.741+09:00 ERROR 268509 --- [module-spring] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : inside Error : java.lang.RuntimeException: error 1 : 6
2024-11-26T23:25:09.741+09:00 ERROR 268509 --- [module-spring] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : inside Error : java.lang.RuntimeException: error 1 : 8
2024-11-26T23:25:09.741+09:00 ERROR 268509 --- [module-spring] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : inside Error : java.lang.RuntimeException: error 1 : 0
```

좋다. 생각대로 동작한다. 근데 잘 보니 `.subscribe()` 에서 컨슘하고 있는 debug log 는 하나도 없다. 왜 그럴까..? (로그 레벨 높아서 그런거였다.)

```html
 Subscribe a {@link Consumer} to this {@link Flux} that will consume all the elements in the  sequence. It will request an unbounded demand ({@code Long.MAX_VALUE}).
 For a passive version that observe and forward incoming data see {@link #doOnNext(java.util.function.Consumer)}.
 For a version that gives you more control over backpressure and the request, see {@link #subscribe(Subscriber)} with a {@link BaseSubscriber}.
 Keep in mind that since the sequence can be asynchronous, this will immediately return control to the calling thread. This can give the impression the consumer is not invoked when executing in a main thread or a unit test for instance.
 @param consumer the consumer to invoke on each value (onNext signal)
 @return a new {@link Disposable} that can be used to cancel the underlying {@link Subscription}
```

`.subscribe()` 의 javadoc 인데, 메인스레드에서 컨슈머가 작동하지 않는 듯 보일 수 있다고 한다. 이건 그냥 메인스레드가 일찍 끝나서 그렇다고 하는 것임. (타이밍 이슈)

# 케이스 2 `.doOnError()`

```java
public static void main(String[] args){
    List<Integer> id = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 0);  
    Flux.fromStream(id.stream())  
            .flatMap(i -> {  
                if (i % 2 == 0) {  
                    throw new RuntimeException("error 1");  
                }  
                return monoMapper(i);  
            })  
            .doOnError(err -> log.error("doOnError 1 " + err.getMessage()))  
            .subscribe(  
                    res -> log.debug("result " + res),  
                    err -> log.error("error " + err.getMessage())  
            );  
    Thread.sleep(999);  
    System.exit(0);  
}  
  
private static Mono<Integer> monoMapper(Integer i) {  
    if (i % 3 == 0) {  
        throw new RuntimeException("error 2");  
    }  
    return Mono.just(i);  
}

>
2024-11-27T01:00:57.675+09:00 DEBUG 315254 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : result 1
2024-11-27T01:00:57.682+09:00 ERROR 315254 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : doOnError 1 error 1
2024-11-27T01:00:57.683+09:00 ERROR 315254 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : error error 1
```

`.doOnError()` 를 사용했을 때 제대로 로깅되었고 stream 이 끝났다.  private 메서드에 하나 더 추가해보자.

```java
  
private static Mono<Integer> monoMapper(Integer i) {  
    if (i % 3 == 0) {  
        throw new RuntimeException("error 2");  
    }  
    return Mono.just(i)  
            .doOnError(err -> log.error("doOnError 2 " + err.getMessage()));  
}
```

하지만 이번엔 doOnErro 2 라는 로그는 뜨지 않는다. 따져보면 이런 것 같다.

1. 첫 번째 데이터 1은 정상적으로 컨슘된다.
2. 두 번째 데이터는 에러가 있었기 때문에 해당 스트림에 있는 에러 핸들러들이 처리해 준다. (1이 겪었던 컨슘과 다른 경로)
3. private method 에 있는 Mono 의 스트림에선 에러가 나지 않았으므로 로깅되지 않는다.

그러면 private 메서드 내부에서 에러가 나도록 바꾸면 될까? main 메서드의 나머지 조건을 바꾸고 실행하면

```java
2024-11-27T01:06:16.997+09:00 DEBUG 317545 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : result 1
2024-11-27T01:06:16.998+09:00 DEBUG 317545 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : result 2
2024-11-27T01:06:17.000+09:00 ERROR 317545 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : doOnError 1 error 2
2024-11-27T01:06:17.001+09:00 ERROR 317545 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : error error 2
```

여전히 doOnErro 2 라는 로그는 남지 않는다. private method 를 다시 자세히보면 , doOnError 2 가 있는 스트림 외부에서 에러가 나기 때문에, 그것을 밖에서 감싼 doOnError 1에서만 에러핸들링이 되는 것이다. doOnError 2 가 제대로 핸들링하려면 

```java
private static Mono<Integer> monoMapper(Integer i) {
	if (i % 3 == 0) {
		throw new RuntimeException("error 2");
	}
	return Mono.just(i)
			.map(j -> {
				if (j % 2 == 0) {
					throw new RuntimeException("error 3");
				}
				return j;
			})
			.doOnError(err -> log.error("doOnError 2 " + err.getMessage()));
}

>
2024-11-27T01:09:48.014+09:00 DEBUG 319031 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : result 1
2024-11-27T01:09:48.016+09:00 ERROR 319031 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : doOnError 2 error 3
2024-11-27T01:09:48.016+09:00 ERROR 319031 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : doOnError 1 error 3
2024-11-27T01:09:48.016+09:00 ERROR 319031 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : error error 3
```

스트림의 가장 내부에서 발생한 error 3 은 외부 에러 핸들러들이 전부 처리하는 것을 알 수 있다. 따라서 스트림이 복잡하게 얽혀있는데 에러 처리를 일괄적으로 하려면 최상단 스트림에서 해줘야 한다. 실제로 그렇게 맘편하게 에러처리가 되는 경우는 로깅하는 정도 밖에 없을 것이다. 어떤 경우는 내부 스트림에서 에러 처리한 후 더 이상 상위 스트림에서 처리하지 않아야 하는 경우도 있을 것이다. 그런 경우엔 `.onErrorResume()` 을 쓰자.

# 케이스 3 `.onErrorResume()`

상위 스트림으로 에러가 전파되지 않도록 private 메서드를 좀 바꿔봤다.

```java
private static Mono<Integer> monoMapper(Integer i) {  
    if (i % 3 == 0) {  
        throw new RuntimeException("error 2");  
    }  
    return Mono.just(i)  
            .map(j -> {  
                if (j % 2 == 0) {  
                    throw new RuntimeException("error 3");  
                }  
                return j;  
            })  
            .onErrorResume(err -> {  
                log.error("doOnError 2 " + err.getMessage());  
                return Mono.empty();  
            });  
}

>
2024-11-27T01:14:02.386+09:00 DEBUG 320742 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : result 1
2024-11-27T01:14:02.387+09:00 ERROR 320742 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : doOnError 2 error 3
2024-11-27T01:14:02.388+09:00 ERROR 320742 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : doOnError 1 error 2
2024-11-27T01:14:02.388+09:00 ERROR 320742 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : error error 2
```

앞선 케이스와 약간 차이가 생긴다.
1. 1 은 정상적으로 소비된다.
2. 2 는 내부 스트림에서 error 3 을 만든다.
	1. 내부 스트림은 덮어두고 가기로 (resume) 하고, 데이터를 없앤다. (`Mono.empty()` 를 반환)
3. 3 은 외부 스트림 (private 메서드지만 외부 스트림에 속한다!) 에서 error 2를 만든다.
4. error 2 는 외부 에러 핸들러들에 의해 처리된다.

복잡하긴 하지만 룰은 나름 간단하다. 스트림 계층간 각자 에러 처리를 할 수 있고 상위 스트림에 에러를 줄지 말지는 하위 스트림에서 결정할 수 있다.

내부 스트림에서 에러 처리를 안해버리면 어떨까? 최상위 스트림에서 그랬다면 `default onErrorDropped` 가 로깅되었던걸로 기억한다.

```java
private static Mono<Integer> monoMapper(Integer i) {  
    if (i % 3 == 0) {  
        throw new RuntimeException("error 2");  
    }  
    return Mono.just(i)  
            .map(j -> {  
                if (j % 2 == 0) {  
                    throw new RuntimeException("error 3");  
                }  
                return j;  
            });  
}

>
2024-11-27T01:20:34.225+09:00 DEBUG 323156 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : result 1
2024-11-27T01:20:34.231+09:00 ERROR 323156 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : doOnError 1 error 3
2024-11-27T01:20:34.231+09:00 ERROR 323156 --- [reactive-web] [  restartedMain] c.cenacle.edge.ReactiveWebApplication    : error error 3
```

당연히 상위 스트림에서 에러를 받아처리해준다.

이외에도 `.onError*` 시리즈들이 있긴 한데, 어떻게 에러처리를 하느냐만 다르지 어떤 스트림에서 에러처리를 하느냐는 모두 같을 것 같다. 