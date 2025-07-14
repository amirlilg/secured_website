# **Guide: Deploying and Securing a Web-app on AlmaLinux**

This guide documents the complete process of deploying a simple Python Flask web application on a VPS running AlmaLinux. It covers setting up a production-ready web server stack with Gunicorn and Nginx, troubleshooting common errors like a 502 Bad Gateway, and securing the site with a free SSL/TLS certificate from Let's Encrypt.

## **Step 1: Create a Basic Flask Application**

The first step is to create the application that will be served.

1. **SSH into your AlmaLinux server.**  
2. **Create a project directory** and a Python virtual environment to isolate dependencies.  
   mkdir simple\_flask\_app  
   cd simple\_flask\_app  
   python3 \-m venv venv  
   source venv/bin/activate

3. **Install necessary Python packages**: Flask for the web framework and Gunicorn for the application server.  
   pip install Flask Gunicorn

4. **Create the application file** (app.py):  
   nano app.py

5. **Add the following code** to app.py:  
   from flask import Flask

   app \= Flask(\_\_name\_\_)

   @app.route("/")  
   def hello():  
       return "Hello from AlmaLinux\! My secure app is working."

   if \_\_name\_\_ \== "\_\_main\_\_":  
       app.run(host='0.0.0.0')

## **Step 2: Configure Server Environment**

Next, we install and configure the Nginx web server, which will act as a reverse proxy, and set up the server's firewall.

1. **Install Nginx**:  
   sudo dnf install nginx \-y

2. **Start and enable the Nginx service** to ensure it runs on boot:  
   sudo systemctl start nginx  
   sudo systemctl enable nginx

3. **Configure the firewalld firewall** to allow web traffic:  
   sudo firewall-cmd \--permanent \--add-service=http  
   sudo firewall-cmd \--permanent \--add-service=https  
   sudo firewall-cmd \--reload

4. **Create an Nginx configuration file** for your application. We will use the free sslip.io service for a test domain. Replace YOUR\_IP\_ADDRESS with your server's actual IP.  
   sudo nano /etc/nginx/conf.d/simple\_flask\_app.conf

5. **Add the reverse proxy configuration**. This tells Nginx to forward requests for your domain to the Gunicorn application running on port 8000\.  
   server {  
       listen 80;  
       \# Use a domain like myapp-YOUR\_IP\_ADDRESS.sslip.io  
       server\_name myapp-217-197-97-40.sslip.io; 

       location / {  
           proxy\_pass http://127.0.0.1:8000;  
           proxy\_set\_header Host $host;  
           proxy\_set\_header X-Real-IP $remote\_addr;  
           proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;  
       }  
   }

6. **Test and restart Nginx**:  
   sudo nginx \-t  
   sudo systemctl restart nginx

## **Step 3: Run the App and Troubleshoot 502 Error**

With Nginx configured, we run the Flask app with Gunicorn and resolve the inevitable 502 Bad Gateway error caused by SELinux.

1. **Run Gunicorn** from your project directory (\~/simple\_flask\_app):  
   gunicorn \--workers 3 \--bind 0.0.0.0:8000 app:app

   At this stage, visiting your http:// domain will result in a 502 error because SELinux is blocking Nginx from connecting to port 8000\.  
2. **Fix the SELinux Policy**: In a separate terminal, run the following command to allow the web server (httpd) to make network connections. The \-P flag makes the change persistent.  
   sudo setsebool \-P httpd\_can\_network\_connect 1

3. **Restart Nginx** to apply the policy change:  
   sudo systemctl restart nginx

   With Gunicorn still running, refresh the website. The 502 error will be gone, and you should see your Flask app's message.

## **Step 4: Secure the Domain with HTTPS**

The final step is to obtain and install a free SSL/TLS certificate from Let's Encrypt using the Certbot tool.

1. **Install the EPEL repository** which contains the Certbot package:  
   sudo dnf install epel-release \-y

2. **Install Certbot and its Nginx plugin**:  
   sudo dnf install certbot python3-certbot-nginx \-y

3. **Run Certbot**: It will automatically detect your domain from the Nginx configuration, obtain a certificate, and update the Nginx configuration for you.  
   \# Replace with your actual sslip.io domain  
   sudo certbot \--nginx \-d myapp-217-197-97-40.sslip.io

4. **Follow the on-screen prompts**:  
   * Enter your email for renewal notices.  
   * Agree to the Terms of Service.  
   * **Choose option 2** when asked if you want to redirect HTTP traffic to HTTPS. This is crucial for security.  
5. **Verify Automatic Renewal** (Optional but recommended):  
   sudo certbot renew \--dry-run

### **Conclusion**

You have successfully deployed a Flask application on an AlmaLinux server. The application is served by a robust Gunicorn/Nginx stack and is fully secured with HTTPS, ensuring all data transmitted between your users and the server is encrypted.