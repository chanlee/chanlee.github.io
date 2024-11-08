---
layout: post
title: 'Fastify로 NodeJS 백엔드 갈아타기 - (1)시작'
date: 2024-11-08 19:01:00 +0900
categories: javascript nodejs fastify typescript
published: false
---

## 안녕! Fastify 잘가! Express

개발로 밥벌어 먹으면서 참 고마운 것이 있다. 바로 무료로 사용할 수 있는 오픈소스 라이브러리들이다. 그중에서도 NodeJS의 백엔드 프레임워크인 [Express](https://expressjs.com/)는 지난 10여년간 NodeJS로 백엔드를 개발할때 거의 필수적으로 사용해왔다. 구조가 단순하고 빠르게 개발할 수 있으며 검증된 다양한 미들웨어들로 경험이 누적되다보니 굳이 다른 프레임워크를 사용할 이유가 없었다.
하지만, 그동안 Express를 대채할 많은 프레임워크들이 출시되었으나 Express의 발전은 그에 반해 너무 지지부진했다고 생각한다. 최근에 회사에서 파일럿 프로젝트를 시작하며 새로운 프레임워크를 선택할 기회가 생겼고, 이 기회를 통해 Express를 대체할 새로운 프레임워크를 찾아보게 되었다.
Express보다 현대적인 구조와 훨씬 더 빠른 Benchmark 성능을 제시하는 프레임워크들이 많았지만 이번에는 [Fastify](https://fastify.dev/)를 사용해보기로 결심하고 시작하게 되었고, 시작한지 이틀만에 필요한 기능구현은 거의 끝이났다.
결론적으로, DX(Developer Experience)가 훨씬 좋다고 느꼈고, 이번 경험을 통해 Fastify가 가진 특별한 점과 개발과정을 소개해 보고자 한다.

{% highlight ruby %}
def print_hi(name)
puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]: https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
