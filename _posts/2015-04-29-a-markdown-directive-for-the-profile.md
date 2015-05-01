---
layout: post
title: "A markdown directive for the profile"
description: ""
category: 
tags: []
---
{% include JB/setup %}

#The plan 

Our user needs a way to update their public profile. This would normally be a textarea. Plain text is boring though, and traditional wysiwg editors are not very fun for anyone. What our users being totally modern disaffected and disatified with technology want is markdown, they don't know it yet,well some of them that work in tech and keep blogs do, but everyone else will have a great time with it. They will have even more of a great time if they can see what they are writing update in real time. 

This should be a relatively straightforward feature, with a textarea, a div to render the content and a small directive to make the translation. On the server side we aleady have the structure built out, all we really need is to create an extra model/controller to save the changes when the user is happy with them.

#What happened

First off, we should write some html so we can kinda see what we are aiming for. We already have a route set up to render the template with a controller
```
    var account_route_profile = { 
        name: 'account_route_profile', 
        controller: 'NgAccountCtrl',
        views:{
            'content':{
            templateUrl: AngularBonfireUrl+'/account/ngaboutme',
            controller: 'AccountProfileCtrl'
            },
```

Our profile controller looks as below
```
var AccountProfileCtrl = AngularBonfire.controller('AccountProfileCtrl', ['$scope', '$state', 
  'AccountFactory',
  function($scope, $state
    , AccountFactory
    ) {

    $scope.account = {}
  $scope.init = function(){
    AccountFactory.show().then(function(data) {
        console.log(data);
        $scope.account = data;
    });
  }
  $scope.init(); 

}])
```
And  ngaboutme.php 
```
<div class="row">
	<div class="col-6 col-desktop-6">
		<textarea class="markdown-edit" ng-model="account.account_profile">
		
		</textarea>
		

	</div>
	<div class="col-6 col-desktop-6">
		{{account.account_profile}}
	</div>


</div>
```
Now we have two way direction binding between the two. Try it, it's quite fun watching the two sides keep in sync as you type.

Let's get the uninteresting bit out of the way next since the rest of the feature shouldn't be too complicated and it will be difficult to motivate ourselves to write anymore when it's complete. 

Still within ng-account.js we need to add a new property method to our factory which is a angular service that acts as a bridge between our rest server and our angular vc (View/Controller) 

```
AngularBonfire.factory("AccountFactory", function($http, $q) {
  //this runs the first time the service is injected
  //this creates the service
  var factory = {}

	// other methods

  factory.updateProfile = function (id, dataObject) {

    var deferred = $q.defer()

    var post_data = {
      'account_profile' : dataObject, 
      'ci_csrf_token'   : ci_csrf_token()
    }
  
    // so far we have an object we can 'POST' to our form which contains a security token
    $.post(AngularBonfireUrl+'/api/account/updateprofile', post_data).done(function(sdf){
        console.log('saved', sdf)
        deferred.resolve('done')
    })

    return deferred.promise

  }

  return factory
})

```
We also need to remember to add a new route so our api stays clean
```
// application/config/routes.php
Route::any('api/account/updateprofile', 'account/update_profile');

```

Now we can add this method under application/modules/account/controllers/account.php
```
        public function save_profile()
        {
            $user_id = $this->current_user->id; 

            $data = $this->input->post();
            $data = $data['account_profile'];

            $outcome = $this->account_model->update_profile($data, $user_id;

            // // validations should go here
            return $outcome;
        }
```
Add the update method to the account/models/account_model.php
```
        public function update_account_profile($account_profile=NULL, $user_id)
        {
            $data = array('account_profile'=> $account_profile);
            $this->db->where('user_id', $user_id);
            $query = $this->db->update('account', $data);
        }
```
Our ngaboutme.php template needs a save button
```
<button ng-click="update_profile(account.account_profile)">
```
and our AccountProfileCtrl needs new function to handle the click event on the button
```

```


If non of this so far makes sense to you, it might be worth reviewing some of the earlier articles in this series before we move onto the main content of this lesson. Motivation is a funny thing, sometimes it's best to the novel bit first, and that will keep you interested until you solve the puzzle leaving the repitive bit to fist thing in the morning as a reward. Other times you might want to pick away at the challenging part with the tv on in the background safe in the knowledge that the boring stuff is already out the way. Everyone is different and so is everyday. What really happened is I added all the stuff above, and then decided to leave it until the morning while i make the fun bit work.

watch
          $scope.doStuff = function(input) {
            console.log(input)
            mySharedService.prepForBroadcast(input)
          }


AngularBonfire.directive('markdowndisplay', function(theService) {
    return {
        restrict: 'E',
        scope: {username: '@myAttr'},
        controller: function($scope, $attrs, $q, theService, mySharedService) {
          var username = $scope.username
          var defer = $q.defer() 
          defer.resolve(theService.getAllByUsername(username));
          defer.promise.then(function (data) {
              $scope.data = data;
              console.log($scope.data);
          $scope.list = data 
          });
          // console.log($scope.list);

            $scope.$on('handleBroadcast', function() {
              var index =  mySharedService.message;
                 $scope.display = $scope.list[index] //'Directive: ' + mySharedService.message;
            });

        },
        replace: true,
        template: '<article><h4>{{display.name}}</h4><p>{{display.description}}</p><p>{{display.rating}}</p></article>'
        // template: '<p><h2>{{display.name}}</h2><p>{{display.list}}</p></article>'
    };
});

AngularBonfire.factory('mySharedService', function($rootScope) {
    var sharedService = {};

    sharedService.message = '';

    sharedService.prepForBroadcast = function(msg) {
        this.message = msg;
        this.broadcastItem();
    };

    sharedService.broadcastItem = function() {
        $rootScope.$broadcast('handleBroadcast');
    };

    return sharedService;
});

was what i started with. First thing to do 
```
/* this bit */
AngularBonfire.directive('markdowndisplay', function(theService) {
    return {
        restrict: 'E',
        scope: {username: '@myAttr'},
        // controller: function($scope, $attrs, $q, theService, mySharedService) {
        //   var username = $scope.username
        //   var defer = $q.defer() 
        //   defer.resolve(theService.getAllByUsername(username));
        //   defer.promise.then(function (data) {
        //       $scope.data = data;
        //       console.log($scope.data);
        //   $scope.list = data 
        //   });
        //   // console.log($scope.list);

        //     $scope.$on('handleBroadcast', function() {
        //       var index =  mySharedService.message;
        //          $scope.display = $scope.list[index] //'Directive: ' + mySharedService.message;
        //     });

        // },
        replace: true,
        template: '<article><h4>jkljl</h4></article>'
        // template: '<p><h2>{{display.name}}</h2><p>{{display.list}}</p></article>'
    };
});
```

renders some stuff. Thats pretty cool. 

next i did this
```
/* this bit */
AngularBonfire.directive('markdowndisplay', function(theService) {
    return {
        restrict: 'E',
        // scope: {username: '@myAttr'},
        controller: function($scope, $attrs, $q, AccountFactory, markdownBroadcast) {
          // This is an antipattern i found useful the last time i did this
          $scope.list = 'tempdata'
          var defer = $q.defer() 
          // I think this show method on the factory is only called once
          defer.resolve(AccountFactory.show());
          // this because reasons
          defer.promise.then(function (data) {
              $scope.data = data;
          
              console.log($scope.data);
              $scope.list = data 
          });

          // console.log($scope.list);

            // $scope.$on('handleBroadcast', function() {
              // var index =  mySharedService.message;
                 // $scope.display = $scope.list[index] //'Directive: ' + mySharedService.message;
            // });

        },
        replace: true,
        template: '<article><h4>{{list}}</h4></article>'
        // template: '<p><h2>{{display.name}}</h2><p>{{display.list}}</p></article>'
    };
});
```

that word so i added
```
```

it then told me markdown was not defined so in my gulpfile.js i added. I'm enjoying working with an asset pipleine
```
var config = {
  jsGlobOrder: [ // You can add your own dependancies here as you build out your app
      path.assets  +"/angular/angular.js",
      // path.assets  +"/angular-ui/build/angular-ui.js",
      path.assets  +"/angular-ui-router.js",
      path.assets  +"/angular-animate/angular-animate.js",
      path.assets  +"/ng-file-upload/ng-file-upload-all.js",
      path.assets  +"/markdown/lib/markdown.js",
```

i had to notice that it wasnt a bower module and move into my deps folder manually. cp -r node_modules/markdown/ public/bower_components/

now it works, except it doesnt for some reason. I gave up and decided to try

```
npm install --save marked && cp -r node_modules/marked public/bower_components
```

then i tried this

```

/* this bit */
AngularBonfire.directive('markdowndisplay', function(theService) {
    return {
        restrict: 'E',
        // scope: {username: '@myAttr'},
        controller: function($scope, $attrs, $q, AccountFactory, markdownBroadcast) {
          // This is an antipattern i found useful the last time i did this
          $scope.list = 'tempdata'
          var defer = $q.defer() 
          // I think this show method on the factory is only called once
          defer.resolve(AccountFactory.show());
          // this because reasons
          defer.promise.then(function (data) {
              $scope.data = data;
          
              console.log($scope.data);
              $scope.list = marked("#thing \n ##to \n markdown");//data.account_profile);
          });

          // console.log($scope.list);

            // $scope.$on('handleBroadcast', function() {
              // var index =  mySharedService.message;
                 // $scope.display = $scope.list[index] //'Directive: ' + mySharedService.message;
            // });

        },
        replace: true,
        template: '<article ng-bind-html="list"></article>'
        // template: '<p><h2>{{display.name}}</h2><p>{{display.list}}</p></article>'
    };
});
```
it gave a weird error about unsafe data in a safe context

next i installed angular sanatize and added it to the gulp path and also into the main AngularBonfire module

```
// Declare app level module which depends on filters, and services
var AngularBonfire = angular.module('AngularBonfire', 
  [
  'ngAnimate',
  'ui.router',
  'ngFileUpload',
  'ngSanitize'
  // 'NgJoinCtrl'
  // ,'ngResource','ngAnimate','ui.bootstrap','ngRoute','firebase'
  ]);

```

And happily, things now appear to working as they should. Now all we need to do is work out how to get our newlines from the text area.

except it doesnt
```
$scope.list = marked("#thing \n ##to \n markdown");//data.account_profile);
```
works

but 
```
console.log(data.account_profile)// #thing \n ##to \n markdown
$scope.list = marked(data.account_profile);
```
doesn't which is a bit shit at half one in the morning. Was trying to avoid using another third party directive, but i guess this issue has come up before and someone has solved it already, so it's not such a bad thing

its not actually necessary, but at least it includes a markdown parser that works with angular. The target market are high retention users, ie they will have the site loaded in a tab all day, so big massive js files shouldnt be a huge problem.

eventually

```
<div class="row">
  <div class="col-6 col-desktop-6">
<textarea class="markdown-edit" ng-model="account.account_profile" value="{{account.account_profile}}" style="white-space: pre-line">
    
</textarea>
<input type="submit" href="#" class="button" value="save" ng-click="">
  </div>
  <div class="col-6 col-desktop-6 account-profile-preview">
    <!-- {{account.account_profile}} -->
    <!-- <markdowndisplay></markdowndisplay> -->
    <div marked="account.account_profile">
    </div>
  </div>
</div>


</div>
```
```
        public function update_account_profile(){
            $data = $this->input->post();
            $user_id = $this->current_user->id; ;
            $outcome = $this->account_model->update_account_profile($data['account_profile'], $user_id);
            return $outcome;
        }



```
        public function update_account_profile($account_profile=NULL, $user_id)
        {
            $data = array('account_profile'=> $account_profile);
            $this->db->where('user_id', $user_id);
            $query = $this->db->update('account', $data);
        }
```
```
var AccountProfileCtrl = AngularBonfire.controller('AccountProfileCtrl', ['$scope', '$state', '$timeout',
  'AccountFactory',
  function($scope, $state, $timeout
    , AccountFactory
    ) {

    $scope.account = {}
    $scope.saved = '';
  $scope.init = function(){
    AccountFactory.show().then(function(data) {
        console.log(data);
        $scope.account = data;
        // $scope.account.account_profile = "This is line 1\nThis is line 2"
    });
  }
  $scope.init(); 

  $scope.save = function(data) {
    console.log(data);
    var dataObject = {
      account_profile : data
    } 

    AccountFactory.updateProfile(data).then(function(data) {

      $scope.saved = 'saved'
      $timeout(function(){ $scope.saved = ''; }, 3000);
    })
  }  

}])
```
```
  factory.updateProfile = function (data) {

    var deferred = $q.defer();

    var post_data = {
      'account_profile' : data, 
      'ci_csrf_token'   : ci_csrf_token()
    }
  
    // so far we have an object we can 'POST' to our form which contains a security token
    $.post(AngularBonfireUrl+'/api/account/updateprofile', post_data).done(function(sdf){
        console.log('saved', sdf)
        deferred.resolve('done')
    })

    return deferred.promise

  }
  ```
  ```
  <div class="row">
  <div class="col-6 col-desktop-6">
<textarea class="markdown-edit" ng-model="account.account_profile" value="{{account.account_profile}}" style="white-space: pre-line">
    
</textarea>
<input type="submit" href="#" class="button" ng-click="save(account.account_profile)"><p>{{saved}}</p>
  </div>
  <div class="col-6 col-desktop-6 account-profile-preview">
    <!-- {{account.account_profile}} -->
    <!-- <markdowndisplay></markdowndisplay> -->
    <div marked="account.account_profile">
    </div>
  </div>
</div>


</div>
```



