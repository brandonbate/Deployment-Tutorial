# Deployment Tutorial

How to host your site:

1. Create an Amazon Lightsail instance. There are many configuration options available. Choose the "Linux/Unix" blueprint. Click "OS Only" and select the "Amazon Linux 2" option. Get the static IP address for it. For instance, 3.83.146.181 is an IP address I have a for a Lightsail instance.
2. Let me know your IP address. I can then have your your subdomain point to that IP address. For instance, I have brandon.bearcornfield.com point to 3.83.146.181.
3. Open a console to your Lightsail instance. Install the nginx web server by running the following command at the console:
```
sudo amazon-linux-extras install nginx1
```
Launch the nginx web server by running:
```
sudo systemctl start nginx
```
Check that the web server is running by visiting either
```http://(your static IP)``` or ```http://(your subdomain address)```.

4. We need to configure the nginx server. This is a bit complicated.
Navigate ```/etc/nginx/```. Then running
```
sudo nano nginx.conf
```
This will open a configuration file for nginx. Scroll down until you find
a line that begins with ```http {```. Inside this block, insert
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
We can test of the above configuration step worked by having nginx reload its
configuration files:
```
sudo systemctl reload nginx
```
On a web browser, visit either
```http://(your static IP)``` or ```http://(your subdomain address)```.
You should see an error message ```502 Bad Gateway```.

5. We have a ```502 Bad Gateway``` error because we are not running a Django server instance.
Navigate to your home directory using ```cd ~```. Then visit github.com and find your responsitory for the online forum project. Click on the "<> Code" button to get an https link to your respository. Save this somewhere convenient for now (like notepad). Then click on your avatar button. This will produce a drop down menu. Select "Settings". On the left side panel, click "<> Developer Settings". On the left side panel, under "Personal Access Tokens" click "Tokens (classic)". Click "Generate new token". Under "Select scopes", click on the "repo" check box. Give an appropriate Note and Expiration Date. You can then click to copy this access token.

Return to the console for Lightsail. Enter the following:
```
git clone your_repository_address
```
You should replace ```your_repository_address``` with the http link you copied earlier for your
repository.  You will be promted to enter your username. After that, provide the personal access token you generated as your password. This will copy your repository to the Lightsail instance.

6. Navigate into the folder for your repository. 
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
./virtualenv/bin/python manage.py runserver
```
to launch the Django server.
On a web browser, visit either
```http://(your static IP)``` or ```http://(your subdomain address)```.
You should now see your site.


