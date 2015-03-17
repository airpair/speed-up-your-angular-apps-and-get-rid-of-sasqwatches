$asqwatches (sas-kwatch) are real and they're hiding in your angular apps. To understand why, we have to step back and remember how we used to, in some cases still do, build our angular apps. Like all of you, when I first learned about angular and got my hands on it, I was in love. The ability to create your own HTML elements, not having to use jQuery, and two-way data binding changed the game.

## Your first angular app
Building our first apps with angular felt like we were using magic with all those baked in directives and DI. Before patterns and best practices evolved we just did what felt right. We wrote code like this:

```markup
<div class="card" ng-repeat="item in items | orderBy:'createdAt' | filter:filter">
 <ul>
   <li ng-repeat="action in item.actions" 
    ng-class="{active: action.active}"
    ng-mouseenter="action.active = true" ng-mouseleave="action.active = false">
    {{ action.title | uppercase }}
   </li>
 </ul>
</div>
```
This code looks fine--and in most cases it is. We're smarter and more experienced now so we're building bigger apps, but there's one problem. Our apps are slow, crawling really. Our users are suffering because we ngOverDidIt. Sure, it's fun to just jump in and ngAllTheThings, but you will soon pay for it with performance issues and it's all your fault. We're going to learn how to speed up our janky UI's and still have fun doing it, but first we'll go over why writing code like this is killing performance.

## The $digest cycle is your worst friend
I say worst friend because it is our friend. The same friend you trust all your secrets with then goes behind your back and tells everyone about them. The $digest cycle is the engine behind angular's magic. It starts at the current scope down to all the child scopes and evaluates watch expressions. When these expressions return different values than the previous `$digest`, listener functions will be called. This entire process can take forever. The more watchers you have the longer the digest will take to run. There are other performance bottlenecks in angular, but most of the time it is the `$digest` that slows things down. Here are the common ways in which most developers are triggering a $digest.
```javascript
/* 1 */
$scope.$apply() // will call a $digest at the $rootScope
/* 2 */
$scope.$digest() // will call a $digest at the current scope
/* 3 */
// using any ng-directive which calls $apply internally

```

## $asqwatches
We now know that the more watchers we have, the slower the $digest will be and that this can cause janky performance. Have you ever thought about where those watchers are coming from? Using a tool like [ng-stats](http://github.com) in development, you can now see how many watchers are active in your app. So what exactly is a $asqwatch?
* The watch(es) you did not know you were making or didn't care to optimize
* The watch(es) that came from 3rd party modules that you installed
* The watch(es) that manifested from nowhere

This proves that you're at fault for having such slow angular apps because we put those $asqwatches there.

## Speed things up
Lets go over some common use cases in angular and how we can optimize them to use less watchers and limit calls to the $digest. Before we do, look at this code below and count how many watchers you think angular will register. I asked 50 developers with at least 3 months experience with angular to count the watchers as well. The comments below are counting them like the average person did.
```javascript
angular.module('app.intro', [])
.controller('IntroController', function($scope) {

  $scope.nums = [1,2,3,4,5,6,7,8,9,10];
  
  $scope.splice = function(index){
    $scope.nums.splice(index, 1);
  };
});
```
```markup
<div id="child" class="row">
  <h3>count the $$watchers</h3>
  <div class="col s10">
    <div class="row">
    
      <div class="input-field col s9">
        <input id="filter" type="text" ng-model="search">
        <label for="filter">filter</label>
      </div>
      
      <div class="card col s3 blue-grey darken-3 demo-card"
        ng-repeat="num in nums | filter:search"
        ng-click="splice($index)">
        
        <div class="card-content" ng-class="{redText: $even}"> <!-- 10 watchers --> 
          {{ num.title }} <!-- 10 watchers -->
        </div>
      </div>
      
    </div>
  </div>
</div>
```
So how many did you count? The average person counted 20 watchers. There's an `ng-class` and a binding for each `num in nums` which is an array of 10. So that's 2 watches for each `num`. If you counted 20 , then your're as wrong as they are. There are actually 24 watchers here. The other 4 are $asqwatches, but where did they come from? `num in nums` creates a watcher on `nums` to keep an eye on changes in that collection. The `ng-model="search"` creates a watcher on the `search` model. So that brings our count to 22. The remainding two are built into the core of angular. Angular watches `$location` and `anchorScroll` right out the box.

### conditionals

Here is an common use case in angular where you might want to show & hide some html based on some expression. Count the watchers.

```markup
<div class="title col s6">
  <h3 ng-show="editMode">Edit mode</h3>
  <h3 ng-hide="editMode">View mode</h3>
</div>
      
<div ng-show="editMode" class="col s6">
  <div class="row">
    <div class="input-field col s12">
      <textarea id="textarea1" ng-model="desc"></textarea>
      <label for="textarea1">Admin stuff</label>
    </div>
  </div>
</div>
      
<div ng-hide="editMode" class="col s6">
  <div>
    <h4>message of the day</h4>
  </div>
  <div>
    <p>{{ desc }}</p>
  </div>
  <ul>
    <li ng-repeat="message in messages"> <!-- messages == [x3] -->
      {{ message }}
    </li>
  </ul>
</div>
```
If you counted 10, then you're almost right. Plus the two that angular has brings us to 12. We can do better. `ng-show` & `ng-hide` use css to hide your elements, but they are still in the DOM. This means that any watchers in those elements will be registered, even if hidden. Since we don't have any directives with DOM event listeners, we could safely use `ng-if` instead of show & hide. `ng-if` will completely remove the element from the DOM, making sure we won't have any $asqwatches lurking around. Note, using `ng-if` may cause other performance issues associated with adding and removing heavy elements from the DOM, especially on mobile. Here is our refactor using `ng-if` which brings down our watcher count.

```markup
<div class="row" ng-if="editMode">
  <div class="title col s6">
    <h3>Edit mode</h3>
  </div>
  <div class="col s6">
    <div class="card-content row">
      <div class="col s12">
        <textarea id="textarea1" ng-model="desc"></textarea>
        <label for="textarea1">Admin stuff</label>
      </div>
    </div>
  </div>
</div>
    
<div class="row" ng-if="!editMode">
  <div class="title col s6">
    <h3>View mode</h3>
  </div>
  <div class="col s6">
    <div class="card-title">
      <h4>message of the day</h4>
    </div>
       
    <div class="card-content">
     <p>{{ desc }}</p>
    </div>
    <ul>
      <li ng-repeat="message in messages"> <!-- messages == [x3] -->
        {{ message }}
      </li>
    </ul>
  </div>
</div> 
```
Now when `editMode` is true, we register only 3 watchers. When `editMode` is false, we register 7 watchers. Sure, we had to write a tad bit more HTML, but it's worth it. These small bite sized optimizations add up.

### less ng

Sure, all those awesome directives we get with angular are fun and easy to use, but they are killing your app's performance. Take a look at the example below that runs a function on `ng-mouseenter`. That function sits in a controller and shows a toast if the current element in the `ng-repeat` is `$even`. It does nothing if it is not even.

```javascript
.controller('MainController', function($scope, Toast, Cards) {
  // cards === [{ tile: 'some title' }] X 9
  $scope.cards = Cards;
  
  $scope.onMouseEnter = function(e, even) {
    if (even) {
      Toast.show('even', 500, 'toasted');
    }
  };
});
```

```markup
<!-- ng-mouseenter will cause a digest everytime -->
<div ng-repeat="card in cards" class="col s3"
ng-mouseenter="onMouseEnter($event, $even)">
  <div>
    <h5>{{ card.title }}</h5>
  </div>
</div>
```

This may seem fine, but what about when the element is not `$even`? Because we are using `ng-mouseenter`, a `$digest` will be called. Angular wraps our expression, `onMouseEnter($event, $even)`, inside a `$scope.$apply()` which calls a `$digest`. A `$digest` is pointless in this case for a few reasons. We're not changing any models in our `onMouseEnter()` function, and we for sure don't want a call to `$digest` if the element is not `$even`. What if you already had a few thousand watchers, calling a digest everytime we mouseenter will kill your app's performace. Lets speed this up by making our own directive and using DOM events.

```javascript
.directive('onMouseEnter', function(){
  return function(scope, ele, attr) {
      // no $digest here, just some jq
    ele.on('mouseenter', function(){
      if (scope.$even) {
        // does not trigger $digest
        // if we're modifying the scope
        // then we can wrap in $apply
        scope.$eval(attr.onMouseEnter);
       }
    });
  }
});
```
```markup
<div ng-repeat="card in cards" class="col s3"
on-mouse-enter="onMouseEnter($event, $even)">
  <div class="card-title">
    <h5>{{ card.title }}</h5>
  </div>
</div>
```

Now we are in control of when a `$digest` will be called by using `jqLite`. Using `$eval`, angular will evaluate the expression attached to the the `onMouseEnter` attribute, our function. This approach will not trigger a single call to `$digest` and will still show our toasts on `$even` elements. If we did want to change a model in our `onMouseEnter` function, then we could just call `$apply` ourselves in our directive which would only trigger a `$digest` on `$even` even elements.

### unwatching

We manually register watchers using `$scope.$watch|$watchGroup|$watchCollection` . These very common and very powerful. Sometimes, however, we just want to do something once when our expression changes and nothing more.

```javascript
$scope.number = 0;
  
  $scope.$watch('number', function(fresh, stale){
    if (fresh !== stale && fresh > 10) {
      Toast.show('updated', 800, 'toasted');
    }
  });
  
  $scope.random = function(){
    $scope.number = _.random(0, 12);
  };
```
```markup
<div class="col s4">
  <a class="btn" ng-click="random()">random</a>
</div>
      
<div class="col s2">
  <h3>{{ number }}</h3>
</div>
```
Whenever `$scope.number` is higher than 10, we show a toast notification. We only want to do this once, though. However, the toast will continue to show everytime `$scope.number` goes over 10 by clicking the button that assigns a random number. We should be responsible and clean up after ourselves and unwatch our watchers when we don't need them anymore, especially if our listener functions are taking forever to process, clogging up the `$digest`. Angular provides a simple way to cancel our watchers.
```javascript
var cancel = $scope.$watch('number', function(fresh, stale){
    if (fresh !== stale && fresh > 10) {
      Toast.show('updated', 800, 'toasted');
      cancel(); // will cancel this watcher
    }
  });
```
The `$watch` methods on the scope all return a function that when called will cancel it so the `$digest` won't process it anymore. Pretty easy and awesome to use right now.

### filters
I don't have a code sample here for filters because you just need to see it. Head over to the [ng-stats Demo](http://kent.doddsfamily.us/ng-stats/) and play with the filters for a minute. Start with no delay on the filter, then increase it to just 1ms and start hovering over the list items. Notice the jank! Angular now uses stateless filters for better performance (previously, these were a major source of slowness).

## Conclusion

Performance is hard, very hard, but if you commit to some of these techniques you can avoid huge bottlenecks in angular. So here are some other tips

* Don't use any `ng-[DOM EVENT]` directives like `ng-mousemove` and `ng-keydown` They're gone on Angular 2 anyway.
* Use `ng-repeat` responsibly.
* Keep your filters stateless unless you absolutely cannot.
* If your filters in `ng-repeat` take longer than 1ms, it won't scale, so avoid filters all together or keep them fast.
* Watch your watchers, $asqwatch creep can happen fast.




