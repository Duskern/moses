# moses

The Military Open Simulator Enterprise Strategy (moses) is a method and strategy for deploying a set of OpenSimulator services and simulators in order to run a private grid.  This grid, once built, can reside in an enclave, offline, or be hosted online with a single entry point.

This composition can be run on a docker host, and any region simulators run on additional machines.  The necessary ports for region connectivity are exposed in this composition, but region traffic is not proxied.  The region UDP and TCP ports must be handled outside of docker.

This should mostly work, but at the time of this writing no integration testing has been performed.

# features

This deployment strategy is a docker composition, allowing for a quick set-up and use of an otherwise difficult to configure system.  The system is being separated into replaceable microservices where possible, and sane defaults have been selected where possible.

## Halcyon

This simulator system is based on the Halcyon fork of OpenSimulator.  The performance/feature trade-off between this fork and upstream OpenSimulator was a simple decision for what the MOSES team needs.  Halcyon runs multiple services in order to be a performant grid, they include:  GridServer.exe, UserServer.exe, MessagingServer.exe, Aperture, Whip, and MySQL.  These are each spun up as a separate container.  In the future we hope to extend this to allow for grid-service clustering and scaling, but it is currently single-instance only.

The Halcyon.exe simulator processes are currently windows-only.  It would be much easier to include the simulators and simplify the networking model drastically, but they must be run on windows machines.

## Grid Management

This strategy includes the MOSES Grid Manager (MGM).  This web application includes an html5 single-page application for grid and user management, and several scripts for user templating, etc.  This application, in concert with mgmNode when deployed, configured ans migrates regions, performs performance monitoring, aggregates and serves region logs, and allows administrative actions from within the web application.

## Freeswitch

This strategy includes both a freeswitch server for voice connectivity, but a management service allowing halcyon voice chat over freeswitch using unmodified OpenSimulator capable clients.

## Offline Messaging

This strategy includes a serviec for storing offline messages for the Halcyon grid.

## Map Service

Coming soon... an improved map service using Anaximander2.

# Build and Deploy

As this composition is composed of so many different pieces, we selected docker also as the build environment.  Contained in these files are scripts that create docker containers to hold the development requirements for the services, as well as slimmed produciton containers for deployment.  While each of these services can be developed and run natively, network and requirements management would be a steep requirement.

Configuration is not as simple as it could be, as the mgmNode and Halcyon.exe components must reside outside of the composition on a Windows host for now.

1. Install the latest docker-ce for your platform using directions from docker.com

1. Install the latest stable docker-compose for your platform using directions from https://docs.docker.com/compose/install/

1. Checkout this repository where you would like to host this composition

1. Perform the following configuration steps:

    1.  SSL certificates.  SSL certs are used in multiple places accross this stack.  you can use different certs for the webserver and for the mgmt portion, but the certs MUST MATCH between the mgmt portion and the Halcyon.exe services.  Put a certificate pair in mgmt/certs/ named cert.pem and key.pem.  Put a certificate pair under webserver/certs (CA issued for convenience) named site.crt and site.key.
  
    1.  environment variables.  Please update the following in docker-compose.yml
  
        * update mysql credentials as desired, but make sure all occurences match
        * FREESWITCH_API: use the LAN address of your docker host
        * OFFLINE_MESSAGES_API: use the LAN address of your docker host
        * PUBLIC_IP: use the client-addressible address for your new grid
        * LAN_IP: use the LAN address of your docker host
        * TEMPLATES: insert the template map here once templates are generated
        * MAIL: email config for your grid
        * GRID_SERVER: use the LAN ip for your docker host
        * USER_SERVER: use the STATIC ip for your grid
        * MESSAGING_SERVER: use the LAN ip for your host
        * WHIP_SERVER: use your updated password and the LAN ip for your grid
        * GRID_NAME: name your grid
        * GRID_NICK: nickname for your grid
        * LOGIN_URI: use the STATIC ip for your grid
        * FREESWITCH_IP: use the STATIC ip for your grid
        
    1. Service Configurations:
    
        * aperture/aperture.config, update the whip password and to match your earlier configs.  Randomize caps token if desired.
        * gridserver/GridServer_Config.xml, update the database credentials if needed, and set the default_user_server using your STATIC ip address.
        * messagingserver/MessagingServer_Config.xml, updating the database credentials if needed, and updating the published_ip to your STATIC ip.
        * userserver/Halcyon.ini, change the database credentials if needed under [Startup] and [Inventory], and update the GridInfo section if desired.
        * userserver/UserServer_Config.xml, change the database credentials if needed, and the welcome message.
        * whip/whip.cfg, update the password if you changed it in the other configs
        
1.  Build the containers by issuing `docker-compose up -d`

    Wait for it.  This composition does not use distributed images, but compiles the components from source.  This will take a while the first time.  The Halcyon services will crash if their database is not initialized yet.  This is expected.

1.  Initialize your databases by either

    * execute a mysqldump restore from a compatible database by executing `cat dump.sql | docker exec -it moses_halcyon-mysql_1 mysql -uroot -phal hal` using your updated credentials from docker-compose.yml; whith a similar command for mgmt-mysql
    * initialize new databases for a new grid by executing: `docker exec moses_gridserver_1 hc-database.exe --init -t core -u hal -s hal -p hal` for halcyon, and `docker exec moses_mgmt_1 node scripts/migrate-db.js`
    
1.  MGM Initialization.  You can create any avatar accounts necessary by running `docker exec moses_mgmt_1 node scripts/create-user.js fname lname email password <userlevel>`.  Create at least one administrative account for yourself (access level 250), and two template accounts for Male and Female avatar templating.  Insert their UUIDs into the docker config for TEMPLATES in the form { M: 'xxxx', F: 'xxx } to enable the default templates.  Once the environment is set for MGM to use the templates, you must restart the container.

1.  Access your MGM instance via port 443 and login using your administrator account.  you can create and add Hosts, which are windows machines running a configured instance of mgmNode, and which will contain your regions.

