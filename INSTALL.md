# NB Installation Procedure

## 0 - Clone Repository
Clone the project from git: https://github.com/nbproject/nbproject. Make sure that the project is in a directory that the user running the server (e.g www-data, /var/www/html/) can access. 

## 1 - Dependencies
On a typical Debian-like (Ubuntu etc...) distribution, NB requires the following base packages:

**NOTE:**
  * packages in square brackets are optional
  * core ubuntu dependencies can be installed by running 'sudo make prereqs'
  
### CORE DEPENDENCIES
These can be installed as Ubuntu packages

    python (>= 2.6)
    postgresql (>= 8.4)
    imagemagick
    mupdf-tools (for pdfdraw)
    context (for rich, i.e. annotated pdf generation)
    apache2
    libapache2-mod-wsgi
    python-setuptools (used for 'easy_install pytz' in make prereqs as well)
    g++ (used to compile node)
    python-pip (used to install google python library and other dependencies).
    nodejs
    npm
    nodejs-legacy (in order to use executable name node instead of nodejs) 
   
### Pip installations
    cd nbproject # (or whatever name you may have choosen for the root NB code directory).
    sudo pip install -r requirements.txt # To install additional required dependencies such as django, pyPdf, etc
      
### grunt
    # As of March 2016 we have been using grunt 0.4.X. 
    # For questions on getting NB to work wit grunt 0.3.X, plrease refer to 
    # http://gruntjs.com/upgrading-from-0.3-to-0.4, and follow the procedure in the opposite way. 
      npm install grunt 
      sudo npm install -g grunt-cli
      sudo npm install -g grunt-init
      npm install grunt-contrib-jshint  --save-dev
      npm install grunt-contrib-concat  --save-dev
      npm install grunt-contrib-cssmin  --save-dev
        
### [optional] To annotate YouTube Videos: The google python library
    sudo pip install --upgrade google-api-python-client


## 2 - Installation commands
   Once you've satisfied the dependencies listed above, you need to run the following installation commands (from nb root folder):
```
    npm install               # in order to install specific grunt modules, such as grunt-css, execSync
    make django               # create configuration files. You can safely ignore the "Error 127" message
    sudo make create_dirs     # create root folder and some folders below that for nb data
    make django               # one more time...
    grunt                     # Compiles JS and CSS
    
```
* Copy ```nbproject/apps/nbsite/settings_credentials.py.skel``` to ```nbproject/apps/nbsite/settings_credentials.py```. Edit the values in ```apps/nbsite/settings_credentials.py```, following the directions there.

* If you are deploying onto a server for a production environment, follow the commands printed out by the ```make django``` in order to properly configure your Apache files and cron jobs. Remember to remove the Apache default conf file in ```/etc/apache2/sites-available/```.
 
* Optional: If you want to use different slave servers for different parts of the app (i.e. one for serving images, one for handling the rpc calls, and one for handling file uploads for instance), edit params in ```content/ui/admin/conf.js```: Tell which server(s) the client should use to fetch various pieces data. If you're going to use just one server for the whole app, you can safely ignore this. Note that this is unrelated to whether or not you're using localhost as your  databse server, but if you do use several servers, make sure they all use the same database, for consistency.  Don't forget to re-run ```grunt``` if you change ```conf.js```.

## 3 - Database Initialization

* Log in as a user who has psql superuser privileges, such as postgres. Create a superuser account and a database. For example, one way to do this is: 
```
    sudo -u postgres -i
    createuser <YOUR_POSTGRES_USER> -P -s 
               # important to setup as superuser since only superusers can create a language (used for plpythonu)
    createdb -U <YOUR_POSTGRES_USER> -h localhost <YOUR_NB_DATABASE_NAME>
    
``` 
* Exit from the database and from the postgres user. Then run:

```
    cd apps
    ./manage.py makemigrations # to create the database migrations files
    ./manage.py migrate # To create the database tables from the migrations files
    ./manage.py sqlcustom base | ./manage.py dbshell # to create custom views. 
```

If the last line returns a ```"Only the sqlmigrate and sqlflush commands can be used when an app has migrations."``` error, you can log in to your postgres database and run the queries contained in https://github.com/nbproject/nbproject/blob/dev/apps/base/sql/ensemble.sql 


* You may also have to allow remote connections by adding the following line to the end of the ```/etc/postgresql/[YOUR_VERSION]/main/pg_hba.conf ``` file:
```
host    your_db_name       your_db_user       127.0.0.1/0     password
```
* And in the "Connections and Authentication" section of ```/etc/postgresql/[YOUR_VERSION]/main/postgresql.conf```, change:
```
   listen_addresses = '*' 
```
* If you make a mistake:
 
 ```
    dropdb  -U <YOUR_POSTGRES_USER> -h localhost <YOUR_NB_DATABASE_NAME>
    createdb -U <YOUR_POSTGRES_USER> -h localhost <YOUR_NB_DATABASE_NAME>
```
* If you are running NB locally, open ```nbproject/apps/nbsite/settings_credentials.py``` and uncomment the lines that set the values of ```EMAIL_BACKEND``` and ```EMAIL_FILE_PATH```.  Then set:
```
EMAIL_BACKEND = 'django.core.mail.backends.filebased.EmailBackend'
EMAIL_FILE_PATH = '<EMAIL_FOLDER>'
```
Make sure that your user has permissions to write to `<EMAIL_FOLDER>`.

* At this point you can try your installation using the Django debug server (but never use this in production...): 
* From the apps directory: ```./manage.py runserver```
* In your browser visit ```http://localhost:8000```

## 4 - Extra stuff
  To be able to genereate annotated pdfs: Configure tex so that it allows mpost commands: make sure that 'mpost' is in shell_escape_commands (cf /tex/texmf/texmg.cnf) 

## 5 - Crontab setup
  A sample crontab generated as part of the 'make django'. You just need to add it to your crontab for it to take effect

## 6 - Backup 
  - **Database:**  use the pg_dump command, for instance, if NB was installed on host example.com, that the DB belonged to postgres used nbuser, and that the DB was called notabene, you'd use the following: 
    -pg_dump -U nbuser -h example.com -Fc notabene > nb.backup.YYYYMMDD
  
  - **Uploaded PDF files:** Use your favorite file backup technique (tar, rdiff-backup, unison etc...) to backup the directory: 
    "%s%s" % (settings.HTTPD_MEDIA,settings.REPOSITORY_DIR) (cf your settings.py files for actual values). 

## 7 - Restore (change localhost to your server name if it's not on loacalhost)
    dropdb  -U nbadmin -h localhost notabene
    createdb -U nbadmin -h localhost notabene
    pg_restore -U nbadmin -h localhost -d nb3 YOUR_NB_BACKUP_FILE

## 8 - [Advanced] Updating the code base
What is a bit delicate is that database structure may change: You need to add the missing fields manually, as described at http://www.djangobook.com/en/2.0/chapter10.html#making-changes-to-a-database-schema

**IMPORTANT:** Make a database backup (cf. step 6) so you can revert to the previous state if you encounter a problem

    git fetch 
    git diff master origin master apps/base/models.py
    #for each difference, add the corresponding field in the database. 
    #once everything is added: 
    git merge
    apachectl restart
    grunt  

**Note:** If you encounter a problem, and would like to restore things as they were before you updated the code and the database:  

    # restore your database (cf 7) 
    git reset --hard HEAD@{1} # revert to the version of the code you were at
    apachectl restart
    grunt      

# Questions
For questions (including how to install NB on other linux distributions), use our forum at
* http://nbproject.vanillaforums.com/ more specifically, the deployers section:
* http://nbproject.vanillaforums.com/categories/deployers

Contact: nb-team@csail.mit.edu (NOTE: You'll likely get a much faster reply if you use the forum above).

# VIDEO ANNOTATOR UPDATE
Since this commit, NB runs on Django 1.7.  This means changes to the database can now be handled with Django migrations!
These instructions will get you up and running from a previous NB install.

### 1) Run the database migrations included in the codebase
    cd apps
    ./manage.py migrate

### 2) Add the new user option
    python base/jobs.py addsettings
(If this fails try changing the id fields in the function do_add_tag_email_setting)


### 3) Schedule the CRON job for automated tag reminders
(The following job runs daily at 8 PM)

    0 20 * * * (cd /home/ubuntu/nbgit/apps;  python -m 'base.jobs' tagreminders)

The tagreminders option of the base/jobs.py script sends reminders to all users about unseen tagged comments.
 
