# jenkins-distributed-builds-gke
Create a continuous integration system using on demand Jenkins agents and Compute Engine
# Overview
1. Create a new project in Google Cloud
2. Create a service account and grant it the required priviledges
3. 

## Create a service account from the cloud console inside your new project and grant it the requisite priviledges
1. Use the gcloud command to create the service account
`gcloud iam service-accounts create jenkins --display-name jenkins`
2. Store your service account email address and Google Cloud Project ID in environment variables for future use
```
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:jenkins" --format='value(email)')
export PROJECT=$(gcloud info --format='value(config.project)')
```
3. Grant the following roles to the newly created service account: storage admin, compute instance admin, compute network admin, compute security admin, iam service account actor
```
gcloud projects add-iam-policy-binding $PROJECT \
    --role roles/storage.admin --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.instanceAdmin.v1 \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.networkAdmin \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.securityAdmin \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/iam.serviceAccountActor \
    --member serviceAccount:$SA_EMAIL
```
## Create the service account key to authenticate with the Compute Engine API in the future
1. Create the service account key and download it to your local machine for future use when configuring the JClouds plugin to authenticate with the Compute Engine API. The following command will generate a key file called `jenkins-sa.json`.
```
gcloud iam service-accounts keys create jenkins-sa.json --iam-account $SA_EMAIL
``` 
2. Click the button at the top of the cloud console that consists of three vertically stacked dots. Then click "Download File".
3. Click the folder icon then click the arrow next to the directory that pops up to browse a list of files. Find the `jenkins-sa.json` file and click it.
4. Click "download" to save the file to your local machine



