---
layout: post
title: "Image upload without page refresh"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Codeignitor comes with a file uploading class. In this session we will integrate this into our angular app so we can upload the image without refreshing the page. 


First of all we need a page to hold our form. account/views/image.php
```
<?php echo form_open_multipart('account/do_upload');?>
<input type="file" name="userfile" size="20" />

<br /><br />

<input type="submit" value="upload" />
</form>
```


Next lets add the upload file library and method to our account controller. We can find in public/index.php that there is a global BASEPATH available to us and we can use this to set the uploads the 

```
        function do_upload()
        {
        $config['upload_path'] = APPPATH.'../public/assets/images';
        // print_r($config['upload_path']);die;
        $config['allowed_types'] = 'gif|jpg|png';
        $config['max_size'] = '100';
        $config['max_width']  = '1024';
        $config['max_height']  = '768';

            $this->load->library('upload', $config);

            if ( ! $this->upload->do_upload())
            {
                $error = array('error' => $this->upload->display_errors());
                print_r($error);die;
                $this->load->view('upload_form', $error);
            }
            else
            {
                $data = array('upload_data' => $this->upload->data());
                print_r($data);die;

                $this->load->view('upload_success', $data);
            }
        }
```
and in the constructor
```
$this->load->helper(array('form', 'url'));
```

we might also need to grant ownership to this directory to apache. On my system this was acheived as follows

```
chgrp www-data public/assets/images
```

Note that we are simply printing the result rather than loading a success or error page. This allows us to see that our upload has been succesful and will be useful when we move the handling of the form submission over to angular and ajax. We can also see from the success data that we have the full path available to use. 

Now before we complicate ourselves any further it's probably a good time to look at that image path. It is simply storing the name of the file. We need some to intercept this. A big ol production ready site would probably hash this in a complicated way and quite likely even store the file binary in a database. For our purposes it should be sufficient to simply append the current users username onto the file alongside a epoch timestamp. This will ensure that since our usernames are unique and only image can be uploaded at a time, each image will have a unique id regardless of whether they upload a second image with the same name. 

Looking at the library we have loaded from bonfire/codigntior/libaries/upload we can that the $config object we pass with our request accepts a $filename parameter. All need do now is find a way to intercept the name of the incoming image. This will also be useful if we wish to add an alt attribute to the image a later date.

A few moments thought might tell up this since this is essentially (well actually) an http post we have access to the information this way. This will become very useful shortly because we will also have to pass a ctfr token alongside our ajax request. A little investigation of the Upload class reminds us that we need to access the $_Field property of the post which the api specifies must be set as 'username'

```
function do_upload($field = 'userfile')
{

    $data = $_FILES[$field];
	print_r($data);die;
```
Which let's us see the following information 
```
Array ( [name] => tweet.png [type] => image/png [tmp_name] => /tmp/phpdd5ysZ [error] => 0 [size] => 7538 ) 
```

We now want a function that will take this data and process it into a unique string. Since we might need to use this function elsewhere and to keep QA happy it makes sense to have this as a seperate function. This follows the principles of DRY (Do not Repeat Yourself) software development. Since this function is only relevant for file uploads it makes to extend the existing Upload library. In Codeignitor this is done by create a new file under application/libraries/MY_Upload.php . The MY_ prefix tells ci to look at this file first and to use it in place of the existing. This way you could completely override the existing function, or in our case simple extend it as follows to get some feedback that it works


```
    public function dothing()
    {
		echo 'works';die;
    }
```
The original library has the following method
```
	/**
	 * Set the file name
	 *
	 * This function takes a filename/path as input and looks for the
	 * existence of a file with the same name. If found, it will append a
	 * number to the end of the filename to avoid overwriting a pre-existing file.
	 *
	 * @param	string
	 * @param	string
	 * @return	string
	 */
	public function set_filename($path, $filename)
	{
		if ($this->encrypt_name == TRUE)
		{
			mt_srand();
			$filename = md5(uniqid(mt_rand())).$this->file_ext;
		}

		if ( ! file_exists($path.$filename))
		{
			return $filename;
		}

		$filename = str_replace($this->file_ext, '', $filename);

		$new_filename = '';
		for ($i = 1; $i < 100; $i++)
		{
			if ( ! file_exists($path.$filename.$i.$this->file_ext))
			{
				$new_filename = $filename.$i.$this->file_ext;
				break;
			}
		}

		if ($new_filename == '')
		{
			$this->set_error('upload_bad_filename');
			return FALSE;
		}
		else
		{
			return $new_filename;
		}
	}
```

Let's try rewriting it to create the fileformat we want

First we need access to the current user

```
	 public function set_filename($path, $filename)
	 {
	  
	  $CI =& get_instance();

      $blah = $CI->load->library('users/auth');
      print_r($blah);die;
	}
```

gives us access to the user auth library. I was a little confused as to the proper way to include this within a libary rather than a controller so the above lets us see the whole library on our screen. By searching for the username we are currently logged in by we can find the field we need access to but at the moment it's impossible to read. Let's neaten it up a little

```
echo'<pre><code>';print_r($blah);echo '</code></pre>';die;
```

It looks a little like below but far too much to show it all so we will just look at the place where our username appears

```
            [session] => CI_Session Object
                (
                    [sess_encrypt_cookie] => 
                    [sess_use_database] => 
                    [sess_table_name] => sessions
                    [sess_expiration] => 7200
                    [sess_expire_on_close] => 
                    [sess_match_ip] => 
                    [sess_match_useragent] => 1
                    [sess_cookie_name] => bf_session
                    [cookie_prefix] => 
                    [cookie_path] => /
                    [cookie_domain] => 
                    [cookie_secure] => 
                    [sess_time_to_update] => 300
                    [encryption_key] => 58f62e7a5c072527eb00fad7ccb6f547
                    [flashdata_key] => flash
                    [time_reference] => utc
                    [gc_probability] => 5
                    [userdata] => Array
                        (
                            [session_id] => 05f02d3d228ea418a8a3723c8451fbab
                            [ip_address] => 127.0.0.1
                            [user_agent] => Mozilla/5.0 (X11; Ubuntu; Linux i686; rv:36.0) Gecko/20100101 Firefox/36.0
                            [last_activity] => 1430308960
                            [user_data] => 
                            [requested_page] => http://gardenspa.ce/index.php/account/do_upload
                            [user_id] => 5
                            [auth_custom] => yoshi
                            [user_token] => 475ccaf35c075196d9f912d5a6ec8c1a09d40aed
                            [identity] => yoshi@gmail.com
                            [role_id] => 4
                            [logged_in] => 1
                            [language] => english
                            [previous_page] => http://gardenspa.ce/index.php/account/ngimage
                        )
```

After a little (lots) of trial and error and some recursively more accurate google searching (it's an important skill), our final method looks like

```
	public function set_filename($path, $filename)
	{
		// assign by reference - not a copy
		$CI =& get_instance();

      	// Access username name from session
      	$username = $CI->session->userdata('auth_custom');

		// The epochtime 
		$time = time();

		if ($this->encrypt_name == TRUE)
		{
			mt_srand();
			$filename = md5(uniqid(mt_rand())).$this->file_ext;
			return 
		}

		// Build our new filename
		$new_filename = $username.$time.$filename; // not this already has the file extension
		return $new_filename;

		}
	}


Note this is a little shorter than the original method. Since we are creating the filename with a timestamp the filename should always be unique. Happy days.

Okay we are ready to integrate with angular. Since this has been solved comprehensively by someone else, it seems sensible to use the pre-existing solution. 

First we need to create a .bowerrc file in the route of our project (if we don't already have one)

```
{
  "directory" : "public/bower_components"
}
```

And then we can install the plugin

```
bower install -save ng-file-upload
```

We can now see that the file has been installed. Next we need to manually add this dependancy into our assets pipeline/

```
// gulpfile.js
var config = {
	jsGlobOrder: [ // You can add your own dependancies here as you build out your app
	    path.assets  +"/angular/angular.js",
	    path.assets  +"/angular-ui-router.js",
	    path.assets  +"/ng-file-upload/ng-file-upload-all.js",
	    ...
```

And finally inject it into our application public/themes/soil/ng/angular-bonfire

```
var AngularBonfire = angular.module('AngularBonfire', 
	[
	'ngAnimate',
	'ui.router',
	'ngFileUpload'
	]);
```

Run gulp and refresh our app to make sure we have not made any errors including the files.

Next we need to make a few modification from the upload controller from the plugin documentation https://github.com/danialfarid/ng-file-upload

```
var AccountImageCtrl = AngularBonfire.controller('AccountImageCtrl', ['$scope', 'Upload', function ($scope, Upload) {
    $scope.$watch('files', function () {
        $scope.upload($scope.files);
    });


    $scope.upload = function (files) {
        if (files && files.length) {
            for (var i = 0; i < files.length; i++) {
                var file = files[i];
                Upload.upload({
                    url: AngularBonfireUrl+'/account/do_upload',
                    headers: {'Content-Type': file.type},
                    method: 'POST',
                    fileFormDataName: 'userfile', 
                    fields: {ci_csrf_token: ci_csrf_token(), userfile: file},
                    file: file
                }).progress(function (evt) {
                    var progressPercentage = parseInt(100.0 * evt.loaded / evt.total);
                    console.log('progress: ' + progressPercentage + '% ' + evt.config.file.name);
                }).success(function (data, status, headers, config) {
                    console.log('file ' + config.file.name + 'uploaded. Response: ' + data);
                });
            }
        }
    };
}]);
```
Notice first that we now able to pass Upload dependany into our controller. The main modifications we have to make however are in the object we pass to the newly available Upload service. 
```
{
                    url: AngularBonfireUrl+'/account/do_upload',
                    fileFormDataName: 'userfile', 
                    fields: {ci_csrf_token: ci_csrf_token()},
```
We need to set the url (using a global we have defined for our app). We also need to pass the csrf token to let ci know that the ajax request is coming from the same session as the user is logged into. Finally we need to set the fileFormDataName to one that codeignitor asks for.

Next we can simply copy in the html from the plugin docs remembering to change it to our controllername. It's kinda not very organised or neat at the moment, but this current sprint is simply concerned with getting the basic functionality in place.

```
<div ng-app="fileUpload" ng-controller="AccountImageCtrl">
    watching model:
    <div class="button" ngf-select ng-model="files">Upload using model $watch</div>
    <div class="button" ngf-select ngf-change="upload($files)">Upload on file change</div>
    Image thumbnail: <img ngf-thumbnail="files[0]" class="thumb">
    Drop File:
    <div ngf-drop ng-model="files" class="drop-box" 
        ngf-drag-over-class="dragover" ngf-multiple="true" ngf-allow-dir="true"
        ngf-accept="'.jpg,.png,.pdf'">Drop Images or PDFs files here</div>
    <div ngf-no-file-drop>File Drag/Drop is not supported for this browser</div>
</div>
```

It would be nice to do some image resizing/cropping to make everything a bit standardized however I'm quite enjoying the anarchic bits welded onto the side style this app is developing, it think it would make for a great mechanical ftoy, so for now lets just finish off our bonfire controller method to save the image to the database. 

Inside application/modules/account/models/account_model.php create a new method to handle our database update
```
        public function update_image($image=NULL, $user_id)
        {
            $data = array('image_path'=> $image);
            $this->db->where('user_id', $user_id);
            $query = $this->db->update('account', $data);
        }
```
We can now call this from within our success conditional inside our do_upload function

```
            if ( ! $this->upload->do_upload())
            {
                $error = array('error' => $this->upload->display_errors());
                print_r($error);die;
            }
            else
            {
                $data = $this->upload->data();
                $user_id = $this->current_user->id;


                $this->account_model->update_image($data['file_name'], $user_id);
                print_r($data);die;


            }
 ```

You will notice again there is very little to no error checking going on here. This is basically because we don't have a test suite set up and writing complix conditional logic is as you will discover totally stupid. Writing things this way as beginner is generally better because it's really hard to understand a test set up when you don't understand the platforms your testing. It's even more difficult to build one. So to keep ourselves interested we will keep going along like this, and then once we have something that feels solid, and like we would be upset if it broke, then we will have the motivation to set up a test suite, and go through each feature as if from new and write comprehensive tests which will be much easier to mock now that we understand how everything is fittin together.

Okay, so last thing to do is adjust our view so it loads the existing picture first, and then updates this with the new one at the appropriate time.

```
<!-- Inside AccountImageCtrl -->
    $scope.image = {}
    
    $scope.init = function(){
      AccountFactory.show().then(function(data) {
        console.log(data);
        $scope.image = data.image_path
      });
    }
    $scope.init();
```
and inside our view filev modules/account/views/image.php
```
I<img ngf-thumbnail="files[0]" class="thumb" ng-src="<?php echo site_url();?>/images/{{image}}">
```
