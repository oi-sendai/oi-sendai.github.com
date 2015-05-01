---
layout: post
title: "Simple, two way communication. Part One"
description: ""
category: 
tags: []
---
{% include JB/setup %}

#The plan

So this session we will try and build a simple chat funcitonality. Essentially it's going to be based around a single database table which we will have to create. For this to work we will need the following structure

message_id, sender_id, recipient_id, message, checked, time

This will be quite flexible, as it will allow us to query the database in the following way

for user_id get all messages where checked is not true

To display in our notifications panel - the user can see how many new messages they have

clicking a message will result in the following query

for user_id = recipient_id get all message where sender_id order by most recent

this is not proper code of course, but with the help of a bit of paper and a few psuedo queries, what seemed like an overwhelming task is actually reasonably approachable. 

So we will also need a form. This is will be quite trivial. It will live in a widget on the profile page of the person we are sending a message to. There will also be one on our own page that we can use to reply to messages we have recieved. 

The form will hit a controller, the controller will save the form to the database via a model. We can add an option on the profile page at a later stage asking the user whether they wish to recieve email notifcations of messages, but this complicates things needlessly for this sprint.  

Now we have a way to add messages to the system, we can make it more user friendly by building this functionality out with angular using services and displaying messages using ng-repeat. This also has the benfit that we can use angulars $timeout on the client side to poll the service for new messages and update the viewmodel accordingly.


#What happened 

#### All code for this tutorial can be found at https://github.com/oi-sendai/AngularBonfire/commit/a3bb58b0c5aabef1ea635e4cf2d44f51c07124f8

First off we created our table.

#####application/modules/chat/migrations/chat.sql

```

CREATE TABLE `bf_chat` (
	`message_id` INT(11) NOT NULL AUTO_INCREMENT,
	`sender_id` INT(11) NULL DEFAULT NULL,
	`recipient_id` INT(11) NULL DEFAULT NULL,
	`message` VARCHAR(250) NULL DEFAULT NULL COMMENT 'Markdown',
	`checked` VARCHAR(50) NULL DEFAULT NULL COMMENT 'bool 1 or 0',
	`timestamp` VARCHAR(50) NULL DEFAULT '3' COMMENT 'now()',
	PRIMARY KEY (`message_id`),
	INDEX `Index 2` (`sender_id`),
	INDEX `Index 3` (`recipient_id`)
)
COLLATE='latin1_swedish_ci'
ENGINE=InnoDB
AUTO_INCREMENT=4
;

```

We realised that this is going to scale horrendously, but it will do just fine to prototype out our ui. Next step is the form, then we have a beginning and an end, and can approach the middle quite simply.


Lets create a little widget we can include on the page. We already have a set up a profile page, and the minimum functionality we need for the site is for people to be able to contact people via there profile. Lets create the widget inside the profile controller 

#####application/modules/chat/controllers/chat.php

```

    class Chat extends Front_Controller
    {

        public function __construct()
        {
            parent::__construct();

            $this->load->library('users/auth');
            $this->set_current_user();

            Assets::add_js('codeigniter-csrf.js');
        }
        public function widget(){
            $viewdata = array('data' => 'hello widget');
            $this->load->view('chat/widget', $viewdata);
        }
    }
```

and a container for our view

#####application/modules/chat/views/widget.php

```
<hr/>
<h4>Contact This User</h4>
<?php if (empty($current_user)) : ?>
	You must be <a href="<?php echo site_url(); ?>">registered</a> to reach out to people. 
<?php else : ?>
	<input type="hidden" value="<?php echo $username; ?>">
    <textarea value="" placeholder="message goes here"></textarea>
    <button type="submit" class="button">
<?php endif; ?>
<hr/>
```

We have already hit our first bottleneck. How to we obtain the username from the current profile page. We are going for some spaggettis now. Or we are avoiding the spaggettis succesfully. Thankfully we have access to $username as a variable in the profile template so we can pass this as an argument when include load our partial

#####somewhere inside profile template

```
<?php echo Modules::run('chat/widget', $username); ?>
```

for our conditional logic to work inside the partial we also need to pass a $currentuser argument to our view. The completed controller code 

#####application/modules/chat/controllers/chat.php

```
public function widget($username=NULL){
	$current_user = $this->current_user->id;
    $viewdata = array('data' => 'hello widget');
    $viewdata = array('username' => $username );
    $viewdata = array('current_user' => $current_user );
    $this->load->view('widget', $viewdata);
}
```

Great, so now we have a little widget on a users profile page which displays a reminder for the non registered/logged in users and a form for those that are already registered.

Let's add some angular js to the widget which we will use to handle the form submission.  First we need to create a factory that we can inject into our controller. this simply exposes a sendMessage function which hits a restful endpoint using an http post to deliver our form data.

#####application/modules/chat/ng/ng-chat.js

```
AngularBonfire.factory("ChatWidgetFactory", function($http, $q) {

  var factory = {}

  factory.sendMessage = function (data) {

    var deferred = $q.defer()
    
    $http.post(AngularBonfireUrl+'/api/chat/sendmessage', data).then(function(resp) {
      
      deferred.resolve(resp.data)
    })

    return deferred.promise
  }

  return factory
})
```

Here we set up the basic Chat Widget Controller. Note that we pass in our factory service which we will use when saving the message. For now we have kept things very light. We set an empty success message on the scope. We also have created a function we can call with ng-click inside our view. The next problem we have however is that angular does not play very nicely with input type="hidden", in fact it ignores it completely when using ng-submit to serialize a form. To get around this, lets create another scope variable $scope.formData

#####application/modules/chat/ng/ng-chat.js

```
var ChatWidgetCtrl = AngularBonfire.controller('ChatWidgetCtrl', 
  ['$scope', '$state', '$timeout','ChatWidgetFactory',
  function($scope, $state, $timeout, ChatWidgetFactory) {

  $scope.success = ''
  $scope.formData = {}

  $scope.send = function() {
    console.log($scope.formData);
  } 
}])

```

Moving back to our template we need to set up a couple of extra parameters on the elements to allow them to work with the controller we have just set up. The ng-controller element is set to ChatWidgetCtrl and the ng-submit="send()" tells the form to ignore it's default behaviour and instend call the send() function in our controller. Each of our form elemements has it's ng-model value bound to a property our $scope.formData object (So when we submit the form we can see in our console: Object { recipient_id="user1", message="Hello user1"} ). Finally because angular doesn't naturally recognise input type="hidden" we need to use the ng-init directive to force it to recognise a starting value for ng-model="formData.recipient_id".

#####application/modules/chat/views/widget.php

```
<?php else : ?>
	<form ng-controller="ChatWidgetCtrl" ng-submit="send()">
    	<input type="hidden" ng-init="formData.recipient_id='<?php echo $username; ?>'"  ng-model="formData.recipient_id" >
    	<textarea placeholder="message goes here" ng-model="formData.message"></textarea>
    	<input type="submit" class="button" value="Send">
    	<span class="form-sent">{{succes}}</span>
	</form>
<?php endif; ?>
```

So far so good, our form handling function now has access to the data we need and we can now add to our send() function accordingly.
A number of important things are happening here. First we call the send method of our factory. This uses angulars $q service which you should probably read up, but basically it is used to asynconious requests, (ie javascript rather running in a straight order as normal, it does several things at once, so $q.defer() tells it to wait until the previous operation is complete $q.resolve is used to record the return value of the http request (see above for full code), and the return deferred.promise stops the return from activating until a value has passed to the resolve function), it's reasonably confusing, so for now the take home from this is that the 'then' portion of your send function will not run until it receives the return value back from the promise. This can be used to process error messages but for now we are simply assuming everything works on the server side and using the then(function{}) to update the scope with a sucess message and then to hide this message after a short interval using angulars $timeout function (basically the same as set.timeOut but with some angular stuff to make sure it resolves the scope after it runs).

#####application/modules/chat/ng/ng-chat.js

```
  $scope.send = function() {
    console.log($scope.formData)
    ChatWidgetFactory.sendMessage($scope.formData).then(function(data) {
       $scope.success = 'Message Delivered'
       $timeout(function(){ $scope.success = ''; }, 3000)
    })
  } 
```
Deep breath here! There is a lot going on, but that is about as complicated as things should get. All things being well, your console should inform you that the restful enpoint has not been found POST http://gardenspa.ce/index.php/api/chat/createmessage 404 Not Found

I like to explicitely set routes within codeIgnitors routing system when create a restful endpoint as it makes it easier to manage your code. For instance should you wish to change the php controller which handles the request you can do so in the router without having to worry about changing all the places it is called from your javascript. 

#####application/config/routes.php

```
// Chat
Route::any('api/chat/sendmessage', 'chat/send_message');
```

Now we know where the code is going, lets add the required function into the controller on our server. We need to add die; at the end to stop the built in bebug profiller from returning it's html alongside our echo, and for the time being let's just print out the submitted post data so we can see it in our console.

#####application/modules/chat/controllers/chat.php

```
	public function send_message(){
	    $data = $this->input->post();
	    print_r($data);die;
	}
```

Unfortunately you will instead see a 500 Not Allowed error message  'The action you have requested is not allowed. You either do not have access, or your login session has expired and you need to sign in again.'. This is a built security measure from the server to stop cross domain attacks and also since anything in the browser could be modified by the user, a 'token' must be sent alongside and PUT or POST requsts that come from javascript.
Let's add this to our factory and modify our controller accordingly


#####application/modules/ng/ng-chat.js

```
  factory.sendMessage = function (data) {

    var deferred = $q.defer()

    var post_data = {
        'form_data'     : data, 
        'ci_csrf_token' : ci_csrf_token() // a function defined by CI_Bonfire we included in the constructor that loads the widget
    }
	
  	// o $http.post(AngularBonfireUrl+'/api/chat/sendmessage', data) // what we had before
  	$.post(AngularBonfireUrl+'/api/chat/sendmessage', post_data).then(function(resp) { -->
```

#####application/modules/chat/controllers/chat.php

```
	constuct
	public function send_message(){
	    $data = $this->input->post();
	    print_r($data['form_data']);die;
	}
```

It's also worth noting (if you don't want an error), that I change $http.post (angulars method), to use the include $.post (jquery). I can't remember why this works but it does, and we need to skip lightly past as one of hundreds of thousands of things we don't fully understand when writing code for the modern web. It's very much like being an electrian that doesn't have to understand quantum physics in order to wire up a house.

All being well, our brower console should now log a 200 Success message with an array

```
	Array
	(
	    [recipient_id] => yoshi
	    [message] => dsfsdfsd
	)
```

If you remember our table we created at the start you should be able to figure out that we need a few more bits of information before we can send our data for saving. Let's add those now

#####application/modules/chat/controllers/chat.php

```
        // api/chat/sendmessage
        public function send_message(){
            // get the data from the http POST
            $data = $this->input->post();
            // and seperate out the form data 
            $form_data = $data['form_data'];
            
            // using the inbuilt auth library loaded in the constructer
            $sender_id = $this->current_user->id; 

            // uses a native php function to record the data/time
            $timestamp = time();

            $mock_message = array(
                'sender_id'    =>  $sender_id,   
                'recipient_id' => $form_data['recipient_id'],
                'message'      => $form_data['message'],
                'timestamp'     => $timestamp,
                'checked'      => 0
            );

            $outcome = $this->chat_model->create_message($mock_message);

            echo $outcome;die;  // we could use this for error messages on the front end
        }
```

Notice we are now referencing $this->chat_model->create_message() we need to add this to our controllers constructor before building the model layer.

#####application/modules/chat/controllers/chat.php

```

        public function __construct()
        {
            parent::__construct();

            $this->load->model('chat/chat_model');
```

#####application/modules/chat/chat_model.php

```
	class Chat_model extends MY_Model
    {

        protected $table_name   = 'chat';
        protected $key          = 'chat_id';
	
	
        public function create_message($data=NULL)
        {
        	// do substitution username for userid in case user changes name
        	$abilities = $this->db->
                    where('username', $data['recipient_id'])->
                    get('users')->result();

            $foo = $abilities[0];

            $data['recipient_id'] = $foo->id;


            $this->db->insert('chat', $data); 
            
            return true;

        }
}
```

I'm not sure if this the most awesome way to do it , but it works and it's vaguelly testable so lets try it. You will notice we do a bit of a funky substitution within the model swapping out the username for the user id. This avoids rewriting everything whilst still preserving our data. I guess I will leave it in as a reader exercise to work out exactly what's it doing since we now have data in the database I want to crack on with part two, adding the reply facility to the account section.



      

	
 









