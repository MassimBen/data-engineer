# Set up GCP account and using of Terraform for create infrastructure

## Part 1 : Building infra with Terraform

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

### Create infrastructure

If all thing is right, use terraform plan to verify all ressource will be create correctly and use terraform apply to create infra in GCP in this case.
In below example of creating infra 

```
$ terraform apply

Terraform used the selected providers to generate the following execution plan. Resource actions
are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # google_compute_network.vpc_network will be created
  + resource "google_compute_network" "vpc_network" {
      + auto_create_subnetworks                   = true
      + delete_default_routes_on_create           = false
      + gateway_ipv4                              = (known after apply)
      + id                                        = (known after apply)
      + internal_ipv6_range                       = (known after apply)
      + mtu                                       = (known after apply)
      + name                                      = "terraform-network"
      + network_firewall_policy_enforcement_order = "AFTER_CLASSIC_FIREWALL"
      + numeric_id                                = (known after apply)
      + project                                   = (known after apply)
      + routing_mode                              = (known after apply)
      + self_link                                 = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:

```
Go on GCP look the infra create becarefull for looking the region and projet you specify in the configuration.

### Inspect state

Terraform create terraforme.tfstate to save your configuration and manage when you to chnage ou destroy it but he keep all information ( ID_projet and others)

You can use terraform show to inspect you current 

```
$ terraform show
# google_compute_network.vpc_network:
resource "google_compute_network" "vpc_network" {
    auto_create_subnetworks                   = true
    delete_default_routes_on_create           = false
    description                               = null
    enable_ula_internal_ipv6                  = false
    gateway_ipv4                              = null
    id                                        = "projects/test-project/global/networks/terraform-network"
    internal_ipv6_range                       = null
    mtu                                       = 0
    name                                      = "terraform-network"
    network_firewall_policy_enforcement_order = "AFTER_CLASSIC_FIREWALL"
    numeric_id                                = "1234567890123456789"
    project                                   = "test-project"
    routing_mode                              = "REGIONAL"
    self_link                                 = "https://www.googleapis.com/compute/v1/projects/test-project/global/networks/terraform-network"
```

## Part 2 : Change infrastructure

### Create a new resource
You can modify and change your infra in Terraform, you modify the main.tf file you do directly terraform apply to send your modification or use terraform plan to verify before execute terraform apply to provision your infra. In below example of adding new ressource.

```
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
```
### Modify a ressource 

When you modify a ressource it's update the ressource, example below

```
 resource "google_compute_instance" "vm_instance" {
   name         = "terraform-instance"
   machine_type = "f1-micro"
  tags         = ["web", "dev"]
   ## ...
```
But sometime a provider don't allow the modification, in this case you use destroy change, you destrioy and replace with a new you create.


### Destroy infrastructure

In Terraform you can destroy infra when you don't use it and don't pay cost for keep in the provider cloud, to destroy you use terraform destroy.
example below.

```
$ terraform destroy

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # google_compute_instance.vm_instance will be destroyed
  - resource "google_compute_instance" "vm_instance" {
      - can_ip_forward       = false -> null
      - cpu_platform         = "Intel Haswell" -> null
      - deletion_protection  = false -> null
      - enable_display       = false -> null
      - guest_accelerator    = [] -> null
      - id                   = "projects/testing-project/zones/us-central1-c/instances/terraform-instance" -> null
      - instance_id          = "1820538232654796756" -> null
      - label_fingerprint    = "42WmSpB8rSM=" -> null
      - machine_type         = "f1-micro" -> null

      ## ...

    }

  # google_compute_network.vpc_network will be destroyed
  - resource "google_compute_network" "vpc_network" {
      - auto_create_subnetworks         = true -> null
      - delete_default_routes_on_create = false -> null
      - id                              = "projects/testing-project/global/networks/terraform-network" -> null
      - name                            = "terraform-network" -> null
      - project                         = "testing-project" -> null
      - routing_mode                    = "REGIONAL" -> null
      - self_link                       = "https://www.googleapis.com/compute/v1/projects/testing-project/global/networks/terraform-network" -> null
    }

Plan: 0 to add, 0 to change, 2 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value:
```

the `-` specify the instance and ressource will be destroy, terraform establish priority of desctruction when use the ressource, for complex terraforme use operation in paralel.

### Define input variables

In Terraform you can use varaible to use dynamic code otherwise a hard-coding, create variable.tf ( you can give an other name but finish with .tf)

```
variable "project" { }

variable "region" {
  default = "us-central1"
}

variable "zone" {
  default = "us-central1-c"
}


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
 project = var.project
 region  = var.region
 zone    = var.zone
}
```

In this example theyou create VM machine boot_disk and network_interface need use complex paramater, for this example use simple paramater. You use the network you create before to connect the VM machine, add acces_config give a IP adress to VM, you can have access over the internet.
