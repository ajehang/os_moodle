A personal introduction for Moodle on Red Had Openshift Express

###Quick install
In order to get started you will need a system that has git and Ruby and a gem called `rhc`:

    gem install rhc

Register with Openshift over at [https://openshift.redhat.com/](https://openshift.redhat.com/app/) and create a new express php application (follow their guides).

Once you have created an application you can clone this repository:

    git clone git@github.com:burningTyger/os_moodle.git

Enter its directory
    
    cd os_moodle

And set up some important stuff:

    rhc-ctl-app -a moodle -e add-mysql-5.1 -l yourlogin
    git remote add openshift your_openshift_git_url

`yourlogin` is obviously the login email address that you used to sign up for Openshift and `your_openshift_git_url` is the url that Openshift gave your new app. You can find the url in your express dashboard. In this example I called the app `moodle`. If you want to use a different name make sure that you range the name of the `-a` option as well.

Since everything is ready to go you can now push Moodle to Openshift with a simple:

    git push -f openshift master

Now you can visit Moodle at your app url and finish up the remote install process.

Because the Moodle team usually updates Moodle weekly be sure to update your app on a regular basis. To do so you can pull from this repository:

    git pull origin master

and update your Openshift app:

    git push openshift master

*Have fun*

###Roll your own
To get started we clone Moodle into a directory and set up a remote tracking branch for stable releases (which is at 2.1 right now).

I also use a working directory called `~/openshift` where I store my Openshift apps for easy maintenance:
	
	cd ~/openshift
	mkdir moodlestage
	cd moodlestage
    git clone git://git.moodle.org/moodle.git
	cd moodle
	git checkout --track origin/MOODLE_21_STABLE

Back in `~/openshift` I create a new app called moodle with my rhc client:

	rhc-create-app -a moodle -t php-5.3 -l yourlogin

The reason we stored Moodle in `moodlestage` was to let the rhc client create a `moodle` directory for our app.

If everything was set up properly rhc will tell you the new address of your app and its remote git repository.

Now we need to set up MySQL so we can use it with Moodle:

	rhc-ctl-app -a moodle -e add-mysql-5.1 -l yourlogin

If successful it will return some information that you should save somewhere: username, password, db name and connection URL.

Now it's time to put Moodle into its proper directory which we need to clean up first:

	rm –rf  ~/openshift/moodle/php/*
	
then we copy:
	
	cp –av ~/openshift/moodlestage/moodle/* ~/openshift/moodle/php

The next step will set up Moodle's config file so it can connect to the databse and do other great things. The following lines are somewhat compressed as for brevity I left out all the comments. Normally you would edit the `config-dist.php` file and then rename it to `config.php`. If you are lazy create that file and copy these lines into it (but make sure you use your domain name for wwwroot):

	<?PHP
	unset($CFG);
	global $CFG;
	$CFG = new stdClass();
	
	$CFG->dbtype    = 'mysqli';
	$CFG->dblibrary = 'native';
	$CFG->dbhost    = $_ENV['OPENSHIFT_DB_HOST'];
	$CFG->dbname    = $_ENV['OPENSHIFT_APP_NAME'];
	$CFG->dbuser    = $_ENV['OPENSHIFT_DB_USERNAME'];
	$CFG->dbpass    = $_ENV['OPENSHIFT_DB_PASSWORD'];
	$CFG->prefix    = 'mdl_';
	$CFG->dboptions = array(
	    'dbpersist' => false,
	    'dbsocket'  => false,
	    'dbport'    => $_ENV['OPENSHIFT_DB_PORT'],
	);
    $CFG->wwwroot   = "https://$_ENV[OPENSHIFT_APP_DNS]";
    $CFG->sslproxy  =true;	
    $CFG->dataroot  = $_ENV['OPENSHIFT_DATA_DIR'];
	$CFG->directorypermissions = 02777;
	$CFG->admin     = 'admin';
	require_once(dirname(__FILE__) . '/lib/setup.php');

Note that we used an *https* URL for secure connections. We therefore had to add `$CFG->sslproxy = true;` to the config file.

However, this is under the assumption that you want to use SSL connections to your Moodle instance and that you want to use the default URL.

Since you can also have a custom domain alias this might not be the best option. If you want to use your own domain and SSL make sure you have a valid certificate otherwise most browsers will complain about invalid certificates and not let the user see your site. If you don't have a valid certificate but still want to use a custom domain and you don't want to bother users with browser messages change the config to `$CFG->sslproxy = false`.

Once this is finished we need to add all the new files and commit it to the new repository:

	cd ~/openshift/moodle
	git add .
	git commit –am "Initial commit for Moodle"

Now we can push to Openshift and be ready for installing Moodle:

	git push

You should now be able to visit your new app and start the install script.

If you want to keep your site up to date you will have to update the `/moodle/php` dir frequently. There are weekly Moodle updates which I recommend you pull in:

    cd ~/openshift/moodle/php
    git pull
    
This will update Moodle but you still need to update your app as well:

    cd ..
    git add .
    git commit -m 'weekly update'

And then push everything to Openshift:

    git push

*Have fun.*