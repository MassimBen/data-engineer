# Set up GCP account


## Configuration GCP account

**First step** : create a projet and take note the ID, you use for coniguration of terraform.

**Second step** : enable compute engine in the projet to allow terraform to provision the infrastructure of projet.

## Create a working directory

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



