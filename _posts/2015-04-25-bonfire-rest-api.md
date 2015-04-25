---
layout: post
title: "Splicing an Angular directive to a php template engine"
description: ""
category: 
tags: []
---
{% include JB/setup %}

One of the neat things about Angular is that is that it works with rendered html. This is useful if you want to spice up your apps a little using some of angulars dom manipulation and animation facilities, but to use these along side traditional routing in your php application.

The user authenticion and routing I wanted to keep as a standard bonfire/codeignitor facility. This allows us to to have a standard site.com/profile/username route set up to be publically available. The view will display exactly the same for both registered users and vistors, the only difference being that some of the functionality such as the contact this user feature will only display for logged in users.

Inside our applicaiton/modules/profile/controllers/profile.php we can now define the default site.com/profile route to return an error message

```
    class Profile extends Front_Controller
    {

        public function __construct()
        {
            parent::__construct();

            $this->load->model('users/user_model');
        }

        public function index()
        {
            //throw error code
        }
```

Because there is no parameter after profile/ our route setting will not catch the request and use the default behaviour. Conversly when profile/(:username) is called this will be routed to the show method of our controller. Usually this would be called at profile/show/username but this seemed needlessly complicated for the end user hence we can override this behaviour

To achieve this we first need to set a route in application/config/routes.php

```
Route::any('profile/(:any)', 'profile/show/$1');
```

and now when we create our show method inside the controller it will have access to the second parameter as a variable. 

```
        // si.te/profile/$username
        public function show($username=''){ 

            Template::set('username', $username);
            Template::set_view('profile');
            Template::render();
        }
```

We now have access to the username variable inside application/modules/profile/views/profile.php

```
<aside>
</aside>
<main>
	<h2><?php echo $username;?></h2>
</main>
```

Equally we could also add other non dynamic content into the view using Template::set in the same way. 

I made the design decision early on that in 2015 spending time to make web pages work for people without javascript is about as sensible as providing a printed copy for people without web browsers. The means we can do the next bit using javascript based http calls to an api that lives alongside our applications routing.

The next stage then is to build out our api. I'm currently a bit behind on current trends and generally work to a DTT (design then test) workflow. This is mostly because I'm still learning, and I find writing tests for technology I don't yet understand a little cumbersome.

Codeignitor does it's database communication using active record. This is usually okay to work with but for more complex queries i tend actually write them out as sql (If your windows or linux(wine) user HeidiSQL is a really well laid out application), and once I have found a query that returns the information I need Active Record make a available a nice wraper which enables us to use the query directly inside our codebase without losing any of the security benifits of active record.

```
    class Profile_model extends MY_Model
    {
        private function getAbilities($username){

            // get user abilties
            $sql = "SELECT name, description, rating, e.active 
                FROM bf_users e 
                LEFT JOIN bf_abilities t
                ON e.id=t.user_id
                where e.username = ?;";

            $query = $this->db->query($sql, array($username))->result(); 
            
            return $query;
        }
    }
```

This is query is a little more complicated than we possibly need since it joins two tables, taking out the confidential information from the user table and combining it with a list of abilities the user has. The extra complication should be worth it later however as it both gives us access to the username of the page within our javascript data. I hope

We now have everything we need to set up a restful route from modules/profiles/controllers/profile.php 

```
        public function __construct()
        {
            ...
            $this->load->model('profile/profile_model');
        }

        si.te/api/profile/getabilities/$username
        public function getAbilitiesJson($username)
        {
            $abilties = $this->profile_model->getAbilities($username);
            $data = json_encode($abilities);
            echo $data;
        }
```

It might help keep our javascript neater if we also set up a new route under application/config/routes.php and also has the advantage of not forcing us to set up new user validation rules or set a special order for our routes. Now we know all requests to profile/ are concerned with returning the profile template to the end user and all requests to api/profile are concerned with returning data.

```
Route::any('api/profile/getabilites/(:any)', 'profile/getAbilitiesJson/$1');
```

Now visiting site.com/api/profile/getabilites/test (assuming you have added the data to the database, we covered this in an earlier article that hasn't been written yet, this article is actually about integrating Angular js) should show you a page showing a json object similar to the one below

```
[{"name":"Chess","description":"Enjoy, dont stratagise","rating":"3","active":"1"},{"name":"Go","description":"Well, not really","rating":"1","active":"1"},{"name":"MongoDB","description":"Used with express","rating":"3","active":"1"}]

Groovy. That is out page set up. AngularBonfire has an asset pipeline set up using gulp that will include any javascript files under application/modules/modulename/ng/**.js and conbine them with the angular dependancies. If you don't care about CodeIgnitor or bonfire and just want to know about the angular stuff start here with your normal set up and api returning a similar dataset.








