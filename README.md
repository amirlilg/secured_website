# **Guide: Deploying and Securing a Web-app on AlmaLinux**

This guide documents the complete process of deploying a simple Python Flask web application on a VPS running AlmaLinux. It covers setting up a production-ready web server stack with Gunicorn and Nginx, troubleshooting common errors like a 502 Bad Gateway, and securing the site with a free SSL/TLS certificate from Let's Encrypt.

## **Prerequisite: Web App**

As an example, here's a Flask application:

1. **Install necessary Python packages**: Flask for the web framework and Gunicorn for the application server.  
```shell
pip install Flask Gunicorn
```

2. **Create your webapp.** Simple flask app (app.py):
```python
from flask import Flask

app = Flask(__name__)

   @app.route("/")  
   def hello():  
       return "Hello world! My app is working."

if __name__=="__main__":  
   app.run(host='0.0.0.0')
```

## **Step 1: Configure Server Environment**

Install and configure the Nginx web server, which will act as a reverse proxy, and set up the server's firewall.

1. **Install Nginx**:
```
sudo dnf install nginx -y
```  

2. **Start and enable the Nginx service** to ensure it runs on boot:  
```shell
sudo systemctl start nginx  
sudo systemctl enable nginx
```

3. **Configure the firewalld firewall** to allow web traffic: 
```shell
sudo firewall-cmd --permanent --add-service=http  
sudo firewall-cmd --permanent --add-service=https  
sudo firewall-cmd --reload
```

4. **Create an Nginx configuration file** for your application. We will use the free sslip.io service for a test domain. Replace YOUR_IP_ADDRESS with your server's actual IP. 
```shell
sudo nano /etc/nginx/conf.d/simple_flask_app.conf
``` 

5. **Add the reverse proxy configuration**. This tells Nginx to forward requests for your domain to the Gunicorn application running on port 8000.  
```console
server {  
      listen 80;  
      # Enter your domain. 
      # Here, we used sslip.io's free subdomain service.  
      server_name myapp-192-168-0-0.sslip.io; 

      location / {  
         proxy_pass http://127.0.0.1:8000;  
         proxy_set_header Host $host;  
         proxy_set_header X-Real-IP $remote_addr;  
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
      }  
}
```

6. **Test and restart Nginx**:  
```shell
sudo nginx -t  
sudo systemctl restart nginx
```

8. **Run Gunicorn** from your project directory:  
```shell
gunicorn --workers 3 --bind 0.0.0.0:8000 app:app
```
In a browser, reload the domain you introduced. SELinux is blocking Nginx from connecting to port 8000, which results in `502 Bad Gateway error`.

b7. **Fix the SELinux Policy**: Run the following command to allow the web server (httpd) to make network connections. The -P flag makes the change persistent.  
```shell
sudo setsebool -P httpd_can_network_connect 1
```

8. **Restart Nginx** to apply the policy change:  
```shell
sudo systemctl restart nginx
```
With Gunicorn still running, refresh the website. The 502 error will be gone, and you should see your Flask app's message.

## **Step 2: Secure the Domain with HTTPS**

The final step is to obtain and install a free SSL/TLS certificate from Let's Encrypt using the Certbot tool.

1. **Install the EPEL repository** which contains the Certbot package:  
```shell
sudo dnf install epel-release -y
```

2. **Install Certbot and its Nginx plugin**:  
```shell
sudo dnf install certbot python3-certbot-nginx -y
```   

3. **Run Certbot**: It will automatically detect your domain from the Nginx configuration, obtain a certificate, and update the Nginx configuration for you.  
     
```shell 
# Replace with your actual sslip.io domain
sudo certbot --nginx -d myapp-192-168-1-1.sslip.io
```

4. **Follow the on-screen prompts**:  
   * Enter your email for renewal notices.  
   * Agree to the Terms of Service.  
   * **Choose option 2** when asked if you want to redirect HTTP traffic to HTTPS. This is crucial for security.  
5. **Verify Automatic Renewal** (Optional but recommended):
```shell
sudo certbot renew --dry-run
```  

### **Conclusion**

We have successfully deployed a Flask application on an AlmaLinux server. The application is served by a robust Gunicorn/Nginx stack and is fully secured with HTTPS, ensuring all data transmitted between our users and the server is encrypted.