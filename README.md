# Steps from deploying codbex-hades to Snowflake's Snowpark

## Snowflake setup

1. Create a non-trial Snowflake account
2. In a worksheet execute the following commands:

    1. New role, privileges, warehouse and DB:
    ```sql
    // Create an CONTAINER_USER_ROLE with required privileges
    USE ROLE ACCOUNTADMIN;
    CREATE ROLE CONTAINER_USER_ROLE;
    GRANT CREATE DATABASE ON ACCOUNT TO ROLE CONTAINER_USER_ROLE;
    GRANT CREATE WAREHOUSE ON ACCOUNT TO ROLE CONTAINER_USER_ROLE;
    GRANT CREATE COMPUTE POOL ON ACCOUNT TO ROLE CONTAINER_USER_ROLE;
    GRANT CREATE INTEGRATION ON ACCOUNT TO ROLE CONTAINER_USER_ROLE;
    GRANT MONITOR USAGE ON ACCOUNT TO  ROLE  CONTAINER_USER_ROLE;
    GRANT BIND SERVICE ENDPOINT ON ACCOUNT TO ROLE CONTAINER_USER_ROLE;
    GRANT IMPORTED PRIVILEGES ON DATABASE snowflake TO ROLE CONTAINER_USER_ROLE;

    // Grant CONTAINER_USER_ROLE to ACCOUNTADMIN
    grant role CONTAINER_USER_ROLE to role ACCOUNTADMIN;

    // Create Database, Warehouse, and Image spec stage
    USE ROLE CONTAINER_USER_ROLE;
    CREATE OR REPLACE DATABASE CONTAINER_HOL_DB;

    CREATE OR REPLACE WAREHOUSE CONTAINER_HOL_WH
    WAREHOUSE_SIZE = XSMALL
    AUTO_SUSPEND = 120
    AUTO_RESUME = TRUE;
    
    CREATE STAGE IF NOT EXISTS specs
    ENCRYPTION = (TYPE='SNOWFLAKE_SSE');

    CREATE STAGE IF NOT EXISTS volumes
    ENCRYPTION = (TYPE='SNOWFLAKE_SSE')
    DIRECTORY = (ENABLE = TRUE);
    ```

    2. Compute pool and image repository:
    ```sql
    USE ROLE ACCOUNTADMIN;
    CREATE SECURITY INTEGRATION IF NOT EXISTS snowservices_ingress_oauth
    TYPE=oauth
    OAUTH_CLIENT=snowservices_ingress
    ENABLED=true;

    USE ROLE CONTAINER_USER_ROLE;
    CREATE COMPUTE POOL IF NOT EXISTS CONTAINER_HOL_POOL
    MIN_NODES = 1
    MAX_NODES = 1
    INSTANCE_FAMILY = CPU_X64_XS;

    CREATE IMAGE REPOSITORY CONTAINER_HOL_DB.PUBLIC.IMAGE_REPO;

    SHOW IMAGE REPOSITORIES IN SCHEMA CONTAINER_HOL_DB.PUBLIC;
    ```
3. Setup local environment

    1. Download and install the miniconda installer from https://conda.io/miniconda.html.
    2. Create a file `conda_env.yaml`
        ```yaml
        name: snowpark-container-services-hol
        channels:
            - https://repo.anaconda.com/pkgs/snowflake
        dependencies:
            - python=3.10
            - snowflake-snowpark-python[pandas]
            - ipykernel
        ```
    3. Create the conda environment.
        ```
        conda env create -f conda_env.yml
        ```
    4. Activate the conda environment.
        ```
        conda activate snowpark-container-services-hol
        ```
    5. Install hatch so we can build the SnowCLI:
        ```
        pip install hatch
        ```
    6. Install SnowCLI:
        ```bash
        # naviage to where you want to download the snowcli GitHub repo, e.g. ~/Downloads
        cd /your/preferred/path
        # clone the git repo
        git clone https://github.com/Snowflake-Labs/snowcli
        # cd into the snowcli repo
        cd snowcli
        # install
        hatch build && pip install .
        # during install you may observe some dependency errors, which should be okay for the time being 
        ```
    7. Configure your Snowflake CLI connection by following the steps
        ```bash
        snow connection add
        ```

        test the connection
        ```bash
        # test the connection:
        snow connection test --connection "CONTAINER_hol"
        ```
4. Image preperation
    1. Pull latest codbex-hades image
        ```bash
        docker pull ghcr.io/codbex/codbex-hades:latest
        ```
        **_NOTE_**: If you are on ARM architecture `export DOCKER_DEFAULT_PLATFORM=linux/amd64`
    3. Login in your Snowflake image repository
        ```bash
        # snowflake_registry_hostname = org-account.registry.snowflakecomputing.com
        docker login <snowflake_registry_hostname> -u <user_name>
        ```
    4. Tag and push
        ```bash
        # repository_url = org-account.registry.snowflakecomputing.com/CONTAINER_hol_db/public/image_repo
        docker tag codbex-hades:latest <repository_url>/codbex-hades:dev
        docker push <repository_url>/codbex-hades:dev
        ```
5. Create service

    1. Create spec file for service deployment

        codbex-hades-snowpark.yaml
        ```yaml
        spec:
        containers:
            - name: codbex-hades-snowpark
            image: <repository_hostname>/container_hol_db/public/image_repo/codbex-hades:dev
            volumeMounts:
                - name: codbex-hades-home
                mountPath: /home/codbex-hades
        endpoints:
            - name: codbex-hades-snowpark
            port: 80
            public: true
        volumes:
            - name: codbex-hades-home
            source: "CONTAINER_HOL_DB.PUBLIC.VOLUMES"
            uid: 1000
            gid: 1000
        networkPolicyConfig:
            allowInternetEgress: true
        ```
    2. Deploy the spec file:
       ```bash
       snow object stage copy ./src/codbex-hades-snowpark/codbex-hades.yaml  @specs --overwrite --connection CONTAINER_hol
       ```
    3. In the Snowflake worksheet execute the following command:
        ```sql
        CREATE SERVICE codbex_hades
        in compute pool CONTAINER_HOL_POOL
        from @specs
        spec = 'codbex-hades-snowpark.yaml';
        ```
    4. Check your service:
        ```sql
        CALL SYSTEM$GET_SERVICE_STATUS('CONTAINER_HOL_DB.PUBLIC.CODBEX_HADES');
        CALL SYSTEM$GET_SERVICE_LOGS('CONTAINER_HOL_DB.PUBLIC.CODBEX_HADES', '0', 'codbex-hades', 10);
        CALL SYSTEM$REGISTRY_LIST_IMAGES('/CONTAINER_HOL_DB/PUBLIC/IMAGE_REPO');

        SHOW SERVICES like 'codbex_hades';
        ```
    5. Get service endpoint
        ```sql
        SHOW ENDPOINTS IN SERVICE codbex_hades;
        ```
6. Clean-up
    ```sql
    USE ROLE CONTAINER_USER_ROLE;
    ALTER COMPUTE POOL CONTAINER_HOL_POOL STOP ALL;
    ALTER COMPUTE POOL CONTAINER_HOL_POOL SUSPEND;

    DROP SERVICE CONTAINER_HOL_DB.PUBLIC.CODBEX_HADES;
    DROP COMPUTE POOL CONTAINER_HOL_POOL;

    DROP DATABASE CONTAINER_HOL_DB;
    DROP WAREHOUSE CONTAINER_HOL_WH;

    USE ROLE ACCOUNTADMIN;
    DROP ROLE CONTAINER_USER_ROLE;
    ```

### Relates sources and references
1. Intro into snowpark tutorial - https://quickstarts.snowflake.com/guide/intro_to_snowpark_container_services/index.html#0
2. Snowpark documentation - https://docs.snowflake.com/en/developer-guide/snowpark-container-services/overview
3. Hades - https://www.codbex.com/products/hades/
