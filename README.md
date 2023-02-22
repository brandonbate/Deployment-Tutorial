# Deployment Tutorial

How to host your site:

### Step 1
Create an Amazon Lightsail instance. There are many configuration options available. Choose the "Linux/Unix" blueprint. Click "OS Only" and select the "Amazon Linux 2" option. Get the static IP address for it. For instance, 3.83.146.181 is an IP address I have a for a Lightsail instance. Let me know your IP address. I can then have your your subdomain point to that IP address. For instance, I have brandon.bearcornfield.com point to 3.83.146.181.

### Step 2
Open a console to your Lightsail instance. Install the nginx web server by running the following command at the console:
```
sudo amazon-linux-extras install nginx1
```
Launch the nginx web server by running:
```
sudo systemctl start nginx
```
Check that the web server is running by visiting either
```http://(your static IP)``` or ```http://(your subdomain address)```.

### Step 3
We need to configure the nginx server. This is a bit complicated.
Navigate to ```/etc/nginx/```. Then run
```
sudo nano nginx.conf
```
This will open a configuration file for nginx.
Replace ```user nginx;``` with ```user ec2-user;```.
Then scroll down until you find a line that begins with ```http {```. Inside this block, insert
```
include /etc/nginx/sites-enabled/*;
```
Save the file. Next, we create two folders by running the following commands:
```
sudo mkdir sites-available
sudo mkdir sites-enabled
```
Navigate to ```/etc/nginx/sites-available/```.
Run ```sudo nano``` to create a configuration file in this folder.
Enter the following content on ```nano```:
```
server {
	listen 80;
	server_name your_name.bearcornfield.com;
	
	location / {
		proxy_pass http://localhost:8000;
		proxy_set_header Host $host;
	}
}
```
Save this to a file called ```your_name.bearcornfield.com``` within 
```/etc/nginx/sites-available/```. Replace ```your_name``` with your actual first name.
Navigate to ```/etc/nginx/sites-enabled/```.
We create a symbolic link to ```/etc/nginx/sites-available/your_name.bearcornfield.com```
with the command:
```
sudo ln -s /etc/nginx/sites-available/your_name.bearcornfield.com your_name.bearcornfield.com
```
We can test the above configuration step worked by having nginx reload its
configuration files:
```
sudo systemctl reload nginx
```
On a web browser, visit either
```http://(your static IP)``` or ```http://(your subdomain address)```.
You should see an error message ```502 Bad Gateway```.

### Step 4
We have a ```502 Bad Gateway``` error because we are not running a Django server instance.
Navigate to your home directory using ```cd ~```. Then visit github.com and find your responsitory for the online forum project. Click on the "<> Code" button to get an https link to your respository. Save this somewhere convenient for now (like notepad). Then click on your avatar button. This will produce a drop down menu. Select "Settings". On the left side panel, click "<> Developer Settings". On the left side panel, under "Personal Access Tokens" click "Tokens (classic)". Click "Generate new token". Under "Select scopes", click on the "repo" check box. Give an appropriate Note and Expiration Date. 
Gerenate the token. You can then click to copy this access token.

Return to the console for Lightsail. Enter the following:
```
sudo yum install git
git clone your_repository_address
```
You should replace ```your_repository_address``` with the http link you copied earlier for your
repository.  You will be promted to enter your username. After that, provide the personal access token you generated as your password. This will copy your repository to the Lightsail instance.

### Step 5
Navigate into the folder for your repository. 
Create a virtual environment by running:
```
python3.7 -m venv virtualenv
```
Then install Django by running
```
./virtualenv/bin/pip install "django==1.11.17"
```
You can then generate your database and run your server by entering
```
./virtualenv/bin/python manage.py migrate
```
Before running the server, edit ```settings.py``` in your project so that you have
```
ALLOWED_HOSTS = ['*']
```
This is an insecure solution that we will improve upon shortly. Run
```
./virtualenv/bin/python manage.py runserver 8000
```
to launch the Django server.
On a web browser, visit either
```http://(your static IP)``` or ```http://(your subdomain address)```.
You should now see your site.
Terminate Django by entering Ctl-C.

### Step 6
We will replace the Django server with Gunicorn. We install Gunicorn with the following command:
```
./virtualenv/bin/pip install gunicorn
```
You can then run your application by running
```
./virtualenv/bin/gunicorn my_project.wsgi:application
```
Replace ```my_project``` with the name of your project; in our textbook example, they have ```superlists```
as their project name.
This will display your site, but it will fail to include any static file content such as CSS files.
Terminate Gunicorn by entering Ctl-C.

### Step 7
On your local machine (not Lightsail), begin working in the repository for your project. Edit ```settings.py``` so that it contains the following:
```
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR,'static')
```
In the same file, under ```INSTALLED_APPS``` be sure to comment out ```'django.contrib.admin'```.
After activating the virtual environment for this project, run
```
python manage.py collectstatic
```
This will generate a static folder for your static files.
Update your repository and push it to Github. Then on your Lightsail server, pull these updates. You will need the personal access token from earlier to do this. In the Lightsail console, use ```sudo nano``` to edit 
```/etc/nginx/sites-available/your_name.bearcornfield.com```. Add the following to this configuration file:
```
	location /static {
		alias /home/ec2-user/your_project_folder/static;
	}
```
Save the file and then return to the folder for your project.
We have nginx reload the configuration files with the following command:
```
sudo systemctl reload nginx
```
Then restart Gunicorn with
```
./virtualenv/bin/gunicorn my_project.wsgi:application
```

### Step 8
On your local machine, open up ```settings.py``` and replace
```
## SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = 'hy=ydw9b6f57nau_#u+%hh4819!lh0my$!ep#hfci=iw#hni(c'

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True

ALLOWED_HOSTS = ['*']
```
with
```
if 'DJANGO_DEBUG_FALSE' in os.environ:
    DEBUG = False
    SECRET_KEY = os.environ['DJANGO_SECRET_KEY']
    ALLOWED_HOSTS = [os.environ['SITENAME']]
else:
    DEBUG = True
    SECRET_KEY = 's#x5!*1d^7zlvbuob&=jr7dbwj%+gi+cd0cdbxo83(ls052jor'
    ALLOWED_HOSTS = []
```
Then on your local machine command line, we append ```.gitignore``` with the following command:
```
echo .env >> .gitignore
```
Commit these changes to your repository and push them to github. Go to your Lightsail console and pull these changes.
In your project folder run ```nano``` and enter the following:
```
DJANGO_DEBUG_FALSE=y
SITENAME=your_name.bearcornfield.com
DJANGO_SECRET_KEY=$(python3.7 -c"import random; print(''.join(random.SystemRandom().choices('abcdefghijklmnopqrstuvwxyz0123456789', k=50)))")
```
Save this file as ```.env```. Then run the following command to run your application by referencing these environmental variables:
```
set -a; source .env; set +a
./virtualenv/bin/gunicorn my_app.wsgi:application
```

### Step 9
We are going to enable nginx and Gunicorn to communicate using unix sockets. This is difficult to do on AWS.
On the Lightsail console, navigate to ```/var```. Then run the following command to create a folder for the
socket and to grant "appropriate permissions" to that folder:
```
sudo mkdir sockets
sudo chmod -R 777 sockets
```
Then navigate to ```/etc/nginx/sites-available``` and run 
```
sudo nano your_name.bearcornfield.com
```
Replace
```
	proxy pass http://localhost:8000;
```
with
```
	proxy_pass http://unix:/var/sockets/your_project-your_name.bearcornfield.com.socket;
```
Replace ```your_project``` with your project name. Then navigate back to your project's directory and run:
```
sudo systemctl reload nginx
./virtualenv/bin/gunicorn --bind unix:/var/sockets/your_project-your_name.bearcornfield.com.socket your_project.wsgi:application
```

I hope that if you visited your site at this point, it will be running. But it might not be.
If that's the case, then we need to update a configuration file for nginx. Navigate to
```/lib/systemd/system``` and run ```sudo nano nginx.service```. Under the ```[Services]``` header
set ```PrivateTmp = false```. Save these changes and then navigate back to your project folder.
The following command makes sure these changes take effect:
```
sudo systemctl daemon-reload
```
Then once again run the following:
```
sudo systemctl reload nginx
./virtualenv/bin/gunicorn --bind unix:/var/sockets/your_project-your_name.bearcornfield.com.socket your_project.wsgi:application
```
You should be able to view your site from a web browser.

### Step 10

We want Gunicorn to run whenever our server reboots. We do this by creating a systemd service.
Run
```
sudo nano /etc/systemd/system/gunicorn-your_project-your_name.bearcornfield.com.service
```
This will create a service file that systemd will use to boot up Gunicorn. Enter the following:
```
[Unit]
Description=Guincorn
After=network.target

[Service]
Restart=on-failure
User=ec2-user
WorkingDirectory=/home/ec2-user/your_project
EnvironmentFile=/home/ec2-user/your_project/.env
ExecStart=/home/ec2-user/your_project/virtualenv/bin/gunicorn --bind unix:/var/sockets/your_project-your_name.bearcornfield.com.socket your_project.wsgi:application


[Install]
WantedBy=multi-user.target
```
Run the following commands to load this systemd service:
```
sudo systemctl daemon-reload
sudo systemctl enable nginx
sudo systemctl enable gunicorn-your_project-your_name.bearcornfield.com
sudo systemctl start gunicorn-your_project-your_name.bearcornfield.com
```
Your website should now load.

Next, try rebooting your system to confirm that nginx and Gunicorn load as expected. You
can do this by entering:
```
sudo reboot
```
