# Set up GCP account and using of Terraform for create infrastructure

## Parti 1 : Building infra with Terraform

### Configuration GCP account

**First step** : create a projet and take note the ID, you use for coniguration of terraform.

**Second step** : enable compute engine in the projet to allow terraform to provision the infrastructure of projet.

### Create a working directory

After configuration you create folder for terraform projet and go there, create the main.tf file and set configuration, define terraform block with version of terraform you use the required_providers define the source and the proviser.

```terraform
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
      version = "6.8.0"
    }
  }
}

provider "google" {
  project = "<PROJECT_ID>"
  region  = "us-central1"
  zone    = "us-central1-c"
}
```
Replace projet_ID with you iD projet and other parameter with parameter for your situation.

Define ressource you want to create and manage in the infrastructure, in this exemple you create a network 

```
resource "google_compute_network" "vpc_network" {
  name = "terraform-network"

```
The ressource you create depend what you want to use in your infrastructure.

After configuration the main.tf with all element (terraform block, provider and ressource), now we need to authenticate in our googgle cloud account.

### Authenticate to Google cloud

```
$ gcloud auth application-default login
```

A prompt of connection will appear in your screem, you log in when your are succefull authentication credential saved in your computer, the path will give 

Example :

```
Credentials saved to file: [/Users/USER/.config/gcloud/application_default_credentials.json]

These credentials will be used by any library that requests Application Default Credentials (ADC).
```

The GCP provider automatically uses these credentials to authenticate against the Google Cloud APIs.

### Initialize the directory

You need to run terraform init, to dowlaods all element for the provider you speficie in main.tf, he create in the directpry a hidden subdirectry the are .terraform file. Terraform also creates a lock file named .terraform.lock.hcl, which specifies the exact provider versions used to ensure that every Terraform run is consistent. This also allows you to control when you want to upgrade the providers used in your configuration.

### Format and validate the configuration
We recommend using consistent formatting in all of your configuration files. The terraform fmt command automatically updates configurations in the current directory for readability and consistency.

Format your configuration. Terraform will print out the names of the files it modified, if any. In this case, your configuration file was already formatted correctly, so Terraform won't return any file names.

```
$ terraform fmt
```

For verification use terraform validate 

```
$ terraform validate
Success! The configuration is valid.

```

##


