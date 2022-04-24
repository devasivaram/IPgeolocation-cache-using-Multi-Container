# IPgeolocation-cache-using-Multi-Container

## Description

IP-based geolocation is a way that you can find the location of an internet-connected computing or mobile device. To get started, all you need is your target’s IP address and a geolocation lookup tool. A geolocation lookup tool canvasses public databases to determine the contact and registration information for a particular IP address. With both of these tools in hand, you simply input the IP address into the geolocation lookup tool and you will receive the location of your target.

With an IP address, you can access a wide range of information. To reiterate, however, an IP address isn’t enough. You need to use a geolocation database to obtain this user information. The most basic information provided in most geolocation databases includes the continent, country, state/region, city, and time zone of the electronic device. But along with this, you can discover the internet service provider (“ISP”), approximate longitude and latitude, and sometimes even the relevant organization attached to the device.

One of the most significant benefits of accessing your data through an API is that the database is constantly updated. In addition, the onus is on the third-party provider to ensure that the data is available to your application. This allows you to focus on your product rather than on building and maintaining the database. There are some downsides, however. API databases may also contain some downtime, which may be inconvenient when you need the data. API databases, while they contain more updated information, may also limit the number of requests that you can make per day.

This API is free to use up to 1,000 requests/month. Paid plans start at $15/month.

In order to reduce the request count sent to API, we are making use of caching service with the use of Redis.

![image](https://user-images.githubusercontent.com/100773863/163928881-d764a2ff-7727-4a65-bdde-fbb9b68c91c9.png)

When a user makes a request, the API service calls Redis to see if the IP is cached, and if it isn't, the API connects the ipgeolocation database to acquire the IP information. Rather than sending the information to the user, the API service caches it in Redis before giving it to the user. So, the following time a user makes a request for the same IP, there is no need for an API request because the information can be retrieved from Redis cache, reducing API queries.

If there are more than one connection to an API at the same time, it can't handle them all at once, thus we're moving to a multi-layer architecture with three front-end services handled by a load balancer. See the final architecture of the app.



## Pre-requests
- A running 2 Amazon Linux instance with docker installed
- A domain name which points to the EC2 instance
- Get API key from: [API_KEY](https://app.ipgeolocation.io/auth/login)
- SSL for the domain

##Procedure

>To create compose file:

We will create an directory for our project with name "compose-iplocation" and we create the docker compose file inside that.

~~~sh
mkdir compose-iplocation
cd compose-iplocation
vim docker-compose.yml
~~~

> Add this configuration to yml file:

~~~

version: "3"

services:

    redis:
      image: redis:latest
      container_name: ipgeolocation-caching-service
      networks:
        - ipgeolocation-network
      ports:
        - "6379:6379"

    api-service:
      image: devanandts/apiimage:custom
      container_name: api-service
      networks:
        - ipgeolocation-network
      env_file: .env
      environment:
        REDIS_HOST: ipgeolocation-caching-service
        REDIS_PORT: 6379
        APP_PORT: "8080"
        API_KEY: $api_key
      ports:
        - "8080:8080"

    frontend1:
      image: devanandts/frontimagenew:custom
      container_name: front-service1
      networks:
        - ipgeolocation-network
      environment:
        API_SERVER: api-service
        API_SERVER_PORT: 8080
        API_PATH: /api/v1/
        APP_PORT: 8080
      ports:
        - "8081:8080"

    frontend2:
      image: devanandts/frontimagenew:custom
      container_name: front-service2
      networks:
        - ipgeolocation-network
      env_file: .env
      environment:
        API_SERVER: api-service
        API_SERVER_PORT: 8080
        API_PATH: /api/v1/
        APP_PORT: 8080
      ports:
        - "8082:8080"

    frontend3:
      image: devanandts/frontimagenew:custom
      container_name: front-service3
      networks:
        - ipgeolocation-network
      env_file: .env
      environment:
        API_SERVER: api-service
        API_SERVER_PORT: 8080
        API_PATH: /api/v1/
        APP_PORT: 8080
      ports:
        - "8083:8080"

    nginx:
      image: nginx:1.15.12-alpine
      container_name: nginx
      networks:
        - ipgeolocation-network
      volumes:
        - ./nginx-conf/:/etc/nginx/conf.d
        - ./ssl:/etc/nginx/certs
      ports:
        - "80:80"
        - "443:443"

volumes:
  cache:

networks:
  ipgeolocation-network:
~~~

We need to set the .env file with the API-KEY and we need to purchase SSL and add it to the directory inside our project directory, here I created directory "SSL".

~~~sh
vim .env
~~~

>added: 

api_key= "your/API_KEY"

I used this site for purchasing [SSL](https://punchsalad.com/ssl-certificate-generator/) and downloaded the SSL certificate file and key file, after that we placed both files to "/home/ec2-user/compose-iplocation/ssl/"

***Nginx conf contains the upstream section and proxy pass section:***

~~~
upstream loadbalancer {
        server front-service1:8081;
        server front-service2:8082;
        server front-service3:8083;
}

server {
        listen 80;
        listen [::]:80;
        server_name api.devanandts.tk;
        server_tokens off;
        location / {
                return 301 https://api.devanandts.tk$request_uri;
        }
}

server {
     listen 443 ssl;
     server_name api.devanandts.tk;
     ssl_certificate /etc/nginx/certs/dev.crt;
     ssl_certificate_key /etc/nginx/certs/dev.key;
     location / {
         proxy_pass http://loadbalancer;
     }
}
~~~

Here we use six containers, which are:
 - ipgeolocation-caching-service: For caching by making use of redis:latest docker image.
 - api-service: For API Layer
 - front-service1 with port 8081
 - front-service2 with port 8082
 - front-service3 with port 8083
 - nginx: A reverse proxy using Nginx image. 

All these services are created in a network called 'ipgeo-grid'.

***Our structure will be like:***

![image](https://user-images.githubusercontent.com/100773863/163941550-4ae0d47b-0c6d-40fc-968c-3e0b810472fd.png)


The Flask app is used to build the images for api-service and front-services. Both are available in my other repository [api-service](https://github.com/devasivaram/ipgeolocation-api-service-docker.git) & [front-service](https://github.com/devasivaram/ipgeolocation-frontend-docker.git)

## Verify the yml file and start the compose
~~~sh
docker-compose config
docker-compose up -d
~~~

Once those are up and running, we can check the IP info by using the URL http://api.devananats.tk/ip/ followed by the required IP like http://api.devananats.tk/ip/80.80.80.80  

![image](https://user-images.githubusercontent.com/100773863/163940240-c7720844-0b55-4d98-b91f-8b15abde596d.png)


![image](https://user-images.githubusercontent.com/100773863/163940179-4d767d52-ffd2-4ede-88ae-160eca61b651.png)

You can see that when I first request an IP, the cached status is false, but when I request the same IP again, the cached status changes to True, indicating that when we make requests for the same IP, the result is received from the cache service rather than an API request, reducing API queries. Because we are utilising a multi-container system, you can also observe that the results are fetched from different containers by looking at the container hostname or IP.

## Conclusion
This is how we create ipgeolocation-cache service using docker-compose tool with multi-container. Please contact me when you encounter any difficulty error while using this terrform code. Thank you and have a great day!


### ⚙️ Connect with Me
<p align="center">
<a href="https://www.instagram.com/dev_anand__/"><img src="https://img.shields.io/badge/Instagram-E4405F?style=for-the-badge&logo=instagram&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/dev-anand-477898201/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a>
