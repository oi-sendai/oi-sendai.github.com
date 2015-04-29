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










