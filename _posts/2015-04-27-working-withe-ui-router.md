---
layout: post
title: "Working withe ui router"
description: ""
category: 
tags: []
---
{% include JB/setup %}

#The Plan

So far we have a basic ui-router set up inside out AngularBonfire application. One of the design decisions we made early one, is that we are handling all our routing via Bonfire/Codeignitor and this will render most of the html which our view hooks into. Where Angular becomes really useful is within the concept of an spa (single page app), that's to say a page that is rendered using a single http request, is able to dynamically change state as the user interacts with the page, and which will sync itself with the database as required using (for the most part) ajax get and post actions.

Recapping, to make this work with codeignitor we have seperated our sites functionality into what are essentially three seperate SPA's, the front page, where eventually our business logic will live, the profile page which we started to build out in the previous sessions, but which is essentially a static page with a friend/contact button. Finally we the user account section which we are focused on today. 

Being the account section it is naturally the place that requires the most interactivity. Here the user will be able to manage their public profile, the list of interests that will be used to help people find their page, and finally to manage their contacts and messages. Eventually, once this application actually decides what it is, this page will probably need some extra feature, however we already have plenty to get on with.

The page already makes of ui-router to manage it's views, we are going to refactor this to use sref syntax instead of the custom function it currently works with. This will allow us to set an active state on the route more easily.

The interests feature is pretty much complete. The only thing we really need to add here is a delete button for our interests that we no longer have. We might also add an active toggle so users can hide interests they are currently not participating in. I haven't decided yet if a rating system for interests is appropriate but it might a useful feature to add. This section will probably be a post all of its own and not part of this current tutorial.

The profile feature should be quite simple. At a minimum we need an area for text and a file upload system. Many people indentify visually, so while I originally intended to keep the site text only, it seems appropriate to add a profile picture area at least. It would be nice if the text editor could be made to parse markdown. Finally a location could be added either by geolocation, or manually.

The social section should give links to all friends the user currently has. It should also display message threads and notifications. One day it would be nice to add an instant message capability, but this is getting ahead of ourselves. For now all we really need is a way for people to reach out to other uses and to keep track of those users they are interested in.


#What happened

So, originally we had the following inside our html

```
<a  ng-click="doRoute(route.account_route)" class="account-menu">{{route.name}}<span class="div-link-hack"></span></a>
```

Which was instantiated inside our controller

```
  // The routes as used in the menu
  $scope.routes = [
    {
      "account_route" : "interests",
      "name"          : "Interests",
      "image"         : "interests.png"
    },{
      "account_route" : "items",
      "name"          : "Items",
      "image"         : "items.png"
    },{
      "account_route" : "character",
      "name"          : "Character",
      "image"         : "character.png"
    }
  ]
    
  // Changes the current active route
  $scope.doRoute = function(actionName){
    var route = 'account_route_' + actionName
    $state.go(route)
  }
 ```

 To use this with ui-sref and take advantage of the active class we could change it to the following

```
<a href="#" ui-sref="account_route_interests" ui-sref-active="active">{{route.name}}<span class="div-link-hack"></span></a>
                    
```
So far so good, now we just to update our router defintions as follows

```
AngularBonfire.config(['$stateProvider', '$urlRouterProvider', 
    function ($stateProvider, $urlRouterProvider ) {
        
    // namespacing is important here since we extending the sidewide AngularBonfire object 
    // for instance we may need an interests route in the profile section
    var account_route_interests = { 
        name: 'account_route_interests', 
        controller: 'NgAbilityCtrl',
        views:{
            'content':{
            templateUrl: AngularBonfireUrl+'/ability/nglist'
            },
            'status':{
            templateUrl: AngularBonfireUrl+'/ability/ngstatus'
            },
            'actions':{
            templateUrl: AngularBonfireUrl+'/ability/ngactions',
            }
        }
    }

    var account_route_profile = { 
        name: 'account_route_profile', 
        views:{
            'content':{
            template: 'profile content' 
            },
            'status':{
            template: 'profile status'
            },
            'actions':{
            template: 'profile actions'
            }
        }
    }

    var account_route_social = { 
        name: 'account_route_social', 
        views:{
            'content':{
            template: 'social content' 
            },
            'status':{
            template: 'social status'
            },
            'actions':{
            template: 'social actions'
            }
        }
    }

    // Easily allows us to activate/deactive states
    $stateProvider
    .state(account_route_interests) 
    .state(account_route_profile) 
    .state(account_route_profile) 

}])
```

Finally we set inside our controller for the page

```
$state.go('account_route_profile')
```

To set a default route. So, all working now, with active states on the menu so we can style it as we see fit and room to add new routes should this become necessary. Lets add some content to the profile section.

Inside our route definition object we now need to load some dynamic templates by replacing hte template setting with a templateUrl call to a codeiginor controller. Our social, and interests sections have controllers of their own, but this one can live quite neatly under account.

```
    var account_route_interests = { 
        name: 'account_route_interests', 
        controller: 'NgAbilityCtrl',
        views:{
            'content':{
            <!-- template: 'profile content'  -->
            templateUrl: AngularBonfireUrl+'/account/ngaboutme'
            },
            'status':{
            templateUrl: AngularBonfireUrl+'/account/nglocation'
            },
            'actions':{
            templateUrl: AngularBonfireUrl+'/account/ngimage'
            }
        }
    }
```

So far our controller is quite basic with just a constructor and route to load the account/views/template.php which contains the markup for our routing.
```
// application/modules/account/controllers/account.php

    class Account extends Front_Controller
    {

        public function __construct()
        {
            parent::__construct();

            Assets::add_js('codeigniter-csrf.js');

        }

        //--------------------------------------------------------------------

        public function index()
        {

        }

        public function template() {

            $this->load->view('account/template');

        }
        
        public function ngaboutme() {

            $this->load->view('account/aboutme');

        }
        
        public function ngimage() {

            $this->load->view('account/image');

        }
        
        public function ngtemplate() {

            $this->load->view('account/location');

        }
    }

To get prepared we next need to download an external markdown library that we will use to allow people to structure there personal profile with some basic html styles https://github.com/jonlabelle/ci-markdown should be downloaded and the Markdown.php extracted to application/libraries/Markdown.php we can add it to our constructer

```
        public function __construct()
        {
            parent::__construct();

            Assets::add_js('codeigniter-csrf.js');
            $this->load->library('markdown');

        }
```

We will probably have to create a directive in order to give a live preview but again getting ahead of ourselves. First of all we need to create the model that will hold the information.

under modules/account/migrations create and account.sql file. Bonfire does use a migrations file, but for simplicity we are just going to use straight sql and create a new table references by the user_id we already have access to and use this to hold our profile,locaiont, and image url.

```
CREATE TABLE `bf_account` (
	`account_id` INT(11) NOT NULL AUTO_INCREMENT,
	`user_id` INT(11) NULL DEFAULT NULL,
	`account_profile` VARCHAR(50) NULL DEFAULT NULL COMMENT 'Chess',
	`location` VARCHAR(50) NULL DEFAULT NULL COMMENT 'enjoy, dont stratagise',
	`image_path` INT(11) NULL DEFAULT '3' COMMENT '1-5',
	`active` INT(11) NULL DEFAULT '0' COMMENT 'true or false',
	PRIMARY KEY (`account_id`),
	INDEX `Index 2` (`user_id`)
)
COLLATE='latin1_swedish_ci'
ENGINE=InnoDB
AUTO_INCREMENT=4
;
```
With that in place we next need to create account/models/account_model.php. This should a few basic things. It should be able to create a new profile. Once it's complete we will need to add a call somewhere in our users/controllers/user.php register() method with some starter data for the profile

```
    class Account_model extends MY_Model
    {

        protected $table_name   = 'account';
        protected $key          = 'account_id';
        protected $set_created  = true;
        protected $set_modified = true;
        protected $soft_deletes = true;
        protected $date_format  = 'datetime';

        /** @var array Validation rules. */
        protected $validation_rules = array(
            array(
                'field' => 'account_profile',
                'label' => 'account_profile',
                'rules' => 'max_length[250]', 
            ),
            array(
                'field' => 'location',
                'label' => 'location',
                'rules' => 'max_length[120]',
            ),
            array(
                'field' => 'image_path',
                'label' => 'image_path',
                'rules' => 'max_length[120]', 
            ),
            array(
                'field' => 'active',
                'label' => 'ability_active',
                'rules' => 'max_length[1]', 
            ),
        );

        // Accepts an array 
        public function create_user_account($data=NULL) {
            
            $this->db->insert('account', $data); 
            
            return true;

        }

        // Returns the account data for a given user id
        public function get_user_account($user_id=NULL) {
            
            $account = $this->db->
                    where('user_id', $user_id)->
                    get('account')->result();

            return $account;

        }

        public function update_image($data=NULL){

        }

        public function update_user_profile($data=NULL){

        }

        public function update_user_profile($data=NULL){

        }

}
```
Before we begin to construct our controller. I like to build things out in the way they work on the system. So before our account controller can retrieve some information, this needs to be added to the database. To make this work we need to modify application/modules/users/controllers/users.php

``` 
// first we load our account model in the constructor 
		...
        $this->load->model('users/user_model');

        // load the account model
		$this->load->model('account/account_model');
        
```
```
// then we add a call to the relevant method on the model in
        if (isset($_POST['register'])) {
            if ($userId = $this->saveUser('insert', 0, $meta_fields)) {
                // User Activation
                $activation = $this->user_model->set_activation($userId);
                $message = $activation['message'];
                $error   = $activation['error'];

                // create some boilerplate account data
                $activation = $this->account_model->create_user_account($userId);


```

Now we have the data added to our database, next we need a way to retrieve it and display it within our angular app for editing.

Back inside our modules/account/models/account_model.php

```
        // // Returns the account data for a given user id
        public function get_user_account($user_id=NULL) {
            
            $account = $this->db->
                    where('user_id', $user_id)->
                    get('account')->result();

            return $account;

        }
```

And finally inside our account/controllers/account.php

```
// we need to autoload our model within our controllers constructor
$this->load->model('account_model');
```

``        public function show(){

            $user_id = $this->current_user->id;

            $data = $this->account_model->get_user_account($user_id);

            $data = json_encode($data);
            return $data;
        }`

```

Thank goodness for that, we can now get back to our account/ng/ng-account.js file and create a service to retrieve the data

```
AngularBonfire.factory("AccountFactory", function($http, $q) {
  //this runs the first time the service is injected
  //this creates the service
  var factory = {}


  factory.show = function () {

    var deferred = $q.defer()
    
    $http.get(AngularBonfireUrl+'/account/show').then(function(resp) {
      
      deferred.resolve(resp.data)
    })

    return deferred.promise
  }

  return factory
})
  ```

And inside the same file we can also create a controller for the profile section which will retreive this data and render it onto the page

```
var AccountProfileCtrl = AngularBonfire.controller('AccountProfileCtrl', ['$scope', '$state', 
  'AccountFactory',
  function($scope, $state
    , AccountFactory
    ) {

  // and object to hold our data inside the scope
  $scope.account = 'dsfdsfd'//{};

  // $scope.abilityFormData = {};

  $scope.init = function(){
    AccountFactory.show().then(function(data) {
        console.log(data);
        $scope.account = data;
    });
  }
  $scope.init(); 
}])

```

Inside our route definition we need to tell the system which contoller for the view

```
    var account_route_profile = { 
        name: 'account_route_profile', 
        views:{
            'content':{
            templateUrl: AngularBonfireUrl+'/account/ngaboutme',
            controller: 'AccountProfileCtrl'
            },
            'status':{
            templateUrl: AngularBonfireUrl+'/account/nglocation'
            },
            'actions':{
            templateUrl: AngularBonfireUrl+'/account/ngimage'
            }
        }
    }
 ```


And finally we can check everything works so far 

```
<!-- modules/account/views/aboutme.php -->
<p>{{account}}</p>
```

As always, remember to gulp! It still surprises me when all this just works and although it's not quite as satifiying as watching a bank of tests turn green, it is quite rewarding when you add the missing closing bracket and the page renders as it should.

The only thing we need to change (since we are hacking the codeignitor route system and not using a true restful endpoint) is to change the final line of show function in the controller from return $data to eche $data;die; this is a quite and dirty solution to stop all the html from the codeignitor profiler from being returned

        public function show(){

            $user_id = $this->current_user->id;

            $data = $this->account_model->get_user_account($user_id);

            $data = json_encode($data);
            <!-- return $data -->
            echo $data; die;
        }

With that done we can clearly see the error message that ci returns inside our angular app. 'Trying to get property of non object'

What we have forgotton to do is to is make the set_current_user available in our class constructor

```
$this->load->library('users/auth');
$this->set_current_user();
```

Before going any further it's a good idea to keep our api consistent with the rest of the application so inside application/config/routes.php

```
// Account
Route::any('api/account/show', 'account/show');
```

and then inside our factory

```
$http.get(AngularBonfireUrl+'api/account/show').then(function(resp) {
```

hit f5 to run our tests :)

Finally we need to correct an error in our model (should not return an array) due to the way that the active record class returns data as an array
```
        // // Returns the account data for a given user id
        public function get_user_account($user_id=NULL) {
            
            $account = $this->db->
                    where('user_id', $user_id)->
                    get('account')->result();


            $account = $account[0];


            return $account;

        }
```






