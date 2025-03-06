---
layout: post
title: 'Elixir Start 01: Pattern Matching'
date: 2025-03-06 20:19:00 +0900
categories: elixir fp
---

## F = ma

```
iex> a = 1
1
iex> a + 3
4
```

"a 에 1을 할당하고. 거기에 3 을 더하면 결과는 4 가 된다"

지금까지 프로그래밍에서 "="는 자연스럽게 할당(assign)으로 생각해 왔다.

하지만, 수학이나 물리학에서 "="는 그런 의미가 아니다. F = ma 를 위와같이 해석하는가?

F = ma 에서 "=" 는 "할당(assign)"이 아닌, 힘과 질량 가속도에 관한 물리현상을 정의하는 "단언(assertion)" 의 의미이다.

```
iex> a = 1
1
iex> 1 = a
1
iex> 2 = a
** (MatchError) no match of right hand side value: 1
```

Elixir 에서 "="는 "같다"라는 의미이고, match operator 라고 한다.

그러므로, 1 = a 는 a = 1 과 동일한 의미이고, 2 = a 매칭에 실패하게 된다.

조금 더 나가서 리스트의 예를 들어보자.

```
iex> list = [1, 2, 3]
[1, 2, 3]
iex> [a, b, c] = list
[1, 2, 3]
iex> a
1
iex> b
2
iex> c
3
```

Elixir 는 = 를 기준으로 좌우가 같도록 만드는 방법을 찾는다. 이런 과정을 pattern matching 이라고 한다.

```
iex> list = [1, 2, [3, 4, 5]]
[1, 2, [3, 4, 5]]
iex> [a, b, c] = list
[1, 2, [3, 4, 5]]
iex> a
1
iex> b
2
iex> c
[3, 4, 5]
```

이러한 pattern matching 은 앞서 언급했듯이 단순히 할당(assign)의 의미가 아니다 매칭(matching) 또는 assertion 의 측면에서

코드를 더욱 robust 하고 읽기 쉽게 만든다.

```
iex> list = [1, 2, 3]
[1, 2, 3]
iex> [a, 2, b] = list
[1, 2, 3]
iex> a
1
iex> b
3
iex> [a, 2, b] = list
** (MatchError) no match of right hand side value: [1, 2, 3]
```

그외, "\_"(underscore), "^"(pin operator) 와 같은 operator 도 있다.

자세한건 elixir school 이나 elixir 공식 사이트의 문서 페이지를 참고하자.

중요한것은 "="등호를 바라보는 시각의 전환

## 왜 함수형 언어를 사용하는가?

나는 "왜 함수형 패러다임으로의 전환이 필요한가?" 라는 질문에 대한 적극적인 답을 찾기 위해서라고 생각한다.

자바스크립트나 파이썬같은 지금 사용하는 언어로도 함수형 패러다임으로 개발할 수 있고, 당연히 그것이 생산성 측면에서 효율적이다.

다만, 그 과정에서 기존 관습을 유지하려는 관성이라는것이 작동하기 때문에 나는 한번쯤 Elixir나 Haskell, Closure 같은 완전 함수형 언어를 "경험"해 보는것도 좋은 방법이라 생각한다.

오늘은 pattern matching 이라는 주제만으로 가볍게 Elixir 와 함수형 언어의 맛보기만 하고 끝내겠다.

이런식으로 한숟가락씩 찍먹해보는것도 (체하지 않고) 좋은것 같기도 하고.. 핑계같기도 하고.. 그렇네.. ㅎㅎ
