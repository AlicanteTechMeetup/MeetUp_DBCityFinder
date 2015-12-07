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

We can write our Query system within a [**Factory Service**](https://docs.angularjs.org/guide/services). It will allow us to place all the Query logic in there.

Here we see the Query we need. Maybe it looks a bit ugly, but the structure is quite simple, separated on **3 sections**:
* **First**: We declare the type (dbo:Settlement) and variables we need (name, abstract...)
* **Second**: We set the filters we want to apply. Note there is a **pattern sequence (`__-__`)** that we will use to replace it for our filter value. We must build this part dinamically.
* **Third**: This is just to force multilingual fields to be returned only in English. Otherwise we'd get one result for each language.

Final query string will be **First + Second + Third** section. First and Third part are constant, so we can place them into variables already.


```sparql
SELECT DISTINCT * WHERE { 
	?city rdf:type dbo:Settlement;
	dbo:wikiPageID ?id;
	rdfs:label ?name;
	dbo:abstract ?abstract;
	dbo:populationTotal ?population;
	dbo:country ?country;
	geo:lat ?latitude;
	geo:long ?longitude. 
	?country rdfs:label ?country_name. 
	
	FILTER ( ?population > __-__1 and ?population < __-__2 ).
	FILTER regex(?name, "__-__", "i").
	FILTER ( ?latitude < __-__1 and ?latitude > __-__2 and ?longitude < __-__3 and ?longitude > __-__4  ).

	FILTER langMatches(lang(?abstract), "EN"). 
	FILTER langMatches(lang(?country_name), "EN"). 
	FILTER langMatches(lang(?name), "EN")
} LIMIT 200
```
Then we just need to build it and perform the query call.


##### Steps

* Create `QuerySvc.js` and add it to `index.html`
* Add the **Query** and **Filter** variables. Also, DBPedia needs some http headers to be set, we do it on `requestConfig` variable:

```javascript
app.factory('QuerySvc', function($http) {
  
  // - Private 
    // Query vars
    var searchQueryIni = 'SELECT DISTINCT * WHERE { ?city rdf:type dbo:City; dbo:wikiPageID ?id; rdfs:label ?name; dbo:abstract ?abstract; dbo:populationTotal ?population; dbo:country ?country; geo:lat ?latitude; geo:long ?longitude. ?country rdfs:label ?country_name. ';
    var searchQueryEnd = 'FILTER langMatches(lang(?abstract), "__language"). FILTER langMatches(lang(?country_name), "__language"). FILTER langMatches(lang(?name), "__language")} LIMIT 200';

	// Filter vars
    var filterPopulation = 'FILTER ( ?population > __-__ and ?population < __-__ ). ';
    var filterCity = 'FILTER regex(?name, "__-__", "i"). ';
    var filterRectangle = 'FILTER ( ?latitude < __-__1 and ?latitude > __-__2 and ?longitude < __-__3 and ?longitude > __-__4  ). ';
  
  	// Headers for $http request
    var requestConfig = {
      headers:  {
        'Content-type': 'application/x-www-form-urlencoded',
        'Accept': 'application/sparql-results+json'
      },
      params: {
        'query': null,
        'format': 'json'
      }
    };

  
  // - Public
  var api = {};
  
  return api
});
```
* Add function to **Build the query** (in private part, after vars)

```javascript
   // params is the queryObject set on controller
   function BuildFilter(params){

      var filter = '';

      for (var property in params)
        if (params.hasOwnProperty(property)) {
          if(property == 'city' && params[property].trim() != "")
            filter += filterCity.replace('__-__', params[property].trim());
          else if (property == 'population' && params[property]){
            var aux = filterPopulation.replace('__-__1', params[property].min);
            filter += aux.replace('__-__2', params[property].max);
          }
          else if (property == 'rectangle'){
            filter += filterRectangle.replace('__-__1', params[property].ne.latitude);
            filter = filter.replace('__-__2', params[property].sw.latitude);
            filter = filter.replace('__-__3', params[property].ne.longitude);
            filter = filter.replace('__-__4', params[property].sw.longitude);
          }
        }

      return filter;
    }
```

* Finally, add the public method **Search**, to perform the actual query:

```javascript
api.Search = function(params){
    var filter = BuildFilter(params);
    requestConfig.params.query  = searchQueryIni + filter + searchQueryFin;

    $http.get("http://dbpedia.org/sparql", requestConfig).success(function (data) {
      ProcessData(data);
      $rootScope.$broadcast('QuerySvc:dataLoaded'); // Let controller now it is finished
    }).error(function () {
      $rootScope.$broadcast('QuerySvc:dataLoaded');
    });
  }
};
```

* In `.success` callback, we call `ProcessData(data)`. Let's create that function and tidy up the json returned. For now, it will just attach the data to `api.results` so then it can be accessed from the controller.

```javascript
function InsertData(data){
  self.results = data.results.bindings;
}
```

The Service is ready now.

#### 3.2. City filter

* Define a `search` object in your controller $scope.
* In your view, define a `section` tag for the filters
* Place the city filter by inserting an `input` tag with a `ng-model="search.city"` attribute set.

```html
<section class="filters row">
  
  <div class="col-xs-4">
    <input type="text" class="form-control" placeholder="City..." ng-model="search.city">
  </div>
  
</section>
```

* Search button and function!!!!!!!!!!!
* QuerySvc:dataLoaded handle in controller










