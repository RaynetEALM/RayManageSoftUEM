# RayManageSoft Unified Endpoint Manager

RayManageSoft Unified Endpoint Manager provides a cloud-based solution for software deployment as well as patch management and an overview over the entire IT infrastructure.
It is possible to automate the management and software deployment for huge parts of the infrastructure while at the same time, keep individual schedules and lists of optional and mandatory software for specific endpoints.

![Screenshot](https://github.com/RaynetEALM/RayManageSoftUEM/blob/main/uem.png?raw=true)

## Multi-tenancy

RayManageSoft Unified Endpoint Manager can manage a multi-tenant environment, which means that it can manage multiple tenants or multiple infrastructure environments which can be hosted by different storage hosters.

## Supported storage providers

Currently Microsoft Azure, Amazon S3, and MinIO are the storage hoster which are supported by RayManageSoft Unified Endpoint Manager.

## Installation

### Prerequisites

* Docker Images for RayManageSoft Unified Endpoint Manager.
  
* Docker for Linux (on-premise installation)
  
* Microsoft SQL Server.
*An instance of MS SQL Server or SQL Server Express must be available and the server must be reachable from the Docker environment.*

* A cloud storage solution (Azure, MinIO, or Amazon Web Services)
  
* A valid RayManageSoft Unified Endpoint Manager license, either in form of an order number or in form of a license file.

**Note:** RayManageSoft Unified Endpoint Manager uses linux docker images. Make sure that Docker has been switched to Linux Containers mode. It is not possible to pull the images when running Windows Containers.

### Images

RayManageSoft Unified Endpoint Manager images are available on docker hub:

* [`https://hub.docker.com/r/raynetgmbh/raymanagesoft-uem-backend`](https://hub.docker.com/r/raynetgmbh/raymanagesoft-uem-backend)
* [`https://hub.docker.com/r/raynetgmbh/raymanagesoft-uem-frontend`](https://hub.docker.com/r/raynetgmbh/raymanagesoft-uem-frontend)

You can use tags `3.0` or `stable` to get the last 3.0 or the last stable version respectively. You will find the complete list of tags on the respective Docker Hub page.

### Environment variables

Assuming the default setup with MinIO as a storage provider, the following environment variables are used to set-up the system:

* **ConnectionStrings__System** - Connection string to the system database
  
* **ConnectionStrings__ResultDatabase** - Connection string to the database storing the tenant-specific results.
  
* **BackendConfig__Endpoint**, **BackendConfig__Port**, **BackendConfig__Protocol** - together they form the URL, under which your instance is available. The default settings are fine for a quick setup, but not production ready.
  
* **BackendConfig__Authentication** - Basic authentication for the managed device to the backend server communication can be deactivated. This is recommended during migration.
  
* **StorageConfig__Default** - The default hoster for the storage of package files.
  
* **StorageConfig__MinIO__Endpoint** - your MinIO Endpoint (for example ``play.min.io:80``)
  
* **StorageConfig__MinIO__AccessKey** - your access key for MinIO (configured during the MinIO setup)
  
* **StorageConfig__MinIO__SecretKey** - your MinIO secret key (configured during the MinIO setup)
  
* **StorageConfig__MinIO__SSL** - a boolean value indictating whether the MinIO server requires/uses a https connection or not (the usage of an https connection is recommended).

Different environment variables are required for other supported set-up types (see installation guide for more information).

### Setting up with docker compose

1. Set-up a new MS SQL Server or use an existing MS SQL Server which is accessible from the hosting environment.

2. Copy connection string to your SQL Server. Ensure that the connection string is valid.

3. Install container images with ``docker compose`` with the compose file [`docker-compose.yml`](https://github.com/RaynetEALM/RayManageSoftUEM/blob/main/docker-compose.yml?raw=true) and environment file [`env.list`](https://github.com/RaynetEALM/RayManageSoftUEM/blob/main/env.list?raw=true) from this repository. The compose file by default uses the configuration with MinIO for the storage. Refer to installation guide for more information about other supported setups (Azure / Amazon S3). Before creating the containers, make sure to adjust the connection string in the compose file.

### Default docker compose file

    version: "3.7"
     services:
    
     database:
      image: mcr.microsoft.com/mssql/server:2019-latest
      volumes:
        - sql_data:/var/opt/mssql
      ports:
        - "1433:1433"
      environment: 
        - ACCEPT_EULA=Y
        - SA_PASSWORD=<myPassword>
        - MSSQL_PID=Express
      restart: always
      networks:
        - app-tier
    
     minio:
      image: minio/minio:RELEASE.2021-01-16T02-19-44Z
      volumes:
        - ./storage:/container/vol
      ports:
       - "9000:9000"
       - "9001:9001"
      restart: unless-stopped
      networks:
        - app-tier
      environment:
        MINIO_ROOT_USER: minio
        MINIO_ROOT_PASSWORD: Start123
      command: server /container/vol
      healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
        interval: 30s
        timeout: 20s
        retries: 3
      logging:
        driver: json-file
        options:
       max-size: "10m"
       max-file: "5"
    
     frontend:
        image: raynetgmbh/raymanagesoft-uem-frontend:stable
        hostname: "rmsc_frontend"
        depends_on:
        - database
        - minio
        ports:
       - "80:80"
        restart: always
        networks:
       - app-tier
        env_file:
        - env.list
    
     backend:
        image: raynetgmbh/raymanagesoft-uem-backend:stable
        hostname: "rmsc_backend"
        depends_on:
        - frontend
        ports:
       - "8080:80"
        restart: always
        networks:
       - app-tier
        env_file:
        - env.list
        
     networks:
      app-tier:
       driver: bridge
    
     volumes: 
       sql_data:
       storage:

### Default docker environment file

     ConnectionStrings__System="Server=myServerAddress;Database=myDatabase;User Id=myUsername;Password=myPassword;"
     ConnectionStrings__ResultDatabase="Server=myServerAddress;Database=myDatabase;User Id=myUsername;Password=myPassword;"
     
     BackendConfig__Endpoint="myServerAddress"
     BackendConfig__Port="8080"
     BackendConfig__Protocol="http"
     BackendConfig__Authentication="true"
     
     StorageConfig__Default="MinIO"
     StorageConfig__MinIO__Endpoint="myServerAddress:9000"
     StorageConfig__MinIO__AccessKey="minio"
     StorageConfig__MinIO__SecretKey="Start123"
     StorageConfig__MinIO__SSL="false"
     
     Integration__PackageStore__Endpoint="http://packages.packagestore.com/RayPackageService"
     Integration__PackageStore__Url="https://packaging.packagestore.com"
     Integration__PackageStore__ParallelProcessing=5
     
     Integration__Catalog__Url="https://rayventorycatalog.raynet.de"
     Integration__Catalog__ConfidenceRange = 90
     Integration__Catalog__Timeout = 200000
     Integration__Catalog__MaxAttempts = 3
     Integration__Catalog__DevicesBatchSize = 10
     Integration__Catalog__VulnerabilityBatchSize = 100
     Integration__Catalog__FingerprintBatchSize = 100
     Integration__Catalog__TotalNumberOfDevices = 0
     Integration__Catalog__UpdateExistingRecords="false"
     
     Integration__Azure__AzureApiUrl="https://graph.microsoft.com/"
     Integration__Azure__AzureInstance="https://login.microsoftonline.com/{0}"
     
     DbMaintenanceConfig__UpdateInstallStatesJob="0 0 0/4 1/1 * ?"
     DbMaintenanceConfig__FileStorageCleanupJob="0 0 3 1/1 * ?"
     DbMaintenanceConfig__SystemLogCleanupJob="0 0 4 1/1 * ?"
     DbMaintenanceConfig__ActivityLogCleanupJob="0 0 5 1/1 * ?"
     DbMaintenanceConfig__DeviceInventoryCleanupJob="0 0 6 1/1 * ?"

## First start

The initial login information to the system are:

* **E-mail**: ``root@raynet.de``
* **Password**: ``raynet``

After the first login please visit the `Site administration` > `System settings` page. There are a few important checks to be done:

* Ensure that the backend URL, port and protocol defined in the settings page are valid and match the parameters of the backend container. When a local installation is used, the FQDN of the backend will most likely be the same as the web UI, with the only difference in port numbers. Should there be any mismatch, make sure to adjust the values as required.
* Change the initial password of the root user to something secure, using long sequence of letters, numbers and special characters.
* Download Managed Device Client from the Devices page and install it on the computers to manage. Once the agent is started, the device will appear in the Devices tab.

### License Activation

RayManageSoft Unified Endpoint Manager needs a valid license to run. If there is no valid license, RayManageSoft Unified Endpoint Manager will open the activation screen.

The product can be activated using one of the following methods.

* By supplying the order number.
* By supplying an already created license file (.rswl).
* By supplying a license string.

## Documentation

* [Release notes (PDF)](https://github.com/RaynetEALM/RayManageSoftUEM/blob/main/docs/RayManageSoft_Unified_Endpoint_Manager_3.0_Release_Notes.pdf)
* [Installation Guide (PDF)](https://github.com/RaynetEALM/RayManageSoftUEM/blob/main/docs/RayManageSoft_Unified_Endpoint_Manager_3.0_Installation_Guide.pdf)
* [User Guide (PDF)](https://github.com/RaynetEALM/RayManageSoftUEM/blob/main/docs/RayManageSoft_Unified_Endpoint_Manager_3.0_User_Guide.pdf)
* [Operations Supplement (PDF)](https://github.com/RaynetEALM/RayManageSoftUEM/blob/main/docs/RayManageSoft_Unified_Endpoint_Manager_3.0_Operations_Supplement.pdf)

## More information

* [Raynet GmbH corporate website](https://raynet.de)
* [Raynet EALM GitHub](https://github.com/raynetEALM)
