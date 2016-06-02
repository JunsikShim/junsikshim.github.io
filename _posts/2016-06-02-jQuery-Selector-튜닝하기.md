# jQuery Selector 튜닝하기

아마 웹 개발을 하면서 jQuery를 모르거나 안 써본 사람은 없을 겁니다. 웹 개발자들의 오랜 숙적이였던 Cross-browser 문제나 지저분한 DOM-manipulation 문제 등을 한방에 해결해주었으니 말이죠. 거기에 Ajax 지원과 각종 유틸 함수까지 포함되면서 jQuery는 웹 개발에 있어 대체 불가능한 라이브러리로 자리매김 하였습니다.

하지만 아무리 좋은 도구라도 제대로 써야 좋듯이, 고민 없이 쓴 jQuery 코드는 종종 속도저하의 주범이 됩니다. jQuery를 악용(?)하는 법은 여러가지가 있지만, 이 글에선 Selector에 관련해서 이야기하고자 합니다.

jQuery Selector란, 이미 알고 계실 것 처럼 원하는 DOM 객체를 select하는 방법입니다.

{% highlight html %}
<div id="some-div">
	<span class="some-class"></span>
    <span class="some-class"></span>
</div>
{% endhighlight %}

{% highlight js %}
$("#some-div .some-class");		// 위의 span 객체들을 리턴
{% endhighlight %}

저렇게 간단한 selector들만 있다면 문제가 될리 없겠지만, 코드를 짜다보면 다음과 같은 코드도 접하게 됩니다.

{% highlight js %}
$("#mn_report ul.tree > li > ul:eq(0)");
{% endhighlight %}

제가 작업 중인 실제 프로젝트 중 하나에서 가져온 코드입니다.
이 selector은 일단 굉장히 느립니다. 이유는 천천히 살펴보기로 하고, 일단 브라우저에서 지원하는 함수들을 살펴보겠습니다.

전통적으로 아래의 3가지 함수는 대부분의 브라우저에서 지원을 하고 있습니다.
- document.getElementById()
- document.getElementsByTagName()
- document.getElementsByClassName() (IE9+)

이 함수들은 간단한 기능만큼이나 속도가 빨라 최우선적으로 활용되어야 합니다.

또한, 기능은 강력하지만 속도는 상대적으로 느린 아래 함수들도 있습니다.
- document.querySelector() (IE8+)
- document.querySelectorAll() (IE8에선 CSS2 selector만 지원)

속도가 느리다는 건 native 기준으로 봤을 때의 이야기고, 순수 자바스크립트로 구현된 로직에 비해서는 월등히 빠릅니다. DOM 객체를 select할 때 위의 native 함수들을 최대한 사용해야 하는 건 너무나도 당연한 말일 겁니다.

그럼 jQuery는 어떻게 DOM 객체를 select할까요.

먼저 jQuery의 selector 엔진인 Sizzle의 작동 원리에 대해 살펴보겠습니다.

처음에 언급했던 코드로 예를 들면,

{% highlight js %}
$("#some-div .some-class");		// 위의 span 객체들을 리턴
{% endhighlight %}

가장 직관적인 select 방법은 some-div라는 id를 가진 element를 찾고, 그 element의 자식들 중에 some-class라는 class를 가진 element들을 찾는 것일 겁니다.

하지만, Sizzle은 정반대로 동작합니다.

먼저 some-class라는 class를 가진 element들을 찾고, 그 부모들 중에 some-div라는 id를 가진 element를 찾죠. 이 차이점은 실제 selector의 성능에 큰 차이를 가져옵니다.

이러한 HTML이 있다고 가정했을 때,

{% highlight html %}
<div class="class1">
	<div class="class2">
    	<span class="class3">select하고 싶은 span</span>
        <span class="class3">select하고 싶은 span</span>
    </div>
</div>
<span class="class3">select 하면 안 되는 span</span>
{% endhighlight %}

div 태그 안에 있는 span 태그만을 가져오는 selector는 다음과 같이 짤 수 있습니다.

{% highlight js %}
$("div.class1 div.class2 span.class3");		// (1)
$("div.class1 span.class3");		           // (2)
$("div span.class3");				          // (3)
{% endhighlight %}

위의 3가지 코드는 모두 동일한 결과를 가져오지만 속도는 아래로 갈 수록 빨라집니다.

1번의 경우,
현재 문서에 있는 모든 span 태그를 찾고, 그 중에 class3을 class로 가진 객체들을 추려낸 후, 각각의 부모들을 거슬러 올라가 class2를 가진 div객체를 찾고, 또 부모들을 거슬러 올라가 class1를 가진 div를 찾습니다.

2번의 경우엔,
1번과 동일하지만 결과에 영향을 주지 않는 div.class2를 찾지 않음으로써 속도 개선이 이루어집니다.

3번에서는 Sizzle의 특성이 나타납니다.
오른쪽에서 왼쪽으로 진행을 하기 때문에, 가장 오른쪽에 있는 selector를 구체적으로 나타내고, 왼쪽은 그 반대로 설정하는 게 좋습니다. 2번과 3번을 비교해보면, .class1의 유무를 검사하는 로직은 select 결과에 영향을 미치지 않기 때문에 생략하는 것이 바람직합니다.

사실, 위와 같은 경우들은 모던 브라우저에선 큰 문제가 되지 않습니다.
jQuery가 Sizzle을 거치지 않고 document.querySelectorAll()로 select를 해버리기 때문이죠. jQuery는 주어진 selector를 가지고 판단해서,

1. document.getElementById()
2. document.getElementsByTagName()
3. document.getElementsByClassName()
4. document.querySelectorAll()

의 순서로 실행을 합니다.

그렇다면 처음에 언급했던,

{% highlight js %}
$("#mn_report ul.tree > li > ul:eq(0)");
{% endhighlight %}

이 selector는 왜 느린 걸 까요.
그건 바로 jQuery의 확장 selector인 :eq() 때문입니다. [확장 selector 링크](https://api.jquery.com/category/selectors/jquery-selector-extensions/)

:eq()가 포함된 selector를 document.querySelectorAll()에 넣으면, CSS3 selector가 아니기 때문에 exception이 발생합니다. 그러면 jQuery는 그 selector를 Sizzle에 던지게 되고, native의 지원을 거의 받지 못한 채 자바스크립트로만 해당 로직을 수행하게 됩니다. 속도가 많이 떨어질 수 밖에 없겠죠.

하지만 그렇다고 안 쓰기에는 너무 유용한 기능들이니, 최소한의 성능 감소로 구현을 해봐야겠죠.

가장 좋은 방법은 .filter()를 사용하는 겁니다.

{% highlight js %}
$("#mn_report ul.tree > li > ul").filter(":eq(0)");
{% endhighlight %}

앞 부분의 selector는 document.querySelectorAll()로 실행할 수 있으므로 분리해서 실행하고, 확장 부분인 :eq()만 jQuery의 filter() 함수를 이용해서 속도 감소를 최소로 막을 수 있습니다.

혹시 document.querySelectorAll()를 지원하지 않는 IE6, 7에서도 성능을 내고 싶다면, 앞부분도 이런 식으로 변경하는 게 좋습니다.

{% highlight js %}
$("#mn_report").find("ul.tree > li > ul").filter(":eq(0)");
{% endhighlight %}

&#35;mn_report 객체 안에 있는 ul을 찾는 것이 목적이므로 ul을 굳이 DOM 문서 전체에서 찾을 필요가 없겠죠. 그러므로 find()를 사용해서 context를 설정해주는 것이 검색 범위를 좁혀줍니다.

또한, 아무리 selector를 튜닝해서 속도를 빠르게 만들더라도, 아예 호출하지 않는 것 보단 당연히 느립니다. 이미 한번 호출했던 selector라면 결과를 변수에 캐싱해서 재활용하는 것이 훨씬 빠르겠죠. 하지만 jQuery selector로 리턴된 객체 목록은 DOM 트리가 변경되었을 때 자동으로 갱신되지 않기 때문에 캐시 시에 유의해서 사용해야 합니다.

브라우저들의 성능이 점점 좋아지면서 selector 튜닝의 중요도가 예전만큼 크진 않은 게 사실입니다. 하지만 여전히 많은 사이트 들이 느린 selector로 성능 저하를 겪고 있고, 라이브러리의 작동원리를 어느 정도 알고 개발을 하는 게 나쁠 이유가 없으므로 간단하게나마 이 글을 작성하게 되었습니다.

감사합니다.