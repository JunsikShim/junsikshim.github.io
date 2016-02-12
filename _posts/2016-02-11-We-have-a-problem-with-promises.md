> 이 글은 원작자인 Nolan Lawson의 허락을 받고 [We have a problem with promises](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)를 직접 번역한 글입니다.
>
> 프로미스는 이미 수년 전 부터 async 프로그래밍의 핵심으로 자리잡았지만 상당히 많은 수의 개발자들이 제대로 활용을 못하고 있습니다.
> 지금까지 프로미스를 사용하면서 잘 이해되지 않았던 점들이 많았는데, 이 글이 그런 점들을 알기 쉽게 설명해주고 있어서 공유하고자 합니다.
>
> 내용이 길어, 두 파트로 나누어서 올립니다.
> 오역이나 잘못된 부분이 있으면 알려주시면 감사하겠습니다.

자바스크립트 개발자 여러분, 이제 인정할 때입니다. 우리는 프로미스와 문제가 있습니다.
프로미스 자체 얘기가 아닙니다. 프로미스는 A+ 스펙에 정의된대로 아주 멋지죠.

저는 지난 몇 년간 상당수의 프로그래머들이 PouchDB API나 프로미스를 많이 쓰는 다른 API를 사용할 때 고생하는 모습을 자주 볼 수 있었습니다. 가장 큰 문제는 우리의 상당수가 프로미스를 제대로 이해하지 못한 채로 사용하고 있다는 겁니다.

믿겨지지 않으시다면 제가 최근에 트위터에 올린 이 퀴즈를 풀어보세요.

**질문**: 이 네 가지의 프로미스들의 차이점은 무엇일까요?

{% highlight js %}
doSomething().then(function () {
  return doSomethingElse();
});

doSomething().then(function () {
  doSomethingElse();
});

doSomething().then(doSomethingElse());

doSomething().then(doSomethingElse);
{% endhighlight %}

만약 정답을 맞추셨다면 축하드립니다. 당신은 프로미스 고수입니다. 당신은 이 글을 그만 읽을 자격이 있습니다.

나머지 99.99% 여러분은 혼자가 아닙니다. 제 트윗에 답변했던 그 누구도 정답을 맞추지 못했고, 비록 제가 작성한 퀴즈임에도 저도 세번째 결과를 보고 놀랐습니다.
정답은 이 글 가장 마지막에 있지만, 우선 왜 프로미스가 다루기 힘들고, 왜 우리가 (초보자와 전문가 모두 포함) 자꾸 실수하게 되는지를 알아봤으면 합니다.
또한, 프로미스를 쉽게 이해할 수 있도록 제가 생각하는 하나의 좋은 방법을 제시하려고 합니다.

시작하기에 앞서, 프로미스에 대한 일반적인 추정에 의문을 제기해봅시다.

### 왜 프로미스를 사용할까?

프로미스에 대한 글들을 읽어보면, Pyramid of Doom에 관한 언급을 찾을 수 있습니다. 화면의 오른편을 향해 꾸준히 늘어나는 콜백들로 가득한 끔찍한 코드를 볼 수 있죠.

프로미스는 실로 이 문제를 해결할 수 있습니다. 하지만 단순히 들여쓰기를 해결하는 게 아닙니다. 훌륭한 강연인 Redemption from Callback Hell에서 설명됐듯이, 콜백의 진짜 문제는 return이나 throw와 같은 키워드를 우리로부터 박탈한다는 것에 있습니다. 대신에 우리 프로그램의 전체 흐름은 함수가 부수적으로 다른 함수를 호출하는 Side Effect에 기반하게 됩니다.

추가로, 콜백은 더 사악한 짓을 합니다. 우리로부터 스택을 뺏어가죠. 스택은 프로그래밍 언어에서 보통 암묵적으로 제공되는 기능입니다. 스택 없이 코딩을 한다는 것은 브레이크 페달 없이 자동차를 운전하는 것과 같죠. 막상 필요해서 발을 뻗어볼 때 까지 그것이 얼마나 필요한지 모릅니다.

프로미스를 사용하는 것의 요점은, 우리가 async로 가면서 잃어버렸던 return, throw, 스택과 같은 언어의 핵심요소를 되찾는 것입니다. 하지만 활용하기 위해서는 먼저 프로미스를 제대로 사용하는 법을 익혀야 하겠죠.

### 초보자 실수들

어떤 사람들은 프로미스를 설명할 때 이런 [만화](http://andyshora.com/promises-angularjs-explained-as-cartoon.html) 또는 "그건 주고 받을 수 있는 비동기 결과값이야" 식으로 설명합니다.

하지만 전 그런 설명이 별 도움이 안 되더군요. 제가 생각하는 프로미스는 코드구조와 흐름에 대한 것입니다. 그러므로 전 흔히 하는 실수들을 리뷰하며 어떻게 고칠 수 있는지 보여드리도록 하겠습니다. 비록 지금은 초보지만 곧 프로가 될 수 있을 거라는 취지에서 전 이러한 실수들을 "초보자 실수"라고 명명하겠습니다.

### 초보자 실수 #1 : 프로미스틱한 Pyramid of Doom

프로미스 기반인 PouchDB API를 사용하는 사람들을 보다보면, 잘못된 사용패턴이 많이 보입니다. 가장 흔한 잘못은 이것입니다.

{% highlight js %}
remotedb.allDocs({
  include_docs: true,
  attachments: true
}).then(function (result) {
  var docs = result.rows;
  docs.forEach(function(element) {
    localdb.put(element.doc).then(function(response) {
      alert("Pulled doc with id " + element.doc._id + " and added to local db.");
    }).catch(function (err) {
      if (err.status == 409) {
        localdb.get(element.doc._id).then(function (resp) {
          localdb.remove(resp._id, resp._rev).then(function (resp) {
// 그 외...
{% endhighlight %}

네, 프로미스를 마치 콜백인 것 처럼 사용할 수 있습니다. 비록 손톱을 갈려고 전기 사포를 쓰는 것과 비슷하지만 가능하긴 하죠.

혹시 이런 류의 잘못이 아주 초보에게만 국한된다고 생각하신다면, 제가 이 예제 코드를 블랙베리 공식 개발자 블로그에서 가져왔다는 걸 알면 놀라실 겁니다. 콜백 함수를 쓰던 습관은 쉽게 바뀌지 않죠. *(개발자 블로그 작성자에게: 당신을 비난해서 미안하지만, 당신의 예는 유익합니다.)*

보다 나은 스타일은 이렇습니다.

{% highlight js %}
remotedb.allDocs(...).then(function (resultOfAllDocs) {
  return localdb.put(...);
}).then(function (resultOfPut) {
  return localdb.get(...);
}).then(function (resultOfGet) {
  return localdb.put(...);
}).catch(function (err) {
  console.log(err);
});
{% endhighlight %}

이렇게 프로미스를 구성할 수 있다는 점은 프로미스의 가장 강력한 점 중에 하나입니다. 하나의 함수는 그 전 프로미스가 해결되어야지만 실행되고, 그 프로미스의 결과값을 받을 수 있습니다. 나중에 더 설명하겠습니다.

### 초보자 실수 #2 : 프로미스와 forEach()를 어떻게 같이 사용하죠?

여기서 많은 사람들의 프로미스에 대한 이해가 무너지기 시작합니다. 사람들이 프로미스에다가 forEach(), for, while 등의 반복문을 사용하려고 하는 순간, 뭘 어떻게 해야할지 몰라합니다. 그래서 이런 코드를 작성하게 됩니다.

{% highlight js %}
// 모든 문서를 remove() 하고 싶다
db.allDocs({include_docs: true}).then(function (result) {
  result.rows.forEach(function (row) {
    db.remove(row.doc);
  });
}).then(function () {
  // 모든 문서가 remove() 됐을 거라고 순진하게 믿는 상태!
});
{% endhighlight %}

이 코드에 문제는 무엇일까요? 바로 첫번째 함수가 undefined를 리턴함에 있습니다. 결과적으로 두번째 함수는 모든 문서에 대해 db.remove()가 실행되기를 기다리지 않고, 아무때나 실행될 수 있습니다. 몇 개의 문서가 지워졌을 때 실행이 될지 아무도 모르죠.

이러한 버그는 PouchDB가 UI 갱신 속도보다 빨리 문서를 지울 때는 발생하지 아무런 문제점이 발생하지 않기 때문에 특히 교활합니다. 이 버그는 Race condition이나 특정 브라우저에서만 발생할 수도 있고, 디버깅하기는 거의 불가능할 것입니다.

이 얘기의 핵심은 forEach() / for / while 대신 Promise.all()을 사용하는 것에 있습니다.

{% highlight js %}
db.allDocs({include_docs: true}).then(function (result) {
  return Promise.all(result.rows.map(function (row) {
    return db.remove(row.doc);
  }));
}).then(function (arrayOfResults) {
  // 모든 문서가 확실히 remove()된 상태!
});
{% endhighlight %}

Promise.all()은 프로미스 배열을 입력받아 그 안에 있는 각각의 프로미스가 모두 해결됐을 때만 해결되는 새로운 프로미스를 리턴합니다. for의 async 버전이라고 볼 수 있습니다.

또한, 예를 들어 PouchDB에서 여러 정보를 get()하려고 할 때 Promise.all()은 매우 유용하게도 결과값이 든 배열을 그 다음 함수로 전달합니다. 또, 하위 프로미스가 하나라도 거부되면 all() 프로미스도 거부됩니다.

### 초보자 실수 #3 : catch()를 추가하지 않는 것

이것도 흔한 실수입니다. 개발자들이 자신의 프로미스가 절대로 에러를 던질 수 없다고 자신하며 코드에 catch()를 전혀 추가하지 않을 때가 많습니다. 안타깝게도 이로 인해 모든 에러가 삼켜지게 되고 콘솔에서조차 에러를 볼 수 없게 됩니다. 디버깅이 매우 힘들어지죠.

이런 불쾌한 시나리오를 피하기 위해, 전 프로미스 체인에 아래 코드를 추가하는 습관을 들였습니다.

{% highlight js %}
somePromise().then(function () {
  return anotherPromise();
}).then(function () {
  return yetAnotherPromise();
}).catch(console.log.bind(console)); // <-- 이게 강력합니다
{% endhighlight %}

에러가 절대 나지 않을 거라고 예측하는 상황에서도 catch()를 추가하는 건 늘 바람직합니다. 그 추측이 혹시 빗나가더라도 여러분의 삶이 편해지니까요.

### 초보자 실수 #4 : "deferred"를 사용하는 것

항상 발견되는 이 실수는 마치 비틀쥬스(Beetlejuice)처럼 이름을 부르는 것 만으로도 더 많이 소환될 것 같아 설명하기도 꺼려집니다.

요약해서, 프로미스는 길고 유명한 역사가 있고, 자바스크립트 커뮤니티가 제대로 정립시키기까지 긴 시간이 걸렸습니다. 초창기에는 jQuery와 Angular가 모든 곳에 "deferred" 패턴을 사용하였지만, 최근에는 ES6 프로미스 스펙을 "잘" 구현한 Q, When, RSVP, Bluebird, Lie 같은 라이브러리들로 대체되었습니다.

그러므로 만약 코드에 deferred를 타이핑하고 있다면 뭔가를 잘못하고 있는 겁니다. 그럼 어떻게 피할 수 있는 지 알아보죠.

일단, 많은 프로미스 라이브러리들은 다른 third-party 라이브러리들로부터 프로미스를 "import" 할 수 있는 방법을 제공합니다. 예를 들어, Angular의 $q 모듈은 $q.when()을 사용하여 $q로 작성되지 않은 프로미스를 감쌀 수 있습니다. 따라서 Angular 사용자들은 PouchDB 프로미스를 이렇게 사용할 수 있습니다.

{% highlight js %}
$q.when(db.put(doc)).then(/* ... */); // <-- 이렇게만 작성하면 됩니다.
{% endhighlight %}

또 다른 방법은 [revealing constructor pattern](https://blog.domenic.me/the-revealing-constructor-pattern/)을 사용해서 프로미스로 작성되지 않은 API를 감싸는 방법입니다. 콜백 기반의 함수인 노드의 fs.readFile()을 감싸려면,

{% highlight js %}
new Promise(function (resolve, reject) {
  fs.readFile('myfile.txt', function (err, file) {
    if (err) {
      return reject(err);
    }
    resolve(file);
  });
}).then(/* ... */)
{% endhighlight %}

됐습니다! 더 이상 deferred를 사용할 필요가 없습니다.

> 왜 이 것이 안티패턴인지에 대한 보충 설명은 [Bluebird wiki 페이지](https://github.com/petkaantonov/bluebird/wiki/Promise-anti-patterns#the-deferred-anti-pattern)를 참고하세요.

### 초보자 실수 #5 : return 대신 side effect를 사용하는 것

아래 코드의 문제점은 무엇일까요?

{% highlight js %}
somePromise().then(function () {
  someOtherPromise();
}).then(function () {
  // someOtherPromise()가 해결되어 있으면 좋겠네요!
  // 하지만, 아닙니다.
});
{% endhighlight %}

이 시점이 프로미스에 대해 꼭 알아야 할 것들을 설명하기에 좋아 보입니다.

이 것만 이해하고 나면 지금까지 얘기한 모든 에러를 방지할 수 있을 겁니다. 준비되었나요?

이전에 말한 것과 같이, 프로미스의 마법은 우리의 소중한 return과 throw를 돌려받는 것에 있습니다. 하지만 실제로 어떤 모습으로 존재할까요?

모든 프로미스는 then() 함수를 제공합니다. (또는 then(null, ...)을 보다 편리하게 만든 catch()). 아래 코드의 then() 함수 안으로 들어가봅시다.

{% highlight js %}
somePromise().then(function () {
  // then() 안에 있습니다!
});
{% endhighlight %}

여기서 무엇을 할 수 있을까요? 3가지가 있습니다.
1. 다른 프로미스를 리턴하기
2. sync 값(또는 undefined)을 리턴하기
3. sync 에러를 던지기

이게 전부입니다. 이 것을 이해하고나면 프로미스를 이해한 것입니다. 각각의 포인트를 차근차근 봐봅시다.

##### 1. 다른 프로미스를 리턴하기

이 패턴은 프로미스 관련 문서에서 흔히 볼 수 있습니다.

{% highlight js %}
getUserByName('nolan').then(function (user) {
  return getUserAccountById(user.id);
}).then(function (userAccount) {
  // user account를 받았습니다!
});
{% endhighlight %}

제가 두 번째 프로미스를 리턴한 것을 주목해주세요. 그 리턴은 매우 중요합니다. 만약 리턴을 명시하지 않았다면, getUserAccountById()는 side effect가 되었을 것이고, 그 다음 함수는 user account 대신 undefined를 전달받았을 겁니다.

##### 2. sync 값(또는 undefined)을 리턴하기

undefined를 리턴하는 건 보통 잘못이지만, sync 값을 리턴하는 건 사실 sync 코드를 async 코드로 변환하는 멋진 방법입니다. 예를 들어, 사용자들에 대한 메모리 캐시를 구현했다고 칩시다.

{% highlight js %}
getUserByName('nolan').then(function (user) {
  if (inMemoryCache[user.id]) {
    return inMemoryCache[user.id];    // sync 값을 리턴!
  }
  return getUserAccountById(user.id); // 프로미스를 리턴!
}).then(function (userAccount) {
  // user account를 받았습니다!
});
{% endhighlight %}

멋지지 않나요? 두번째 함수는 userAccount가 sync로 받은 결과인지 async로 받은 결과인지 상관하지 않고, 첫번째 함수는 어떤 값이든 리턴할 수 있습니다.

안타깝게도, 자바스크립트에서 값을 리턴하지 않는 함수는 엄밀히 따져서 undefined를 리턴하기 때문에 실수로 side effect가 발생하기 쉽습니다.

따라서, 전 then() 함수 안에서 무조건 return 또는 throw를 하도록 습관을 들였습니다. 여러분들도 똑같이 하기를 권장합니다.

##### 3. sync 에러를 던지기

throw와 관련해서, 이 곳이 프로미스를 더욱 더 멋있어지는 부분입니다. 사용자가 로그아웃 되었을 때 sync 에러를 던지기를 원한다고 가정해봅시다. 상당히 간단하게 구현할 수 있습니다.

{% highlight js %}
getUserByName('nolan').then(function (user) {
  if (user.isLoggedOut()) {
    throw new Error('user logged out!'); // sync 에러를 던지기!
  }
  if (inMemoryCache[user.id]) {
    return inMemoryCache[user.id];       // sync 값을 리턴!
  }
  return getUserAccountById(user.id);    // 프로미스를 리턴!
}).then(function (userAccount) {
  // user account를 받았습니다!
}).catch(function (err) {
  // 에러가 발생했습니다!
});
{% endhighlight %}

우리의 catch()는 사용자가 로그아웃 되어있을 때에는 sync 에러를 받고, 그 어떤 프로미스가 거부되었을 때에는 async 에러를 받을 것입니다. 역시, catch()는 전달받은 에러가 sync이던 async이던 상관하지 않습니다.

만약 then() 안에서 JSON.parse()를 사용했을 때 JSON이 잘못되었다면 sync 에러를 던질 겁니다. 콜백 함수를 사용할 때는 그 에러는 삼켜질 것입니다. 하지만 프로미스를 사용할 땐 간단히 catch()에서 에러 처리를 할 수 있습니다.

*[다음 글에서 이어집니다.]*
