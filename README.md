# Deployment Tutorial

How to host your site:

1. Create an Amazon Lightsail instance. There are many configuration options available. Choose the "Linux/Unix" blueprint. Click "OS Only" and select the "Amazon Linux 2" option. Get the static IP address for it. For instance, 3.83.146.181 is an IP address I have a for a Lightsail instance. Let me know your IP address. I can then have your your subdomain point to that IP address. For instance, I have brandon.bearcornfield.com point to 3.83.146.181.
2. Open a console to your Lightsail instance. Install the nginx web server by running the following command at the console:
```
sudo amazon-linux-extras install nginx1
```
Launch the nginx web server by running:
```
sudo systemctl start nginx
```
Check that the web server is running by visiting either
```http://(your static IP)``` or ```http://(your subdomain address)```.

3. We need to configure the nginx server. This is a bit complicated.
Navigate ```/etc/nginx/```. Then running
```
sudo nano nginx.conf
```
This will open a configuration file for nginx.
Replace ```user nginx;``` with ```user ec2-user```.
Then scroll down until you find a line that begins with ```http {```. Inside this block, insert
```
include /etc/nginx/sites-enabled/*;
```
Save the file. Next, we create two folders by running the following commands:
```
mkdir sites-available
mkdir sites-enabled
```
Navigate to ```/etc/nginx/sites-available/```.
Run ```sudo nano``` to create a configuration file in this folder.
Enter the following content on ```nano```:
```
server {
	listen 80;
	server_name your_name.bearcornfield.com;
	
	location / {
		proxy pass http://localhost:8000;
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

4. We have a ```502 Bad Gateway``` error because we are not running a Django server instance.
Navigate to your home directory using ```cd ~```. Then visit github.com and find your responsitory for the online forum project. Click on the "<> Code" button to get an https link to your respository. Save this somewhere convenient for now (like notepad). Then click on your avatar button. This will produce a drop down menu. Select "Settings". On the left side panel, click "<> Developer Settings". On the left side panel, under "Personal Access Tokens" click "Tokens (classic)". Click "Generate new token". Under "Select scopes", click on the "repo" check box. Give an appropriate Note and Expiration Date. You can then click to copy this access token.

Return to the console for Lightsail. Enter the following:
```
git clone your_repository_address
```
You should replace ```your_repository_address``` with the http link you copied earlier for your
repository.  You will be promted to enter your username. After that, provide the personal access token you generated as your password. This will copy your repository to the Lightsail instance.

5. Navigate into the folder for your repository. 
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

6. We will replace the Django server with Gunicorn. We install Gunicorn with the following command:
```
./virtualenv/bin/pip install gunicorn
```
You can then run your application by running
```
./virtualenv/bin/gunicorn my_app.wsgi:application
```
This will display your site, but it will fail to include any static file content such as CSS files.
Terminate Gunicorn by entering Ctl-C.

7. On your local machine (not Lightsail), begin working in the repository for your project. Edit ```settings.py``` so that it contains the following:
```
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR,'static')
```
In the same file, under ```INSTALLED_APPS``` be sure to comment out 'django.contrib.admin'.
After activating the virtual environment for this project, run
```
python manage.py collectstatic
```
This will generate a static folder for our static files.
Update your repository and push it to Github. Then on your Lightsail server, pull these updates. You will need the personal access token from earlier to do this. In the Lightsail console, use ```sudo nano``` to edit 
```/etc/nginx/sites-available/your_name.bearcornfield.com```. Add the following to this configuration file:
```
	location /static {
		alais /home/ec2-user/your_project_folder/static;
	}
```
Save the file and then return to the folder for your project.
We have nginx reload the configuration files with the following command:
```
sudo systemctl nginx reload
```
Then restart Gunicorn with
```
./virtualenv/bin/gunicorn my_app.wsgi:application
```
