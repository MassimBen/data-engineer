# Set up GCP account


## Configuration GCP account

**First step** : create a projet and take note the ID, you use for coniguration of terraform.
**Second step** : enable compute engine in the projet to allow terraform to provision the infrastructure of projet.

## Create a working directory

After configuration you create folder for terraform projet and go there, create the main.tf file and set configuration, define terraform block with version of terraform you use the required_providers with define the soruce and the proviser.

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


