# Configure backup storage

Configure backup storage for your [backup repositories](backups.md#backup-repositories) in the
`backups.pgbackrest.repos` section of the `deploy/cr.yaml` configuration file.

=== "S3-compatible backup storage"

     To use S3-compatible storage for backups, you need to have the following S3-related information:

     * The name of S3 bucket;
     * The endpoint - the URL to access the bucket
     * The region - the location of the bucket
     * S3 credentials such as S3 key and secret to access the storage. These are stored in an encoded form in [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) along with other sensitive information. 

     **Configuration steps**

     1. Encode the S3 credentials and the pgBackRest repo name that you will use for backups. In this example, we use AWS S3 key and S3 key secret and `repo2`. 

        === "in Linux"     

             ``` {.bash data-prompt="$" }
             $ cat <<EOF | base64 --wrap=0
             [global]
             repo2-s3-key=<YOUR_AWS_S3_KEY>
             repo2-s3-key-secret=<YOUR_AWS_S3_KEY_SECRET>
             EOF
             ```     

        === "in macOS"     

             ``` {.bash data-prompt="$" }
             $ cat <<EOF | base64
             [global]
             repo2-s3-key=<YOUR_AWS_S3_KEY>
             repo2-s3-key-secret=<YOUR_AWS_S3_KEY_SECRET>
             EOF
             ```     

     2. Create the Secret configuration file and specify the base64-encoded string from the previous step. The following is the example of the  `cluster1-pgbackrest-secrets.yaml` Secret file:     

         ```yaml
         apiVersion: v1
         kind: Secret
         metadata:
           name: cluster1-pgbackrest-secrets
         type: Opaque
         data:
           s3.conf: <base64-encoded-configuration-contents>
         ```     

        !!! note     

             This Secret can store credentials for several repositories presented as
             separate data keys.     

     3. Create the Secrets object from this yaml file:     

         ``` {.bash data-prompt="$" }
         $ kubectl apply -f cluster1-pgbackrest-secrets.yaml
         ```     

     4. Update your `deploy/cr.yaml` configuration. Specify the Secret file you created in the `backups.pgbackrest.configuration` subsection, and put all other S3 related information in the `backups.pgbackrest.repos` subsection under the repository name that you intend to use for backups. This name must match the name you used when you encoded S3 credentials on step 1.

     For example, the S3 storage for the `repo1` repository looks as follows:     

     ```yaml
     ...
     backups:
       pgbackrest:
         ...
         configuration:
           - secret:
               name: cluster1-pgbackrest-secrets
         ...
         repos:
         - name: repo2
           s3:
             bucket: "<YOUR_AWS_S3_BUCKET_NAME>"
             endpoint: "<YOUR_AWS_S3_ENDPOINT>"
             region: "<YOUR_AWS_S3_REGION>"
     ```     

     5. Create or update the cluster:     

         ``` {.bash data-prompt="$" }
         $ kubectl apply -f deploy/cr.yaml
         ```

=== "Google Cloud Storage"

    To use [Google Cloud Storage](https://cloud.google.com/storage) as
    an object store for backups, you need the following information:    

    * a proper GCS bucket name. Pass the bucket name to `pgBackRest` via the
    `gcs.bucket` key in the `backups.pgbackrest.repos` subsection of
    `deploy/cr.yaml`.
    * your service account key for the Operator to access the storage.    
    
    **Configuration steps** 

    1. Create your service account key following the [official Google Cloud instructions](https://cloud.google.com/iam/docs/creating-managing-service-account-keys).
    2. Export this key from your Google Cloud account.    

        You can find your key in the Google Cloud console (select *IAM & Admin*
        → *Service Accounts* in the left menu panel, then click your account and
        open the *KEYS* tab):    

        ![image](assets/images/gcs-service-account.svg)    

        Click the *ADD KEY* button, chose *Create new key* and chose *JSON* as a key
        type. These actions will result in downloading a file in JSON format with
        your new private key and related information (for example, `gcs-key.json`).    

    3. Create the [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/). The Secret consists of 
         base64-encoded versions of two files: the `gcs-key.json` file with the Google service account key you have just downloaded, and the special `gcs.conf` configuration file.    

        * Create the `gcs.conf` configuration file. The file contents depends on the repository
        name for backups in the `deploy/cr.yaml` file. In case of the `repo3` repository, it looks as follows:    

        ```
        [global]
        repo3-gcs-key=/etc/pgbackrest/conf.d/gcs-key.json
        ```    

        * Encode both `gcs-key.json` and `gcs.conf` files. 

            === "Linux"

                ```
                base64 --wrap=0 <filename>
                ```

            === "MacOS"
                
                ```
                base64 <filename>
                ```

        * Create the Kubernetes Secret configuration file and specify your cluster name
        and the base64-encoded contents of the files from previous steps. The following is the example of the
        `cluster1-pgbackrest-secrets.yaml` Secret file:    

           ```yaml
           apiVersion: v1
           kind: Secret
           metadata:
             name: cluster1-pgbackrest-secrets
           type: Opaque
           data:
             gcs-key.json: <base64-encoded-json-file-contents>
             gcs.conf: <base64-encoded-conf-file-contents>
           ```       

           !!! note       

               This Secret can store credentials for several repositories presented as
               separate data keys.    

    4. Create the Secrets object from the Secret configuration file:    

        ``` {.bash data-prompt="$" }
        $ kubectl apply -f cluster1-pgbackrest-secrets.yaml
        ```    

    5. Update your `deploy/cr.yaml` configuration. Specify your GCS credentials
        Secret in the `backups.pgbackrest.configuration` subsection, and put GCS
        bucket name into the `bucket` option 
        in the `backups.pgbackrest.repos` subsection. The repository name must be the same as the name you specified when you created the `gcs.conf` file. For example, GCS storage configuration
        for the `repo3` repository would look as follows.    

        ```yaml
        ...
        backups:
          pgbackrest:
            ...
            configuration:
              - secret:
                  name: cluster1-pgbackrest-secrets
            ...
            repos:
            - name: repo3
              gcs:
                bucket: "<YOUR_GCS_BUCKET_NAME>"
        ```    

    6. Create or update the cluster:    

        ``` {.bash data-prompt="$" }
        $ kubectl apply -f deploy/cr.yaml
        ```

=== "Azure Blob Storage (tech preview)"

    To use [Microsoft Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/) for storing backups, you need the following:

    * a proper Azure container name. 
    * Azure Storage credentials. These are stored in an encoded form in the [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/).

    **Configuration steps**

    1. Encode the Azure Storage credentials and the pgBackRest repo name that you will use for backups with base64. In this example, we are using `repo4`.   

        === "Linux"    

            ``` {.bash data-prompt="$" }
            $ cat <<EOF | base64 --wrap=0
            [global]
            repo4-azure-account=<AZURE_STORAGE_ACCOUNT_NAME>
            repo4-azure-key=<AZURE_STORAGE_ACCOUNT_KEY>
            EOF
            ```    

        === "macOS"    

            ``` {.bash data-prompt="$" }
            $ cat <<EOF | base64
            [global]
            repo4-azure-account=<AZURE_STORAGE_ACCOUNT_NAME>
            repo4-azure-key=<AZURE_STORAGE_ACCOUNT_KEY>
            EOF
            ```    

    2. Create the Secret configuration file and specify the base64-encoded string from the previous step. The following is the example of the  `cluster1-pgbackrest-secrets.yaml` Secret file:    

        ```yaml
        apiVersion: v1
        kind: Secret
        metadata:
          name: cluster1-pgbackrest-secrets
        type: Opaque
        data:
          azure.conf: <base64-encoded-configuration-contents>
        ```    

        !!! note    

            This Secret can store credentials for several repositories presented as
            separate data keys.    

    3. Create the Secrets object from this yaml file:    

        ``` {.bash data-prompt="$" }
        $ kubectl apply -f cluster1-pgbackrest-secrets.yaml
        ```    

    4. Update your deploy/cr.yaml configuration. Specify the Secret file you have created in the previous step in the `backups.pgbackrest.configuration` subsection. Put Azure
        container name in the `backups.pgbackrest.repos` subsection under the repository name that you intend to use for backups. This name must match the name you used when you encoded S3 credentials on step 1. 

        For example, the Azure storage
        for the `repo1` repository looks as follows.    

        ```yaml
        ...
        backups:
          pgbackrest:
            ...
            configuration:
              - secret:
                  name: cluster1-pgbackrest-secrets
            ...
            repos:
            - name: repo4
              azure:
                container: "<YOUR_AZURE_CONTAINER>"
        ```    

    5. Create or update the cluster:    

        ``` {.bash data-prompt="$" }
        $ kubectl apply -f deploy/cr.yaml
        ```

## Next steps

* [Make an on-demand backup](backups-ondemand.md)
* [Make a scheduled backup](backups-schedule.md)