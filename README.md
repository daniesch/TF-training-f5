# TF-training-f5


## Terraform for Infrastructure as Code

Terraform is an infrastructure as code (IaC) tool by HashiCorp that allows provisioning of a wide variety of infrastructure resources through a collection of plugins. IaC means we write configuration files that describe the infrastructure we want, and when we run Terraform, it compares it with the current state of the deployed resources and provisions or edits the necessary resources to match the wanted state.

Run the following commands to download and install Terraform.

On Linux:

```
wget https://releases.hashicorp.com/terraform/0.12.20/terraform_0.12.20_linux_amd64.zip
unzip terraform_0.12.20_linux_amd64.zip
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
gcloud projects add-iam-policy-binding <project_name> --member serviceAccount:<service_account_name>@<project_name>.iam.gserviceaccount.com --role roles/container.admin
gcloud projects add-iam-policy-binding <project_name> --member serviceAccount:<service_account_name>@<project_name>.iam.gserviceaccount.com --role roles/compute.admin
gcloud projects add-iam-policy-binding <project_name> --member serviceAccount:<service_account_name>@<project_name>.iam.gserviceaccount.com --role roles/iam.serviceAccountUser
gcloud projects add-iam-policy-binding <project_name> --member serviceAccount:<service_account_name>@<project_name>.iam.gserviceaccount.com --role roles/resourcemanager.projectIamAdmin
```

Finally, we can create and download a key file that Terraform will use to authenticate as the service account against the Google Cloud Platform API:

```
gcloud iam service-accounts keys create terraform-gke-keyfile.json --iam-account=<service_account_name>@<project_name>.iam.gserviceaccount.com
```


## Terraform state in Google Cloud Storage

To work on your infrastructure with a team, we can use source control to share your infrastructure code. By default, Terraform stores the state of your infrastructure in a local state file. We could commit it with your infrastructure code, but the best practice for sharing a Terraform state when working with teams is to store it in remote storage. In our case, we will configure Terraform to store the state in a Google Cloud Storage Bucket.

First, let’s create a bucket, we could do it graphically on the Google Cloud Console, or we can use the Google Cloud SDK we just installed:

```
gsutil mb -p <project_name> -c regional -l <location> gs://<bucket_name>/
```

## Availabal GCP Regions
```
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
```

For location, you can choose europe-west4 or anything else you like.
For bucket_name, choose a meaningful globally unique name. If the name you chose is unavailable, try again with a different name.


Once we have our bucket, we can activate object versioning to allow for state recovery in the case of accidental deletions and human error:

```
gsutil versioning set on gs://<bucket_name>/
```

Finally, we can grant read/write permissions on this bucket to our service account:

```
gsutil iam ch serviceAccount:<service_account_name>@<project_name>.iam.gserviceaccount.com:legacyBucketWriter gs://<bucket_name>/
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


### Create the GKE cluster

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

We will create each of those files and learn their purpose.

The .terraform/ directory is created and managed by Terraform, this is where it stores the external modules and plugins we will reference. To create the GKE cluster with Terraform, we will use the Google Terraform provider and a GKE community module. A module is a package of Terraform code that combines different resources to create something more complex. In our case, we will use a single module that will create for us many various resources such as a Google Container cluster, node pools, and a cluster service account.

To configure terraform to communicate with the Google Cloud API and to create GCP resources, create a providers.tf file:

```
provider "google" {
  version     = "2.7.0"
  credentials = "${file(var.credentials)}"
  project     = var.project_id
  region      = var.region
}

provider "google-beta" {
  version     = "2.7.0"
  credentials = "${file(var.credentials)}"
  project     = var.project_id
  region      = var.region
}
```

Then create the following main.tf file where we reference the module mentioned earlier and provide it with the appropriate variables:

```
module "gke" {
  source                     = "terraform-google-modules/kubernetes-engine/google"
  version                    = "4.1.0"
  project_id                 = var.project_id
  region                     = var.region
  zones                      = var.zones
  name                       = var.name
  network                    = "default"
  subnetwork                 = "default"
  ip_range_pods              = ""
  ip_range_services          = ""
  http_load_balancing        = false
  horizontal_pod_autoscaling = true
  kubernetes_dashboard       = true
  network_policy             = true

  node_pools = [
    {
      name               = "default-node-pool"
      machine_type       = var.machine_type
      min_count          = var.min_count
      max_count          = var.max_count
      disk_size_gb       = var.disk_size_gb
      disk_type          = "pd-standard"
      image_type         = "COS"
      auto_repair        = true
      auto_upgrade       = true
      service_account    = var.service_account
      preemptible        = false
      initial_node_count = var.initial_node_count
    },
  ]

  node_pools_oauth_scopes = {
    all = []

    default-node-pool = [
      "https://www.googleapis.com/auth/cloud-platform",
    ]
  }

  node_pools_labels = {
    all = {}

    default-node-pool = {
      default-node-pool = true
    }
  }

  node_pools_metadata = {
    all = {}

    default-node-pool = {
      node-pool-metadata-custom-value = "my-node-pool"
    }
  }

  node_pools_taints = {
    all = []

    default-node-pool = [
      {
        key    = "default-node-pool"
        value  = true
        effect = "PREFER_NO_SCHEDULE"
      },
    ]
  }

  node_pools_tags = {
    all = []

    default-node-pool = [
      "default-node-pool",
    ]
  }
}
```

Now create a variables.tf file to describe the variables referenced in the previous file and their type:

```
variable "credentials" {
  type        = string
  description = "Location of the credentials keyfile."
}

variable "project_id" {
  type        = string
  description = "The project ID to host the cluster in."
}

variable "region" {
  type        = string
  description = "The region to host the cluster in."
}

variable "zones" {
  type        = list(string)
  description = "The zones to host the cluster in."
}

variable "name" {
  type        = string
  description = "The name of the cluster."
}

variable "machine_type" {
  type        = string
  description = "Type of the node compute engines."
}

variable "min_count" {
  type        = number
  description = "Minimum number of nodes in the NodePool. Must be >=0 and <= max_node_count."
}

variable "max_count" {
  type        = number
  description = "Maximum number of nodes in the NodePool. Must be >= min_node_count."
}

variable "disk_size_gb" {
  type        = number
  description = "Size of the node's disk."
}

variable "service_account" {
  type        = string
  description = "The service account to run nodes as if not overridden in `node_pools`. The create_service_account variable default value (true) will cause a cluster-specific service account to be created."
}

variable "initial_node_count" {
  type        = number
  description = "The number of nodes to create in this cluster's default node pool."
}
```

Finally, create the following variables.auto.tfvars file to specify values for the variables defined above:

```
credentials        = "./terraform-gke-keyfile.json"
project_id         = "<project_name>"
region             = "<region>"
zones              = ["<region>-a", "<region>-b", "<region>-c"]
name               = "gke-cluster"
machine_type       = "<machine_type>"
min_count          = 1
max_count          = 3
disk_size_gb       = 10
service_account    = "<service_account_name>@<project_name>.iam.gserviceaccount.com"
initial_node_count = 3

```

For region, you can choose the same as the location of your bucket, for example europe-west4.
For machine_type,  you can choose g1-small, it corresponds to a Compute Engine with 1 vCPU and 1.7 GB memory and is sufficient for a small Kubernetes cluster.

Now that we have created all the necessary files, let’s run terraform init again to install the required plugins. If you are curious, you can compare the content of the .terraform/ directory before and after running this command.

To get a complete list of the different resources Terraform will create to achieve the state described in the configuration files you just wrote, run :

```
terraform plan
```

And to create the GKE cluster, apply the plan:

```
terraform apply
```
When prompted for confirmation, type in “yes” and wait a few minutes for Terraform to build the cluster.

When Terraform is done, we can check the status of the cluster and configure the kubectl command line tool to connect to it with:

```
gcloud container clusters list
gcloud container clusters get-credentials gke-cluster
```

In this article, we have learned how to use Terraform to build a Kubernetes cluster on Google Cloud Platform.

Thanks
