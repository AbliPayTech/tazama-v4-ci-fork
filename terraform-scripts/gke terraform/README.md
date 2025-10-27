## Terraform Setup for an GKE Cluster.

The terraform files under `gke-terraform/` folder and automate the creation of the GKE cluster needed to run helm commands under the helm section..

The `gke-terraform` folder contains the following files;

|File                           | Description                                                                                     |
|-------------------------------|-------------------------------------------------------------------------------------------------|
|`backend.tf`                   | Specifies the backend storage bucket where the terraform state is remotely stored               |
|`sample.terraform.tfvars`      | Holds assigned values for the `variables.tf` staging env                                        |
|`main.tf`                      | Defines modules and resources for the infrastructure to be created in Google Cloud.             |
|`outputs.tf`                   | Return values that will be displayed after terraform runs successfully.                         |
|`variables.tf`                 | Defines valid variables for the templates which serve as parameters for the modules / resources.|
|`provider.tf`                  | Specifies terraform provider versions.                                                          |

### Prerequisites

- Terraform Version "~> 1.2.0"
