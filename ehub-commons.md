## What is the earthquake-hub-commons intended for
This repository integrates all the essential programs necessary for hosting a citizen science network of ground motion sensors(footnote: such as but not limited to raspberryshakes). It enables data transmission, archiving, and allows feeding the network data to earthquake detection software(footnote: such as but not limited to SeisComP).
A version of this repository is deployed live on [earthquake.science.upd.edu.ph](https://earthquake.science.upd.edu.ph)

## Server Deployment via Docker Compose
There are two different docker-compose files written for two scenarios, *(1)* for *deploying in a server* and *(2)* for *development/testing* in your local machine.
1. **Clone the repository:** Begin by cloning [this repository](https://github.com/UPRI-earthquake/earthquake-hub-commons.git) to your local machine using the git clone command.
2. **Install Docker:** Make sure you have [Docker](https://docs.docker.com/get-docker/) installed on your system. Docker Compose is essential for managing multi-container Docker applications.
3. **Configure env variables:** Create a file named .env in the root of the repository, and add the necessary configuration variables. You can find an example of these variables in the .env.example file.
4. **Configure ringserver:** Make sure that ringserver-configs/auth/secret.key exists (contains brgy token to AuthServer). Then set the *AuthServer* value in ringserver-configs/ring.conf to the AuthServer API address (i.e. http://172.21.0.3:5000 or https://earthquake.science.upd.edu.ph/api).
5. **Set up nginx certificates:** For this, there will be another series of steps to be done. These steps will only be done during development/testing to replicate the deployment server enviroment into your local machine; to ensure that you have valid SSL certificates for Nginx.
    - Create a locally trusted self-signed SSL certificate (you may use [mkcert](https://www.howtoforge.com/how-to-create-locally-trusted-ssl-certificates-with-mkcert-on-ubuntu/) to do this). And store the `pem` files in `https_data/certbot/conf/live/<server-name>/` 
    - Configure the [nginx configuration file](https_data/nginx.dep-test.d/nginx.dep-test.conf):
        > ℹ️ If you name your `pem` files as `localhost.pem` and `localhost-key.pem`, and then store them in the folder `https_data/certbot/conf/live/localhost/`, then you shouldn't have to alter the nginx configuration file.
        * `server_name` must match the one set for the certificates
        * `ssl_certificate` and `ssl_certificate_key` should both correspond to the location and filenames of the previously generated `pem` files.
6. **Run the containers:** Once the setup is configured, run the following Docker Compose command to start all the necessary containers:
    ```bash
    docker compose --env-file .dep-test.env up --build
    ```
    This command will start the following containers:
    - **Nginx Proxy:** This acts as a reverse proxy server which handles incoming HTTP/HTTPS traffic and distributes it to the corresponding services within the Docker network.
    - **Earthquake-hub-frontend:** This hosts the front-end application for the earthquake-hub network which serves the user interface and interacts with the backend services to display data and handle user requests.
    - **Earthquake-hub-backend:** This hosts the back-end application for the earthquake-hub network. It handles various functionalities, such as user authentication, data processing, and database interactions, to support the front-end application and process incoming data from the sensors.
    - **MongoDB:** This is a NoSQL database used to store and manage data, such as account information, device information, and recorded seismic events, within the earthquake-hub network.
    - **Ringserver:** This is a TCP-based ring buffer designed for packetized streaming data which utilizes time-series data from various station, archive seismic data, and serve those data (or diagnostics of such data) towards the clients.
    - ***Certbot:** This is a tool used to automatically obtain and manage SSL/TLS certificates from the Let's Encrypt Certificate Authority that provides valid SSL certificate for the server’s domain which ensures secure communication between the server and its clients. You may refer to this [post](https://mindsers.blog/post/https-using-nginx-certbot-docker/) on how to request SSL certificate using docker compose.  
      - ***Note that Certbot needs to be renewed `every three (3) months` to keep the certificates valid and up-to-date. Certificate renewal process can easily be done using the following command:*
        ```bash
        docker compose run --rm certbot renew
        ```
    - ***Geoserve:** This is a service that provides geographic information for the earthquake-hub network. It is used to convert latitude and longitude values into names of places, such as their city name, region, and country name.
        > ℹ️ *Note that Certbot and Geoserve are not used in local development/testing. The two will only be run using the docker compose which is intended for deploying in a server.  
        ** Certbot needs to renew the SSL certificates every three (3) months.

## Interfacing with a processor
### SeisComP
The earthquake-hub-commons repository allows easy interfacing with earthquake processing softwares such as SeisComP (Seismic Communication Processor) which is a widely used earthquake detection  and seismic data processing software. 
SeisComP is a collection of *software modules* used to for seismic data transmission, processing, analysis, monitoring, and archiving. These functions can be done in both real-time and offline. 
For further information about SeisComP check this [overview](https://www.seiscomp.de/doc/index.html) on how to get started with SeisComP.

### Backend Endpoints and Python Scripts 
This repository exposes two Server-Sent-Event endpoints (`/messaging/restricted/new-pick` and `/messaging/restricted/new-event`) and run two Python scripts to interact with the processor. The two python scripts are run as services which publish `pick events` and `recorded seismic events` from SeisComP to the aforementioned endpoints.

### MongoDB and Data Population
We have included scripts to help initialize mongodb’s `events` collection with SeisComP event data in the scenario that you have an existing record of SeisComP detections that you want to migrate into your instance of EarthquakeHub.