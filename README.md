# AngularJS Tutorial: Create DBCityFinder

*Follow up this tutorial and create a City Finder app using AngularJS and DBpedia!*

Hi! I'm Alex Jover, nice to meet you! We're creating an AngularJS app called DBCityFinder step by step (yes, this app searches cities, as the name suggests). The aim of this guide is to create a **practical app** for any **AngularJS beginner who wants to get a bit deeper into AngularJS**

You can find a full working and more complete app in [DBCityFinder repository](https://github.com/alexjoverm/DBCityFinder). This repo is intended to be a full-guide for the MeetUp speech [AngularJS Introduction: create DBCityFinder app](http://www.meetup.com/AlicanteTech/events/225370473/)

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
SELECT DISTINCT ?city
  (SAMPLE(?name) as ?name)
  (SAMPLE(?abstract) as ?abstract)
  (SAMPLE(?country_name) as ?country_name)
  (MAX(?population) as ?population)
  (MAX(?latitude) as ?latitude)
  (MAX(?longitude) as ?longitude)
WHERE { 
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
app.factory('QuerySvc', function($http, $rootScope) {
  
  // - Private 
    // Query vars
    var searchQueryIni = 'SELECT DISTINCT ?city (SAMPLE(?name) as ?name) (SAMPLE(?abstract) as ?abstract) (SAMPLE(?country_name) as ?country_name) (MAX(?population) as ?population) (MAX(?latitude) as ?latitude) (MAX(?longitude) as ?longitude) WHERE { ?city rdf:type dbo:Settlement; rdfs:label ?name; dbo:abstract ?abstract; dbo:populationTotal ?population; dbo:country ?country; geo:lat ?latitude; geo:long ?longitude. ?country rdfs:label ?country_name.';
    var searchQueryEnd = 'FILTER langMatches(lang(?abstract), "EN"). FILTER langMatches(lang(?country_name), "EN"). FILTER langMatches(lang(?name), "EN")} LIMIT 200';

	// Filter vars
    var filterPopulation = 'FILTER ( ?population > __-__1 and ?population < __-__2 ). ';
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

Notice the line `$rootScope.$broadcast('QuerySvc:dataLoaded');`. This will allow us to let the controller now the data is received and processed by sending an event.

* In `.success` callback, we call `ProcessData(data)`. Let's create that function and tidy up the json returned. For now, it will just attach the data to `api.results` so then it can be accessed from the controller.

```javascript
function ProcessData(data){
  api.results = data.results.bindings;
}
```

The Service is ready now.

#### 3.2. City filter

* Define a `search` object in your controller $scope.
* In your view, define a `section` tag for the filters
* Place the city filter by inserting an `input` tag with a `ng-model="search.city"` attribute set. Also let's set the **Submit button** and add `ng-click="Search()"`. 

```html
<section class="filters row form-group">
  <div class="col-xs-4">
    <input type="text" class="form-control" placeholder="City..." ng-model="search.city">
  </div>
</section>


<section class="submit row form-group text-center">
    <button class="btn btn-primary btn-lg" ng-click="Search()">Search</button>
</section>
```

* Implement **Search** function on the controller. Remember that we already created the `QuerySvc.js` with a `Search(params)` function, so let's make use of it. 
 * First, inject `QuerySvc` into the controller
 * Create the `Search` function. It must call `Search` Service function and pass it `search` object.

```javascript
$scope.Search = function(){
  var params = angular.copy($scope.search);
  QuerySvc.Search(params);
}
``` 

* We need to handle `QuerySvc:dataLoaded` event on the controller, so let's add the next code within the controller:

```javascript
$scope.Search = function(){
  var params = angular.copy($scope.search);
  QuerySvc.Search(params);
}
``` 

* Finally, we just need to visualice the results. **`ng-repeat`** comes in handy for the purpose. Also we want to show it only when results are available, use `ng-show="results && results.length"` for that. Let's go to `main_view.html` and add the following code:

```html
<section class="results" ng-show="results && results.length">
  <table class="table table-hover">
    <thead>
      <tr>
        <th>City</th>
        <th>Country</th>
        <th>Population</th>
      </tr>
    </thead>

    <tbody>
      <tr ng-repeat="city in results" ng-click="ShowAbstract(city.abstract.value)">
        <td>{{ city.name.value }}</td>
        <td>{{ city.country_name.value }}</td>
        <td>{{ city.population.value }}</td>
      </tr>
    </tbody>
  </table>
</section>
```

* We are showing the abstract in the `ng-click="ShowAbstract(city.abstract.value)"`. Let's go to the controller and add the function:

```javascript
$scope.ShowAbstract = function(abstract){
  alert(abstract);
}
```

#### 3.3. Population filter

Most of previous steps were generic for all the filters we want to add. This won't take too much for that reason.

* Take a look at [how to work with select tag in AngularJS](https://docs.angularjs.org/api/ng/directive/select#using-select-with-ngoptions-and-setting-a-default-value). We are using `ng-options`, since it is more simple, flexible and fast. Let's add another filter in the *filter section* on **main_view.html**:

```html
  <div class="col-xs-6">
    <select class="form-control" ng-model="search.population" ng-options="pop.text for pop in population">
      <option value="">-- Population --</option>
    </select>
  </div>
```

Notice we also have set `ng-model="search.population"` which is the actual filter. Also we added `<option value="">-- Population --</option>`, this is to don't force the user to select the filter.

* Now add the population array to the controller, since `ng-options` is making use of it:

```javascript
$scope.population = [
  {
    text: '0 - 5,000',
    min: 0,
    max: 5000
  },
  {
    text: '5,000 - 100,000',
    min: 5000,
    max: 100000
  },
  {
    text: '> 100,000',
    min: 100000,
    max: 9999999999
  }
];
```

That's all for this one.

 

### 4. Final step! Map representation and rectangle filter

In this final step we wanna show all the results locations on a map. For that we'll use [Angular Google Maps](https://angular-ui.github.io/angular-google-maps/#!/), a very powerful set of AngularJS Directives to work with Google Maps and part of [Angular-ui](https://angular-ui.github.io/).

* Do following steps (For more info see [Quick Start](https://angular-ui.github.io/angular-google-maps/#!/use)):
 * Add `angular-google-maps` in Plunkr, and its dependencies (`lodash`, `angular-simple-logger`)
 * Add `'uiGmapgoogle-maps'` to module dependencies
 * Add **css rule** `.angular-google-map-container { height: 200px; }`
 * Config the provider, by adding following code to the end of `app.js`

```javascript
app.config(function(uiGmapGoogleMapApiProvider) {
    uiGmapGoogleMapApiProvider.configure({
        //    key: 'your api key',
        v: '3.20', //defaults to latest 3.X anyhow
        libraries: 'geometry,visualization'
    });
})
```

* We need to add now the **Map, Markers, Windows and Rectangle directives** in `main-view.html`:
 * Map ([official docs](https://angular-ui.github.io/angular-google-maps/#!/api/google-map)): the same than Quick Start guide but adding events.
 * Markers ([official docs](https://angular-ui.github.io/angular-google-maps/#!/api/markers)): use `ui-gmap-markers` directive. 
 * Windows ([official docs](https://angular-ui.github.io/angular-google-maps/#!/api/windows)): use `ui-gmap-windows` directive inside `ui-gmap-markers` for showing a window with the city name on click event. See official docs example.
 * Rectangle ([official docs](https://angular-ui.github.io/angular-google-maps/#!/api/rectangle)): this will be the filter. We need to create it on map click, set it's dimensions and show/hide it. 

```html
<ui-gmap-google-map center='map.center' zoom='map.zoom' events='map.events'>
  <ui-gmap-markers models="markers" coords="'self'" click="onClick()" fit="true">
    <ui-gmap-windows show="'show'">
      <div ng-non-bindable>{{title}}</div>
    </ui-gmap-windows>
  </ui-gmap-markers>

  <ui-gmap-rectangle visible="rectangle.show" bounds="rectangle.bounds" fill="'#FFFFFF .5'" draggable="true" editable="true">
  </ui-gmap-rectangle>
</ui-gmap-google-map>
```

#### 4.1. Creating loggic on MapsSvc

As per previous google-maps directives, we need **map, markers and rectangle** objects, and some additional logic. We can create them directly on the Controller, but better do it on a Service to have a cleaner code.

What we want to accomplish is:
* Create **markers** from the results locations
* Show a **popup** with the city name **on click event**
* Create a **rectangle filter area**

##### Steps

* Create `MapsSvc.js` file, and add it to `index.html`
* Scaffold the service and add the **map** and **rectangle** objects

```javascript
app.factory('MapsSvc', function() {
 
  // PUBLIC
  var api = {}
  
  api.map = { 
    center: { latitude: 45, longitude: -73 }, 
    zoom: 8 
  };
  
  api.rectangle = {
    bounds: {
      sw: {
        latitude: 47.46,
        longitude: 13.07
      },
      ne: {
        latitude : 47.76,
        longitude: 13.77
      }
    },
    show: false
  };
  
  return api
});
```
* Add **CreateMarker** and **CreateRectangle** functions to the Service api object

```javascript
api.CreateMarker = function(id, lat, lng, title){
  return {
    id: id,
    latitude: lat,
    longitude: lng,
    title: title,
    show: false,
    onClick: function() { this.show = !this.show; }
  }
}

api.CreateRectangle = function(map, rectangle){
  if(!rectangle.show){
    // Create rectangle centered, and within the bounds of the map
    var xMin = map.getBounds().getSouthWest().lng();
    var xMax = map.getBounds().getNorthEast().lng();
    var yMin = map.getBounds().getNorthEast().lat();
    var yMax = map.getBounds().getSouthWest().lat();

    // Set the rectangle in the middle of the map
    var factor = 0.2; // 20% margin to the bounds
    rectangle.bounds.ne.latitude = yMin + ((yMax - yMin) * factor);
    rectangle.bounds.sw.latitude = yMin + ((yMax - yMin) * (1 - factor));
    rectangle.bounds.sw.longitude = xMin + ((xMax - xMin) * factor);
    rectangle.bounds.ne.longitude = xMin + ((xMax - xMin) * (1 - factor));

    rectangle.show = true;
  }
}
```

#### 4.2. Using logic on the Controller

We have `MapsSvc` created, but we still must do some things!
* First of all, we must **create all markers** when we get the data
* Get map and rectangle from service (we still don't have it on the controller scope)
* Implement **map click event** to create rectangle, and button to remove it

##### Steps

* Go to `MainCtrl.js` and reference `MainSvc`
* Get **map **and** rectangle** from `MainSvc`

```javascript
  $scope.map = MapsSvc.map;
  $scope.rectangle = MapsSvc.rectangle;
  $scope.markers = [];
```

* Implement **map click event**. We need to call `$scope.$apply()` since we are modifying `$scope.rectangle` in a callback not assigned to `$scope`:

```javascript
$scope.map.events = {
  click: function(map){
    MapsSvc.CreateRectangle(map, $scope.rectangle)
    $scope.$apply();
  }
}
```

* Inside `$scope.$on('QuerySvc:dataLoaded'`, add following code to **create markers**:

```javascript
$scope.markers = [];
if($scope.results){
  $scope.results.forEach(function(result){
    var marker = MapsSvc.CreateMarker($scope.markers.length, +result.latitude.value, +result.longitude.value, result.name.value);
    $scope.markers.push(marker);
  })
}
```

* Let's adapt `$scope.Search` function to add the rectangle to the `params` variable. The final code should look like:

```javascript
$scope.Search = function(){
  var params = angular.copy($scope.search);
  if($scope.rectangle.show) {
    if (params)
      params.rectangle = $scope.rectangle.bounds;
    else // First filter
      params = { rectangle: $scope.rectangle.bounds };
  }
  MapsSvc.Search(params);
}
```

* Finally, we need a **button** to hide the rectangle. Go to `main_view.html` and add the following button after **Search button**:

```html
<button class="btn btn-danger" ng-show="rectangle.show" ng-click="rectangle.show = false">Remove rectangle</button>
```

#### 5. We are done! Why not adding something cool?

The best thing of AngularJS is that you have a wide range of **third-party libraries** to work with it. We already used **[Angular Google Maps](https://angular-ui.github.io/angular-google-maps/#!/)**, but there is a lot of them out there!

As an example, let's see how easy is to add a **[loading bar](http://chieffancypants.github.io/angular-loading-bar/)**, like youtube's one:

* Install `angular-loading-bar` from Plunkr (be sure script is placed after `angular.js` script on `index.html`)
* Add the module `angular-loading-bar` to modules dependencies

That's all, not kidding! You can also customize it by adding some `.css`, for example:

```css
#loading-bar .bar {
  height: 4px;
}
#loading-bar-spinner .spinner-icon{
  border-width: 4px;
}
```
























