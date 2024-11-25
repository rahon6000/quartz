# 요약

NPE 를 피하기 위해 null 체크를 넣다보면 코드가 엄청 더러워진다.

이를 피하기 위해 Optional 과 Functional interface 를 잘 사용합시다.

# Java 의 경우

Optional 만들려면 `Optional.ofNullable()` 를 사용하면 된다. 비어있는 박스만 만들려면 `Optional.empty()` 를 사용한다.

Endpoint operation 으론 `.get()`, `.orElse()` 가 있다. `.get()` 은 `.isPresent()` 같은 것과 같이 사용하고 보통 `.orElse()` 를 사용하는 편임.

Lambda 와 같이 사용하려면 `.ifPresent()`, `orElseGet()` 을 사용한다. `orElseGet()` 은 supplier 와 같이 사용한다. `.map()` 은 연속 null check 할 때 쓸수도 있다.

# Javascript 의 경우

Null coalescense 가 매우 유용하다. (Java 엔 왜 없는지 모르겠음)

```
item?.map();
item?? "default"
```

# Python 의 경우

이 방법이 국룰이다.

```
item = s if s in not None else "default"
```

`or 'default'` 를 사용하는 방법은 antipattern 으로 여겨지기도 해서, 안전하게 간다.

Javascript 는 Java 하고 lambda 사용법이 비슷해서 따로 공부안해도 되지만, (reduce 같은 건 좀 다르긴 하지만) Python 은 모양새가 꽤 다르다. Java 와 Python 의 lambda 를 비교해보자.

```
// Function, BiFunction
Item i -> i * 2;
Item i1, Item i2 -> i1 + "," + i2;

// Consumer
Item i -> System.out.println(i);

// Supplier
() -> new Item();
```

Python 은 생긴것만 다르다.

```
// 기본 모양
lambda x: x * 2

// 기본 사용
(lambda x: x * 2)(2)
> 4

// map
list(map(lambda x: x * 2, [1,2,3]))
> [2,4,6]

// reduce 는 모듈 불러와야 한다.
from functools import reduce
reduce(lambda (x,y): y + x, 'test')
> 'tset'
```

Java 는 인수가 왼쪽, 함수가 오른쪽이지만 Python 은 반대다. 덕분에 chaining 을 하면 오른쪽에서 왼쪽으로 읽어야 되는 별로 맘에 안드는 코드가 된다. 근데 사실 왼쪽에서 오른쪽으로 읽으면 함수 기준으로 읽게 되는데, 이것도 나름 괜찮을지도 모른다.

```
result = map(
    lambda x : x + 2,
    filter(
        lambda x : x % 2 == 0,
        [1, 2, 3, 4, 5]
    )
)
print(result)
> [4,6]
```

수학에서 f(g(x)) 처럼 쓰는거랑 비슷한 방식이다.
