** TF-training-f5 **


** Terraform for Infrastructure as Code **

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


** Set up your Google Cloud Platform working environment **

Before we can start using GCP, we need to create a project and activate billing on it. Don’t worry, this will not cost you anything, Google offers $300 in free credit when you start using GCP and will never charge you unless you manually upgrade to a paid account.
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
