# tech-express


Tech voting application with Express JS

This application was written for a Hands on lab about the MEAN Stack.

You can test this application by cloning the repository then running the following commands (assuming npm and bower are already installed or see step 1)

```shell
npm install
cd public
bower install
cd ..
npm start
```

There are some faster generator, that can help you to create CRUD and other functionnalities, but the main aim of this Hands on Lab is to understand the technologies, so we will do many things by hand.

There is a tag for each step of the process.

## Step 1 : Installation of the tools
You need to have installed [Node](http://nodejs.org/) in order to have npm.
Then you install the express-generator

```shell
npm install -g express-generator
```
For the Angular part of the application, you will need bower :

```shell
npm install -g bower
```


## Step 2 : Initialisation of the project
Use the express generator to create the projet.

```shell
express --ejs tech-express
cd tech-express
````
the ejs option is to use standard HTML for the template engine. By default, the express generator use Jade.

## Step 3 : npm modules installation
Use the following command : 
 
 ```shell
 npm install
 ```
 It will install the dependencies declared in the ```packages.json``` file and store them in the ```node_modules/``` directory. You can see that it manages transitive dependencies.
You can then testing the application by using the command 

```shell
npm start
```
and opening a browser on [http://localhost:3000](http://localhost:3000) 
 
 
## Step 4 : creating a first route
In the route directory, you create a file ```techs.js`` with the code : 

```JavaScript
var express = require('express');
var router = express.Router();

router.get('/', function(req, res, next){
	res.json([{name:'JavaScript', votes:3}, {name:'Java', votes:2}]);
});

module.exports = router;
``` 
then you need to register the route in the app.js file by adding at the beginning of the file :

```JavaScript
var techs = require('./routes/techs');
```
and at the definition of the routes : 
 
```JavaScript
app.use('/techs', techs);
``` 

You need to restart the server to see the route working on the url [http://localhost:3000/techs](http://localhost:3000/techs)

## Step 5: enhancing the development process
In order not to restart the server to see our modifications, we can use a tool called [nodemon](https://github.com/remy/nodemon) that will watch the directories and restart the server on save. To install nodemon, you use npm

```shell
npm install -g nodemon
```
then at the project root : 

```shell
nodemon
```

## Step 6: Adding new Techs
In order to be able to add new techs, we need to be able to store them. For the moment, we will store them in a variable in the techs module.
Change the techs.js file as follow : 

```JavaScript
var express = require('express');
var router = express.Router();

var techs = [{name:'JavaScript', votes:3}, {name:'Java', votes:2}];

router.get('/', function(req, res, next){
	res.json(techs);
});

module.exports = router;
```
then you add a middleware to manage the post request : 

```JavaScript
router.post('/', function(req, res, next){
	techs.push(req.body);
	res.status(201).json(req.body);
});
```

You can test your post route with a Rest Client or the command line : 

```shell
curl -i -d '{"name":"Mongo", "votes":0}' -H "Content-Type: application/json" http://127.0.0.1:3000/techs
```

## Step 7: managing the vote
We need to be able to vote for a tech by sending POST query on the URL like ```http://application/techs/{id}/vote```

First we update the data model to have id.

```JavaScript
var techs = [{id: 1, name:'JavaScript', votes:3}, {id: 2,name:'Java', votes:2}];
```

Then we add a middleware to handle the query

```JavaScript
router.param('techId', function(req, res, next, techId){
	console.log('param '+ techId);
	var tech = null;
	for (var i = techs.length - 1; i >= 0; i--) {
		if(techs[i].id == techId){
			req.tech = techs[i];
			break;
		};
	};
	return next();
});

router.post('/:techId/vote', function(req, res, next){
	if(req.tech === null){
		console.log('not found');
		res.status(404).json({message:'not found'});
	}
	req.tech.votes = req.tech.votes +1;
	res.json(req.tech);
});
```

The first method is used to extract the param from the URL and add the searched tech to the req object.
The second one is used to to treat the request.

## Step 8: adding persistance : Configuration
First we add [mongoose](http://mongoosejs.com/) to the project.

```shell
npm install --save mongoose
```
the ```--save``` option is used to add the dependency to the package.json file.
Then in the ```app.js``` file we add the connection to mongoDB

```JavaScript
var mongoose = require('mongoose');
mongoose.connect('mongodb://localhost/techs');
```

## Step 9: Creation of a model
You need to create a model for the object that will be persisted in the mongo database.
You create a directory models and a file ```Tech.js``` with the following code : 

```JavaScript
var mongoose = require('mongoose');
var TechSchema = new mongoose.Schema({
	name: String,
	votes: {type:Number, default:0}
});

mongoose.model('Tech', TechSchema);
```
In the ```app.js``` file, after the ```mongoose.connect(...)``` you need to add the require for the model : 

```JavaScript
require('./models/Tech');
```
**This part of code need to appear before the require for the routes.**

##Step 10: Using the model

Finally in the routes for the techs in the file ```./routes/techs.js``` you require mongoose and the model to be able to use them : 

```JavaScript
var mongoose = require('mongoose');
var Tech = mongoose.model('Tech');
```

Then in the tech route file, we can delete the local model to use the mongoDB in middleware :

```JavaScript
router.get('/', function(req, res, next){
	Tech.find(function(err, techs){
		if(err){return next(err);}
		res.json(techs);
	})
});

router.post('/', function(req, res, next){
	var newTech = new Tech(req.body);
	newTech.save(function(err, tech){
		if(err){return next(err);}
		res.status(201).json(tech);
	})
});
```
## Step 11: Handling the vote
To handle the vote, you can add methods in the model.

```JavaScript
TechSchema.methods.vote = function(cb){
	this.votes +=1;
	this.save(cb);
};
```
 
Then in the middleware, you can use this method :

```JavaScript
router.param('techId', function(req, res, next, techId){
	var query = Tech.findById(techId);

	query.exec(function(err, tech){
		if(err){return next(err);}
		if(!tech){res.status(404).json('no tech for this id');}
		req.tech = tech;
		return next();
	});
});

router.post('/:techId/vote', function(req, res, next){
	if(req.tech){
		req.tech.vote(function(err, tech){
			if(err){return next(err);}
			res.status(201).json(tech);
		});	
	}
	
});
```

## Step 12: Preparing the front end part
In the public directory, you use the command 

```shell
bower init
```
and answer the different questions. Then you add the dependency for angular using : 

```shell
bower --save install angular
```
After this, you can edit the ```index.ejs``` file in the views directory to set the code : 

```HTML
<html>
  <head>
    <title>Tech Express</title>
    <script src="/bower_components/angular/angular.js"></script>
    <script src="/javascripts/app.js"></script>
  </head>
  <body ng-app="TechApp" ng-controller="MainCtrl">
    <div>
      {{test}}
    </div>
  </body>
</html>
```
and you create the file ```./javascripts/app.js``` with the code :

```JavaScript
angular.module('TechApp',[])
	.controller('MainCtrl', ['$scope',function($scope){
		$scope.test = 'Hello from Angular';
	}]);
```

You then should see your application at the url [http://locahost:3000](http://localhost:3000).

## Step 13: Showing a list of tech
In the index.ejs file, you create the template to show the list of tech.

```HTML
<html>
  <head>
    <title>Tech Express</title>
    <script src="/bower_components/angular/angular.js"></script>
    <script src="/javascripts/app.js"></script>
  </head>
  <body ng-app="TechApp" ng-controller="MainCtrl">
  	<H1> Welcome to Tech Express </H1>
    <div>
      <ul>
      	<li ng-repeat="tech in techs">{{tech.name}} - votes : {{tech.votes}}</li>
      </ul>
    </div>
  </body>
</html>
```
and add some data in the controller : 

```JavaScript
angular.module('TechApp',[])
	.controller('MainCtrl', ['$scope',function($scope){
		$scope.techs= [{name:'express', votes: 4},{name:'mongoDB', votes:3}];
	}]);
```

## Step 14: Connecting to the backend

To query the backend, we will use angular-resource, a module to be used with rest resources.
In the ```public``` directory add the dependency with the commande : 

```shell
bower --save install angular-resource
```
Add then the import for the new js file in ```index.ejs``` : 

```HTML
    <script src="/bower_components/angular-resource/angular-resource.js"></script>
```

then in the ```./public/javascripts/app.js``` file change the code as follow : 

```JavaScript
angular.module('TechApp',['ngResource'])
	.controller('MainCtrl', ['$scope', 'Techs',function($scope, Techs){
		$scope.techs=Techs.query();
	}])
	.factory('Techs', ['$resource', function($resource){
		return $resource('/techs/');
	}]);
```

These modifications will : 
 - add the dependency to the module ngResource
 - create a service that will return a resource for Techs
 - add the result of the call to ```/tech``` with GET to the scope.

## Step 15: submitting tech
You create a form in the ``ìndex.ejs```page

```HTML
    <div>
	    <form ng-submit="addTech()">
		    <input type="text" placeholder="Tech" ng-model="newTechName"></input>
		    <br>
		    <button type="submit">Add</button>
	    </form>
    </div>
```
And then you add the function in the controller : 

```JavaScript
$scope.addTech = function(){
	var newTech = new Techs({name:$scope.newTechName});
	newTech.$save();
	$scope.techs.push(newTech);
	$scope.newTechName='';
}
```

## Step 16: adding the vote
Add a vote button at the end of the tech lines : 

```HTML
    <li ng-repeat="tech in techs">{{tech.name}} - votes : {{tech.votes}}  <button ng-click="voteFor(tech)">Vote for me</button></li>
 ```
 and then edit the ```app.js``` file to add the voteFor method to the resource and manage the button click in the controller : 
 
 ```JavaScript
 angular.module('TechApp',['ngResource'])
	.controller('MainCtrl', ['$scope', 'Techs',function($scope, Techs){
		$scope.techs=Techs.query();
		$scope.addTech = function(){
			var newTech = new Techs({name:$scope.newTechName});
			newTech.$save();
			$scope.techs.push(newTech);
			$scope.newTechName='';
			console.log(newTech);
		};
		$scope.voteFor=function(tech){
			var foo = tech.$voteFor().then(function(){
				$scope.techs=Techs.query()
			});
		};
	}])
	.factory('Techs', ['$resource', function($resource){
		return $resource('/techs/:techId',{techId:'@_id'},{
			'voteFor':{method:'POST', url:'/techs/:techId/vote'}
		});
	}]);
```
   

