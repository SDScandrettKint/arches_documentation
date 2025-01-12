# Installing Arches and creating a new project 
This documentation is specifically for Arches version 7+.
Between 6 and 7, lots of features have been removed or changed such as:
- Elasticsearch upgraded from 7 to 8
- The introduction of webpack to manage front end bundles
- collectstatic command has been deprecated for build_production


## Steps
1. Create a new user to install arches under 

   ```
   sudo adduser archesadmin
   sudo usermod -aG sudo archesadmin
   ```

2. Update all packages

   ```
   sudo apt-get update
   ```

3. Clone the arches repo (as this allows for easier development in the future) and checkout the latest stable version
   ```
   git clone https://github.com/archesproject/arches.git
   cd arches 
   git checkout {VERSION}
   cd
   ```

4.  Create virtual python environment 
    ```
    sudo apt-get update
    sudo apt install python3.x-venv     [match version with x]
    sudo apt-get install python3-dev
    python3 -m venv env
    ```

5. Activate the virtual enviroment

   ```
   source env/bin/activate
   ```

6. Install software dependencies 
   ```
   sudo apt-get install gcc
   pip install wheel
   ```
   Dependencies such as postgres, elasticsearch, yarn, node etc can be installed using
   ```
   yes | sudo bash ~/arches/arches/install/ubuntu_setup.sh
   ```
   However, it is typically bad practice to install ES via apt-get.
   ```
   wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.4.3-linux-x86_64.tar.gz
   ```
   Unzip the tar file using `tar -xvf <the .tar.gz file>`
   Run ES in daemon mode using `bin/elasticsearch -d`
   See ES running by `ps aux | grep elastic` and kill using the pid and `kill -9 pid`
   
   In core Arches, on the level of package.json run `yarn install`

7. Install Arches
   ```
   cd arches
   pip install -e .
   ```

8. Check postgres and es
   ```
   psql -U postgres
   \q 
   curl localhost:9200
   ```

9. Create a project
   ```
   cd
   arches-project create project_name
   ```
   so that `/home/archesadmin/project_name`


## Elasticsearch settings
In the ES directory we created earlier, in config/elasticsearch.yml ensure that `http.port: 9200` is uncommented/added.
Set `xpack.security.enabled` to `true`

Reset the password for the default "elastic" user:
```
bin/elasticsearch-reset-password -u elastic
```

If the arches instance is http, then copy the following core arches line into the project settings.py and change it to include http not https
```
ELASTICSEARCH_HOSTS = [{"scheme": "http", "host": "localhost", "port": ELASTICSEARCH_HTTP_PORT}]
```

Add the password to the following line in settings.py
```
ELASTICSEARCH_CONNECTION_OPTIONS = {"timeout": 30, "verify_certs": False, "basic_auth": ("elastic", "{PASSWORD}")}
```

## Load package

Arches packages set up a database.
If not loading a package, type the following to setup a database. `python manage.py setup_db`

1. Clone package
   ```
   git clone 'package_name' 
   ```
   HER - Package : https://github.com/archesproject/arches-her

2. Load package
   ```
   python manage.py packages -o load_package -s 'directory/to/package/' -db
   ```


## Run project

   ```
   cd project_name
   python manage.py runserver
   ```


## Setting up webpack
If an error like the following occurs:
```
Error reading /mnt/c/testing/v7arches/v7arches/v7arches/webpack/webpack-stats.json. Are you sure webpack has generated the file and the path is correct?
```
This is because we need to correct the webpack settings to point at the correct place.

In development, ensure `yarn build development` or `yarn start` is run in a separate terminal at the same time as `python manage.py runserver` is running



# Serving the project with Apache

1. install software
   ```
   sudo apt-get install apache2
   sudo apt install apache2-dev python3-dev
   pip install mod_wsgi
   mod_wsgi-express module-config
   ```
2. Copy `mod_wsgi-express module-config` output

   example: 
   ```
   LoadModule wsgi_module "/home/archesadmin/env/lib/python3.8/site-packages/mod_wsgi/server/mod_wsgi-py38.cpython-38-x86_64-linux-gnu.so"
   WSGIPythonHome "/home/archesadmin/env"
   ```

3. Create an apache config file for arches

   ```
   sudo nano /etc/apache2/sites-available/arches-default.conf 
   ```
   Then copy the mod_wsgi-express module-config output and pase it on top o fhte document above `<VirtualHost>`

   Populate the document with the following

   _Ensure alias is changed to static_

```
# If you have mod_wsgi installed in your python virtual environment, paste the text generated
# by 'mod_wsgi-express module-config' here, *before* the VirtualHost is defined.

<VirtualHost *:80>

    WSGIDaemonProcess arches python-path=/home/archesadmin/project_name
    WSGIScriptAlias / /home/archesadmin/project_name/project_name/wsgi.py process-group=arches

    # Necessary to support Arches Collector
    WSGIPassAuthorization on

    ## Uncomment the ServerName directive and fill it with your domain
    ## or subdomain if/when you have your DNS records configured.
    # ServerName heritage-inventory.org

    <Directory /home/archesadmin/project_name>
        Require all granted
    </Directory>

    # This section tells Apache where to find static files. This example uses
    # STATIC_URL = '/media/' and STATIC_ROOT = os.path.join(APP_ROOT, 'static')
    # NOTE: omit this section if you are using S3 to serve static files.
    Alias /static/ /home/archesadmin/project_name/project_name/static/
    <Directory /home/archesadmin/project_name/project_name/static/>
        Require all granted
    </Directory>

    # This section tells Apache where to find uploaded files. This example uses
    # MEDIA_URL = '/files/' and MEDIA_ROOT = os.path.join(APP_ROOT)
    # NOTE: omit this section if you are using S3 for uploaded media
    Alias /files/uploadedfiles/ /home/archesadmin/project_name/project_name/uploadedfiles/
    <Directory /home/archesadmin/project_name/project_name/uploadedfiles/>
        Require all granted
    </Directory>

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
    # error, crit, alert, emerg.
    # It is also possible to configure the loglevel for particular
    # modules, e.g.
    #LogLevel info ssl:warn
    # Recommend changing these file names if you have multiple arches
    # installations on the same server.
    ErrorLog /var/log/apache2/error-arches.log
    CustomLog /var/log/apache2/access-arches.log combined

</VirtualHost>
```

4. Enable the new config created in apache
   ```
   sudo a2dissite 000-default
   sudo a2ensite arches-default
   sudo service apache2 reload
   ```

3. Create static files directory 
   ```
   mkdir project_name/project_name/static
   ```

4. Change `settings.py` to following 
   ```
   STATIC_ROOT = os.path.join(APP_ROOT, 'static')
   STATIC_URL = '/static/'
   ```
   and comment out `#STATIC_ROOT = '/var/www/media'`


   finally run
   ```
   python manage.py collectstatic
   ```

5. Grant apache write settings =
```
cd
```
```
sudo chmod -R 664 /home/ubuntu/Projects/demo_project/demo_project/arches.log
sudo chgrp -R www-data /home/ubuntu/Projects/demo_project/demo_project/arches.log
```
```
sudo chmod -R 775 /home/ubuntu/Projects/demo_project/demo_project/uploadedfiles
sudo chgrp -R www-data /home/ubuntu/Projects/demo_project/demo_project/uploadedfiles
```
```
sudo chmod -R 775 /home/ubuntu/Projects/demo_project/demo_project
sudo chgrp -R www-data /home/ubuntu/Projects/demo_project/demo_project
```
```
sudo chmod -R 775 /home/ubuntu/Projects/demo_project/demo_project/static
sudo chgrp -R www-data /home/ubuntu/Projects/demo_project/demo_project/static
