# AngularJS Tutorial: Create DBCityFinder

Follow up this tutorial and create a City Finder app using AngularJS and DBpedia!

You can find a full working and more complete app in [DBCityFinder repository](https://github.com/alexjoverm/DBCityFinder). This repo is for the MeetUp speech [AngularJS Introduction: create DBCityFinder app](http://www.meetup.com/AlicanteTech/events/225370473/)

## DBCityFinder, step by step

### 1. Scaffold your app... in Plunkr!

[Plunkr](http://plnkr.co/edit) is a great webpage to do simple apps in a collaborative way, and share it with the people. So let's show how it's done:

* Open [Plunkr](http://plnkr.co/edit) and create an app from AngularJS1.4 template
* Add `bootstrap-css` package to have a simple css scaffolding
* Create the main **view** `main_view.html`
* Create the main **[controller](https://code.angularjs.org/1.4.8/docs/guide/controller)** `MainCtrl.js`. Then separate the controller code on `app.js` file.
* To avoid having a large controller, we can use a **[factory](https://code.angularjs.org/1.4.8/docs/guide/services)** `MainSvc.js`. They're [Singleton](https://en.wikipedia.org/wiki/Singleton_pattern) instances perfect for sharing code between controllers and sorting out!

*Follow up the links to get more info*

### 2. Route the app

Even though it is not necessary for this app, we'll do a basic routing in order to show how to use it.

* Install **[angular-route](https://docs.angularjs.org/api/ngRoute)**
* As all external packages, add module dependency on `app.js` module declaration.

```javascript
var app = angular.module('plunker', ['ngRoute']);
```

* Let's use `$routeProvider` to route our application. We need a `config` function

```javascript
app.config(['$routeProvider', function($routeProvider) {
  $routeProvider.
    when('/', {
      templateUrl: 'main_view.html',
      controller: 'MainCtrl'
    }).
    when('/about', {
      templateUrl: 'about.html'
    }).
    otherwise({
      redirectTo: '/'
    });
}]);
```

* Add a dummy `about.html` filled with some html just to check routing is working
* We need now a **Navigation bar**. **ng-include** directive comes in handy for this. It allows to load an external `html` file. Let's create a **`navbar.html`** with the following code (**notice** href starts with "#/"=:

```html
<nav class="navbar navbar-inverse">
  <div class="container">
    
    <ul class="nav navbar-nav">
      <li class="active"><a href="#/">Home</a></li>
      <li><a href="#/about">About</a></li>
    </ul>
    
  </div>
</nav>
```
* Now add the `ng-include` and `ng-view` directives onto `index.html` file. **Notice** the use of **single quotes** inside _ng-include_:

```html
<body>
  <header ng-include="'navbar.html'"></header>
  <main class="container" ng-view></main>
</body>
```

### 3. Let's add some filters!

We want to search for cities according to certain filters. We'll have 3 filters:

* **City**: cities which name contains *City*
* **Population**: population ranges
* **Map area**: cities inside a rectangular area
* They can be used or not
* They can be used combined

#### 3.1 Prepare our DBPedia Query system



#### 3.2. City filter

* Define a `search` object in your controller $scope.
* In your view, define a `section` tag for the filters
* Place the city filter by inserting an `input` tag with a `ng-model="search.city"` attribute set.

```html
<section class="filters row">
  
  <div class="col-xs-4">
    <input type="text" placeholder="City..." ng-model="search.city">
  </div>
  
</section>
```










