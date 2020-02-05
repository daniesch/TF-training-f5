# TF-training-f5


## Terraform for Infrastructure as Code

Terraform is an infrastructure as code (IaC) tool by HashiCorp that allows provisioning of a wide variety of infrastructure resources through a collection of plugins. IaC means we write configuration files that describe the infrastructure we want, and when we run Terraform, it compares it with the current state of the deployed resources and provisions or edits the necessary resources to match the wanted state.

Run the following commands to download and install Terraform.

On Linux:

```
wget https://releases.hashicorp.com/terraform/0.12.6/terraform_0.12.6_linux_amd64.zip
unzip terraform_0.12.6_linux_amd64.zip
sudo mv terraform /opt/terraform
sudo ln -s /opt/terraform /usr/local/bin/terraform
```

On Mac:

```
brew install terraform
```


## Set up your Google Cloud Platform working environment

Once we have our project, we can install and configure the Google Cloud SDK and the Kubernetes command line tool. The SDK provides the tools used to interact with the Google Cloud Platform REST API, they allow us to create and manage GCP resources from a command-line interface. Run the following commands to install and initialize it:

On Linux:

```
echo "deb https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-get update && sudo apt-get install google-cloud-sdk kubectl
gcloud init
```

On Mac:

```
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl
chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl
gcloud init
```


Google Cloud offers an advanced permissions management system with Cloud Identity and Access Management (Cloud IAM). Terraform needs to be authorized to communicate with the Google Cloud API to create and manage resources in our GCP project. We achieve this by enabling the corresponding APIs and creating a service account with appropriate roles.

First, enable the Google Cloud APIs we will be using:

```
gcloud services enable compute.googleapis.com
gcloud services enable servicenetworking.googleapis.com
gcloud services enable cloudresourcemanager.googleapis.com
gcloud services enable container.googleapis.com
```
Then create a service account:

```
gcloud iam service-accounts create <service_account_name>
```
Here service_account_name is the name of our service account, it cannot contain spaces or fancy characters, you can name account-yourname for example.

Now we can grant the necessary roles for our service account to create a GKE cluster and the associated resources:

```
gcloud projects add-iam-policy-binding f5-gcs-4261-sales-emea-dach --member serviceAccount:<service_account_name>@f5-gcs-4261-sales-emea-dach.iam.gserviceaccount.com --role roles/container.admin
gcloud projects add-iam-policy-binding f5-gcs-4261-sales-emea-dach --member serviceAccount:<service_account_name>@f5-gcs-4261-sales-emea-dach.iam.gserviceaccount.com --role roles/compute.admin
gcloud projects add-iam-policy-binding f5-gcs-4261-sales-emea-dach --member serviceAccount:<service_account_name>@f5-gcs-4261-sales-emea-dach.iam.gserviceaccount.com --role roles/iam.serviceAccountUser
gcloud projects add-iam-policy-binding f5-gcs-4261-sales-emea-dach --member serviceAccount:<service_account_name>@f5-gcs-4261-sales-emea-dach.iam.gserviceaccount.com --role roles/resourcemanager.projectIamAdmin
```

Finally, we can create and download a key file that Terraform will use to authenticate as the service account against the Google Cloud Platform API:

```
gcloud iam service-accounts keys create terraform-gke-keyfile.json --iam-account=<service_account_name>@f5-gcs-4261-sales-emea-dach.iam.gserviceaccount.com
```


##Terraform state in Google Cloud Storage

To work on your infrastructure with a team, we can use source control to share your infrastructure code. By default, Terraform stores the state of your infrastructure in a local state file. We could commit it with your infrastructure code, but the best practice for sharing a Terraform state when working with teams is to store it in remote storage. In our case, we will configure Terraform to store the state in a Google Cloud Storage Bucket.

First, let’s create a bucket, we could do it graphically on the Google Cloud Console, or we can use the Google Cloud SDK we just installed:

```
gsutil mb -p f5-gcs-4261-sales-emea-dach -c regional -l <location> gs://<bucket_name>/
```

###Availabal GCP Regions
northamerica-northeast1 (Montréal)
us-central (Iowa)
us-west2 (Los Angeles)
us-east1 (South Carolina)
us-east4 (Northern Virginia)
southamerica-east1 (São Paulo)
europe-west (Belgium)
europe-west2 (London)
europe-west3 (Frankfurt)
europe-west6 (Zürich)
asia-northeast1 (Tokyo)
asia-northeast2 (Osaka)
asia-east2 (Hong Kong)
asia-south1 (Mumbai)
australia-southeast1 (Sydney)


For location, you can choose europe-west4 or anything else you like.
For bucket_name, choose a meaningful globally unique name. If the name you chose is unavailable, try again with a different name.


Once we have our bucket, we can activate object versioning to allow for state recovery in the case of accidental deletions and human error:

```
gsutil versioning set on gs://<bucket_name>/
```

Finally, we can grant read/write permissions on this bucket to our service account:

```
gsutil iam ch serviceAccount:<service_account_name>@f5-gcs-4261-sales-emea-dach.iam.gserviceaccount.com:legacyBucketWriter gs://<bucket_name>/
```

We can now configure Terraform to use this bucket to store the state. Create the following terraform.ft file in the same directory where you downloaded the service account key file. Make sure to replace the bucket name with yours.

```
terraform {
  backend "gcs" {
    credentials = "./terraform-gke-keyfile.json"
    bucket      = "<bucket_name>"
    prefix      = "terraform/state"
  }
}
```

We can now run terraform init and the output will display that the Google Cloud Storage backend is properly configured.


###Create the GKE cluster

We should now have a terraform.tf file and the service account key-file in our working directory. Here is our target directory structure:

```
.
├── .terraform
│   ├── modules
│   │   ├── gke
│   │   └── modules.json
│   └── plugins
│       └── ...
├── main.tf
├── providers.tf
├── terraform-gke-keyfile.json
├── terraform.tf
├── variables.tf
└── variables.auto.tfvars

```
