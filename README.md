# Ridgeline Development
This repo provides configuration and details for provisioning a development environment consisting of Ghost, Nginx, and MySQL running on Ubuntu 16.04

Note: This guide and setup was all done on a Mac, though instructions should still apply on other common platforms (with OS specific tools)

Please see [Config Scenarios](#config-scenarios) later on in this document for different ways we might apply this configuration to customer projects

## Docker Environment

### Requirements

You will need the following software installed on your local machine:

- [Docker](https://www.docker.com/community-edition) (Provides docker and docker-compose)

You will also need this repo checkouted locally. 

### TL;DR

In a terminal, clone the repository, cd to the project directory, and start the services:
    
    # Clone project repo
    $ git clone https://github.com/dthornton/ridgeline.git ~/Projects/ridgeline

    # Enter the project directory
    $ cd ~/Projects/ridgeline
    
    # Start mysql first
    $ docker-compose up -d mysql
    
    # Start all other services (Ghost and Nginx)
    $ docker-compose up -d

    # Connect to Ghost by hitting http://localhost in your browser

### Usage in more detail

We will use the `docker-compose` command to launch the required containers, which are defined in the [docker-compose.yml](docker-compose.yml).

#### Start the services 

*Note: The following commands assume you are in the cloned repo (~/Projects/ridgeline).*

MySQL should be started first so it is available when we start Ghost: 

    $ docker-compose up -d mysql

This will start the `mysql` container specified under the `services` section of the [docker-compose.yml](docker-compose.yml). The `-d` specifies that we want the container run in detached mode, returning us to our command prompt.

Next we will start the remaining containers defined in the docker-compose.yml (Ghost and Nginx). 

    $ docker-compose up -d

Without a service specified (e.g. mysql), docker-compose will start all services specified in the docker-compose.yml that are not already running. 

To verify that all of our services are running, we execute a `ps`:

    $ docker-compose ps

This should return output similar to this:

             Name                       Command             State                    Ports
    --------------------------------------------------------------------------------------------------------
    ridgelinedocker_ghost_1   /entrypoint.sh npm start      Up      0.0.0.0:2368->2368/tcp
    ridgelinedocker_mysql_1   docker-entrypoint.sh mysqld   Up      0.0.0.0:3333->3306/tcp
    ridgelinedocker_nginx_1   nginx -g daemon off;          Up      0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp

In the "Ports" section, the first port listed is the port exposed on our local machine and the respective port it is mapped to within the container. E.g. Port 3333 on our local machine maps to 3306 on the mysql container (this is done to avoid port conflicts with other services we may have running locally).

#### Accessing Ghost

Generally we would map a more obscure port to the Ghost process because 80 can be a commonly used port on our local machines, but for this example, we're just mapping 80 -> 80.

We can now hit http://localhost in a browser on our local machine and we should land on the Ghost welcome page.

#### Stopping the services

There are a couple options to actually stop the services. We can use the docker-compose command `stop` or `down`

    # Stop the services but keep the containers around to be started later
    $ docker-compose stop

    # Stop and destroy the containers. This will remove any data/config that is not specified as persistent in the project
    $ docker-compose down

*Note on data persistence: Shared volumes can be configured for data persistence even in the event of the containers being destroyed. This example does not use shared volumes, so any data not in the project repo will be lost when containers are destroyed. For example, any data added to MySQL after the service is started would be lost if the container is destroyed.*


### Additional Setup Details

#### Volumes

In this configuration, the ridgeline/ghost directory is mounted inside the Ghost container via these lines in the docker-compose.yml and contains the Ghost site files:

    volumes:
      - ./ghost:/var/lib/ghost

This means any files changed in our local ghost directory will be immediately updated within the ghost container (e.g. iterating on files during the development process). We should only need to refresh our browser to see the changes.

#### MySQL

The MySQL database name and credentials can be viewed in the docker-compose.yml. In this case, MySQL can be accessed from our local machine by connecting to port 3333 with these credentials:

    mysql:
      image: mysql
      environment:
        MYSQL_USER: ghost
        MYSQL_PASSWORD: test1234
        MYSQL_DATABASE: ghost
        MYSQL_ROOT_PASSWORD: test1234
      ports:
        - 3333:3306

## Config Scenarios

In a real-world scenario, we could add the Docker related config to any Ghost project and adjust the volume mapping as necessary to make the Ghost site files available within the container. We could also point our docker-compose.yml file at a different Ghost project to load files for another customer.

#### One Option

For example, assume we work with multiple customer projects within the ~/Projects directory (~/Projects/customer_1, ~/Projects/customer_2, etc). We might clone this ridgeline repo to ~/Projects/ridgeline so we can work with the various customer sites. In this case, we can change our volume mapping to mount the site files for customer_1:

     volumes:
      - ../customer_1:/var/lib/ghost

This scenario would run all of the services from the ~/Projects/ridgeline directory, but would load the site files for customer\_1 from ~/Projects/customer\_1. 

#### Another Option

**Note: This option can quickly get confusing and is not highly recommended**

We could have a separate docker-compose.yml for each customer within the ridgeline project. We might have a `docker-compose-customer_1.yml`, `docker-compose-customer_2.yml`, etc. Each of these config files would mount the respective ~/Projects/customer_x directory to the Ghost container (similar to the previous option). We would then start each customer stack by specifying the docker-compose file we wanted to use:

    $ docker-compose -f docker-compose-customer_1.yml up -d

#### Last and best option, in my opinion

In this scenario, as mentioned earlier, we could include the necessary Docker config to each customer project. Each project would have it's own Ghost stack and we would work with that stack from within the customer directory.

Example workflow would be:

    # Enter the customer directory
    $ cd ~/Projects/customer_1

    # Start the services
    $ docker-compose up -d mysql
    $ docker-compose up -d

The project specific `docker-compose.yml` files would be configured to mount the current customer project to the Ghost container. 

** Note: There are other docker config considerations that need to be made in this scenario. For example, each customer project would need to map a different local port to the port 80 on the Ghost service to avoid port conflicts:

    # Example mappings
    customer_1: 81 -> 80
    customer_2: 82 -> 80
    customer_3: 83 -> 80
    etc, etc, ...

The same thing applies for MySQL ports. There are other networking options that can be utilized to avoid this issue but are beyond the scope of this document. 


Created and maintned by David Thornton in 2017

