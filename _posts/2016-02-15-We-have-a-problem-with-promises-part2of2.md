---
layout: post
title: We have a problem with promises (Part 2 of 2)
---

> 이 글은 원작자인 Nolan Lawson의 허락을 받고 [We have a problem with promises](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)를 직접 번역한 글입니다.
>
> 프로미스는 이미 수년 전 부터 async 프로그래밍의 핵심으로 자리잡았지만 상당히 많은 수의 개발자들이 제대로 활용을 못하고 있습니다.
> 지금까지 프로미스를 사용하면서 잘 이해되지 않았던 점들이 많았는데, 이 글이 그런 점들을 알기 쉽게 설명해주고 있어서 공유하고자 합니다.
>
> 이전 글에서 부터 이어집니다.
> 오역이나 잘못된 부분이 있으면 알려주시면 감사하겠습니다.

### 숙련자 실수들

자. 이제 프로미스를 쉽게 이해할 수 있게 되었으니 특이사항들에 대해 얘기해봅시다. 그런 것들은 언제나 존재하죠.

이 실수들은 프로미스 사용에 꽤 익숙한 프로그래머들이 저지르는 것만 봤기 때문에, 전 이 실수들을 "숙련자 실수들"이라고 부르겠습니다. 제가 처음에 제시했던 퀴즈를 풀려면 이 실수들에 대해 논의해야만 합니다.

### 숙련자 실수 #1 : Promise.resolve()에 대해 모르는 것

위에서 보여드린 대로 프로미스는 sync 코드와 async 코드를 감싸는데 매우 유용합니다. 하지만, 만약 여러분이 이렇게 작성하고 있다면,

{% highlight js %}
new Promise(function (resolve, reject) {
  resolve(someSynchronousValue);
}).then(/* ... */);
{% endhighlight %}

Promise.resolve()를 사용하여 좀 더 간결하게 표현할 수 있습니다.

{% highlight js %}
Promise.resolve(someSynchronousValue).then(/* ... */);
{% endhighlight %}

이런 방식은 sync 에러를 잡는데도 매우 유용합니다. 워낙 유용해서 전 제가 작성하는 거의 모든 프로미스 API를 이런 식으로 작성하도록 습관을 들였습니다.

{% highlight js %}
function somePromiseAPI() {
  return Promise.resolve().then(function () {
    doSomethingThatMayThrow();
    return 'foo';
  }).then(/* ... */);
}
{% endhighlight %}

기억하세요: sync 에러를 발생시킬 수 있는 모든 코드는 코드 속 어딘가에서 에러가 삼켜져서 거의 디버깅이 불가능해질 때가 많습니다. 하지만 Promise.resolve()로 모든 코드를 감싸면, 나중에 언제든지 catch()할 수 있죠.

이와 유사하게, 바로 거부되는 프로미스를 리턴하는 Promise.reject()도 있습니다.

{% highlight js %}
Promise.reject(new Error('some awful error'));
{% endhighlight %}

### 숙련자 실수 #2 : catch()는 then(null, ...)과 동일하지 않다

제가 위에서 catch()는 좀 더 간편한 API라고 언급했었죠. 따라서 아래 두 코드는 동일합니다.

{% highlight js %}
somePromise().catch(function (err) {
  // 에러 처리
});

somePromise().then(null, function (err) {
  // 에러 처리
});
{% endhighlight %}

하지만, 아래 두 코드가 동일하다는 뜻은 아닙니다.

{% highlight js %}
somePromise().then(function () {
  return someOtherPromise();
}).catch(function (err) {
  // handle error
});

somePromise().then(function () {
  return someOtherPromise();
}, function (err) {
  // handle error
});
{% endhighlight %}

만약, 왜 두 코드가 동일하지 않은지 궁금하다면, 첫번째 함수에서 에러가 발생했을 때를 가정해 보세요.

{% highlight js %}
somePromise().then(function () {
  throw new Error('oh noes');
}).catch(function (err) {
  // 에러를 잡았음! :)
});

somePromise().then(function () {
  throw new Error('oh noes');
}, function (err) {
  // 에러를 잡지 못했음! :(
});
{% endhighlight %}

then(resolveHandler, rejectHandler) 포맷을 사용할 땐, rejectHandler는 resolveHandler에서 발생한 에러는 잡지 못합니다.

그래서, 전 개인적으로 then()의 두번째 인자는 절대 사용하지 않고 대신 catch()를 사용하는 습관을 들였습니다. 에러가 발생하는지를 확인하는 async Mocha 테스트를 작성할 때만 예외로 삼고 있습니다.

{% highlight js %}
it('should throw an error', function () {
  return doSomethingThatThrows().then(function () {
    throw new Error('I expected an error!');
  }, function (err) {
    should.exist(err);
  });
});
{% endhighlight %}

말이 나온 김에, [Mocha](http://mochajs.org/)와 [Chai](http://chaijs.com/)는 프로미스 API를 테스트 하기에 매우 좋은 콤비입니다. [pouchdb-plugin-seed](https://github.com/pouchdb/plugin-seed) 프로젝트에 [샘플 테스트](https://github.com/pouchdb/plugin-seed/blob/master/test/test.js)들이 들어 있습니다.

### 숙련자 실수 #3 : 프로미스 vs 프로미스 팩토리

여러 개의 프로미스를 순차적으로 실행하려고 한다고 가정해봅시다. Promise.all()과 비슷하지만 병렬로 실행하지 않는 거죠.

쉽게 생각하고 이런 식으로 작성할 지도 모릅니다.

{% highlight js %}
function executeSequentially(promises) {
  var result = Promise.resolve();
  promises.forEach(function (promise) {
    result = result.then(promise);
  });
  return result;
}
{% endhighlight %}

안타깝지만 이 코드는 의도대로 실행되지 않습니다. executeSequentially()에 넘긴 프로미스들은 여전히 병렬로 실행될 겁니다.

이렇게 되는 원인은 프로미스 배열을 가지고 작업을 했기 때문입니다. 프로미스 스펙에 의하면, 프로미스는 생성과 동시에 실행됩니다. 그러므로 여러분이 실제로 원하는 건 프로미스 팩토리의 배열입니다.

{% highlight js %}
function executeSequentially(promiseFactories) {
  var result = Promise.resolve();
  promiseFactories.forEach(function (promiseFactory) {
    result = result.then(promiseFactory);
  });
  return result;
}
{% endhighlight %}

여러분이 어떤 생각을 하고 계시는지 압니다: "이 자바 프로그래머는 대체 누구이며, 왜 팩토리에 대해 얘기하는 거지?". 프로미스 팩토리는 매우 간단합니다, 그저 프로미스를 리턴하는 함수일 뿐이죠.

{% highlight js %}
function myPromiseFactory() {
  return somethingThatCreatesAPromise();
}
{% endhighlight %}

왜 이게 작동할까요? 프로미스 팩토리는 실행될 때 까지 프로미스를 만들지 않기 때문이죠. then 함수와 사실상 동일합니다.

위의 executeSequentially()에서 result.then(...) 안의 내용이 myPromiseFactory로 대체되는 걸 상상해보시면, 여러분의 머리 속의 전구에 불이 들어올 겁니다. 그 순간, 여러분은 프로미스에 대한 깨우침을 얻게 된 것입니다.

### 숙련자 실수 #4 : 만약 프로미스 두 개의 결과를 얻고 싶다면?

종종, 하나의 프로미스는 다른 프로미스에 종속되지만, 그 두 개의 결과를 모두 갖고 싶을 때가 있습니다. 예를 들면,

{% highlight js %}
getUserByName('nolan').then(function (user) {
  return getUserAccountById(user.id);
}).then(function (userAccount) {
  // 이런, "user" 객체도 필요한데!
});
{% endhighlight %}

Pyramid of Doom을 피할 줄 아는 좋은 자바스크립트 개발자가 되기 위해 우리는 그냥 user 객체를 외부 변수로 놓을 수 있습니다.

{% highlight js %}
var user;
getUserByName('nolan').then(function (result) {
  user = result;
  return getUserAccountById(user.id);
}).then(function (userAccount) {
  // 이제 "user"와 "userAccount" 객체를 모두 얻었습니다
});
{% endhighlight %}

작동은 합니다만, 전 개인적으로 약간 불편하다고 생각합니다. 제가 권장하는 전략은 편견을 버리고 피라미드를 포용하라는 겁니다.

{% highlight js %}
getUserByName('nolan').then(function (user) {
  return getUserAccountById(user.id).then(function (userAccount) {
    // 이제 "user"와 "userAccount" 객체를 모두 얻었습니다
  });
});
{% endhighlight %}

...적어도 당분간은요. 만약 들여쓰기가 혹시 문제가 된다면, 자바스크립트 개발자들이 태곳적부터 해왔던 것을 하면 됩니다. 별도 함수로 빼는 거죠.

{% highlight js %}
function onGetUserAndUserAccount(user, userAccount) {
  return doSomething(user, userAccount);
}

function onGetUser(user) {
  return getUserAccountById(user.id).then(function (userAccount) {
    return onGetUserAndUserAccount(user, userAccount);
  });
}

getUserByName('nolan')
  .then(onGetUser)
  .then(function () {
  // 이 시점에선, doSomething()은 해결되었고, 들여쓰기는 0번째가 되었습니다
});
{% endhighlight %}

프로미스 코드가 점점 복잡해지면, 점점 더 많은 함수를 별도 함수로 빼게 될 것입니다. 전 이런 현상이 다음과 같은 매우 보기 좋은 코드를 만든다고 생각합니다.

{% highlight js %}
putYourRightFootIn()
  .then(putYourRightFootOut)
  .then(putYourRightFootIn)
  .then(shakeItAllAbout);
{% endhighlight %}

이게 바로 프로미스의 존재 이유이죠.

### 숙련자 실수 #5 : 프로미스는 통과합니다

마지막으로, 이 실수는 제가 위에 퀴즈를 냈을 때 암시했던 실수입니다. 이건 매우 특이한 케이스고, 여러분의 코드에선 전혀 등장하지 않을 수도 있지만, 저는 상당히 놀랐었습니다.

이 코드는 어떤 결과를 출력할까요?

{% highlight js %}
Promise.resolve('foo').then(Promise.resolve('bar')).then(function (result) {
  console.log(result);
});
{% endhighlight %}

만약 bar를 출력할 거라고 생각하셨다면, 잘못된 겁니다. 실제로는 foo를 출력하죠!

then()에 프로미스와 같이 함수가 아닌 값을 전달하면, then(null)로 해석이 되어서 이전 프로미스의 결과값이 통과해버립니다. 직접 테스트 해보죠.

{% highlight js %}
Promise.resolve('foo').then(null).then(function (result) {
  console.log(result);
});
{% endhighlight %}

then(null)을 얼마나 많이 붙이던, 출력값은 foo입니다.

이건 아까 제가 언급했던 프로미스 vs 프로미스 팩토리와도 연결됩니다. 요약하자면, then()에 프로미스를 직접 전달하는 건 가능하지만, 의도한 대로 작동하진 않습니다. then()은 아래와 같이 함수를 받아야만 합니다.

{% highlight js %}
Promise.resolve('foo').then(function () {
  return Promise.resolve('bar');
}).then(function (result) {
  console.log(result);
});
{% endhighlight %}

이 코드는 예상과 같이 bar를 출력합니다.

기억하세요: 언제나 then()에는 함수를 전달하기!

### 퀴즈 풀기

이제 프로미스에 대한 모든 것(거의!)을 익혔으니, 제가 처음에 냈던 퀴즈를 풀 수 있을 겁니다.

각각의 정답을 좀 더 보기 쉽게 그림으로 표현해보겠습니다.

#### 퀴즈 #1

{% highlight js %}
doSomething().then(function () {
  return doSomethingElse();
}).then(finalHandler);
{% endhighlight %}
정답:

{% highlight js %}
doSomething
|-----------------|
                  doSomethingElse(undefined)
                  |------------------|
                                     finalHandler(resultOfDoSomethingElse)
                                     |------------------|
{% endhighlight %}

#### 퀴즈 #2

{% highlight js %}
doSomething().then(function () {
  doSomethingElse();
}).then(finalHandler);
{% endhighlight %}

정답:

{% highlight js %}
doSomething
|-----------------|
                  doSomethingElse(undefined)
                  |------------------|
                  finalHandler(undefined)
                  |------------------|
{% endhighlight %}

#### 퀴즈 #3

{% highlight js %}
doSomething().then(doSomethingElse())
  .then(finalHandler);
{% endhighlight %}

정답:

{% highlight js %}
doSomething
|-----------------|
doSomethingElse(undefined)
|---------------------------------|
                  finalHandler(resultOfDoSomething)
                  |------------------|
{% endhighlight %}

#### 퀴즈 #4

{% highlight js %}
doSomething().then(doSomethingElse)
  .then(finalHandler);
{% endhighlight %}

정답:

{% highlight js %}
doSomething
|-----------------|
                  doSomethingElse(resultOfDoSomething)
                  |------------------|
                                     finalHandler(resultOfDoSomethingElse)
                                     |------------------|
{% endhighlight %}

만약 이 정답들이 여전히 이해되지 않는다면, 이 글을 다시 읽어 보시거나 doSomething()과 doSomethingElse()를 브라우저에서 직접 구현해서 실행해보시길 바랍니다.

> 해명: 이 예제들에서, doSomething()과 doSomethingElse()가 프로미스를 리턴하고, 그 프로미스들이 자바스크립트 이벤트 룹에서 벗어나서 작동한다고 가정합니다. (IndexedDB, 네트워크, setTimeout 등). 따라서 상황에 따라 병렬로 실행됩니다. [JSBin](http://jsbin.com/tuqukakawo/1/edit?js,console,output)에서 확인하실 수 있습니다.

또, 보다 고급 사용 예를 원하시면 제가 작성한 [프로미스 전문가팁](https://gist.github.com/nolanlawson/6ce81186421d2fa109a4)을 참고하세요.

### 끝으로

프로미스는 훌륭합니다. 만약 아직 콜백을 사용하고 계시다면, 프로미스로 전환하시길 강력히 권장합니다. 여러분의 코드가 작아지고, 우아해지고, 이해하기 편해질 겁니다.

저를 믿지 않으신다면 여기 증거가 있습니다. [PouchDB의 map/reduce 모듈 리팩토링](https://t.co/hRyc6ENYGC) 결과: 290 추가, 555 삭제.

그런데, 그 기존의 형편없는 콜백 코드를 작성한 사람은... 저였죠! 따라서 이 것은 프로미스의 본 파워를 느낄 수 있는 저의 첫 레슨이었고, 진행하면서 절 도와준 다른 PouchDB 기여자들께 감사를 드립니다.

하지만, 프로미스는 완벽하지 않습니다. 콜백보다 나은 건 사실이지만, 그건 주먹을 배에 맞는 게 이빨에 맞는 거 보다 낫다라는 것과 비슷하죠. 물론 한 쪽 보단 다른 한 쪽을 더 선호하겠지만, 먄약 가능하다면 둘 다 피하고 싶을 겁니다.

콜백보다 우수하지만, 프로미스는 여전히 이해하기 어렵고, 제가 이 글을 작성하게 된 것 처럼 실수하기가 쉽습니다. 초보자나 전문가 모두 종종 실수하게 되지만 사실 그건 그들의 잘못이 아닙니다. 문제는 프로미스가 우리가 sync 코드에 쓰는 패턴과 유사하고 괜찮은 대체 자원일지는 몰라도, 절대 동일하진 않다는 점입니다.

사실, 우리는 sync 세상에서 아주 손쉽게 할 수 있는 return, catch, throw, for 룹 등을 구현하기 위해 불가사의한 새로운 규칙과 API를 배울 필요가 없어야 합니다. 항상 정신을 똑바로 차려야 다룰 수 있는 두 가지의 병렬 시스템이 존재해서는 안 되죠.

### async/await을 기다리며

제가 [Taming the asynchronous beast with ES7](http://pouchdb.com/2015/03/05/taming-the-async-beast-with-es7.html)에서 ES7 async / await을 분석하며 이 키워들이 프로미스와 언어 자체를 어떻게 더 밀접하게 만드는가를 설명했던 것과 일맥상통합니다. catch와 비슷하지만 다른 catch()등의 pseudo-sync 코드를 작성할 필요 없이 ES7은 우리가 처음 코딩을 배웠을 때 처럼 진짜 try / catch / return 키워드들을 사용할 수 있게 해줍니다.

이것은 자바스크립트 언어 입장에서 매우 좋은 친구입니다. 왜나하면 결국, 우리가 실수하고 있다고 알려주는 도구가 있지 않는한 이러한 안티패턴들은 계속 자라날 것이기 때문이죠.

자바스크립트의 역사에서 예를 들면, [JSLint](http://jslint.com/)와 [JSHint](http://jshint.com/)가 [JavaScript: The Good Parts](http://amzn.com/0596517742) 보다 자바스크립트 커뮤니티에 더 큰 기여를 했다고 생각합니다. 사실 그 둘은 동일한 내용을 갖고 있는데도 말이죠. 이러한 차이는 다른 사람들이 저지른 실수에 대해 책을 읽으며 이해하려는 것과 본인이 한 실수를 그 자리에서 알려주는 것의 차이입니다.

ES7 async / await의 큰 장점은 여러분의 실수가 찾기 힘든 런타임 버그가 아닌 문법/컴파일러 에러로 나타난다는 것입니다. 하지만 그때까지는 ES5와 ES6에서 프로미스의 능력과 제대로 활용하는 법을 익히는 게 좋겠죠.

따라서, 이 글 자체도 JavaScript: The Good Parts 처럼 제한적인 영향만 줄 수 있겠지만, 이러한 실수를 저지르고 있는 사람들이 주변에 보일 때 알려줄 수 있는 글이 되었으면 합니다. 왜냐하면 아직 우리 중엔 "난 프로미스와 문제가 있어!" 라고 인정해야 할 사람이 너무나도 많기 때문이죠.

> 추가: Bluebird 3.0에선 이 글에 언급된 많은 실수에 대해 경고를 출력한다고 합니다. 따라서, Bluebird는 우리가 ES7를 기다리며 쓸 수 있는 좋은 옵션입니다!

> 역주: jQuery의 프로미스는 잘못 구현된 것으로 유명하여 절대 사용하지 않기를 권장합니다. 하지만 3.0(현재 alpha 버전)에서 드디어 Promise/A+ 스펙에 맞춰 제대로 구현되었다고 합니다.
