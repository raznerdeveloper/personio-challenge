# IAC Overview
## Infrastructure
The infrastructure was built using Terraform for IAC. Points considered when designing this setup were, production readiness, scalability and maintainability. This setup spins up a kubernetes cluster on AWS EKS along with zones, vpc, etc., necessary to work with the cluster. The folder structure is as follows:
```
infrastructure
├── modules
│   └── external_dns
│   │   └── main.tf
│   │   └── variables.tf
│   └── null_resources
│   │   └── main.tf
│   │   └── variables.tf
│   └── remote_state
│       └── main.tf
│       └── variables.tf
└── data.tf
└── locals.tf
└── main.tf
└── output-eks.tf
└── output-zones.tf
└── providers.tf
└── variables-eks.tf
└── variables-vpc.tf
```

### Modules
Default modules were used in this setup (zones, vpc, eks) however, some custom modules were created as well which I will explain:
### external_dns
This ensures that whenever we have an application specific ingress with a subdomain specified, this record will be automatically created in AWS Route53 if it doesn't exist. It uses the helm provider as it's a helm chart applied to the EKS cluster.
### null_resources
This enables us to run regular bash commands as it uses the `local-exec` provisioner.
### remote_state
This enables us to have a synchronized terraform state across members of a team using terraform. Now, team members working with local states can be a problem when everyone has different states of the same infrastructe on their local machines. The remote state module helps mitigate this by creating what's needed to store and lock the terraform state on AWS which are, an s3 bucket and a dynamo db table. How this works is, when someone newly clones the repo, during the initialization process (`terraform init`), if there's a remote state, they'll be asked to use it and once they say yes, they start working with the remote state.
### Files
* The `data` files generally hold data from data sources.
* The `variables-*` files were created as such so as to be able to make clear what variable is for what. This makes it easier when working with variables in the codebase. 
* Same deal with the `output-*` files. We can easily distinguish what output is for what.
* We have three `providers` setup, `helm`, `kubernetes` and `aws`. The helm one helps with applying the `external_dns` helm chart and any other helm chart in the future into the cluster.

## Running The Infrastructure Setup (Putting it all together)
In order to run the infrastructure bit, you need these:
* Terraform version 13.3
* AWS cli configured with and AWS credential that has access to an AWS account with some credits.
* kubectl
* Go in the `main.tf` in the `infrastructure` root directory and change the value for the `zone` module from `k8s.lawrencetalks.com` to a subdomain you manage

Then do `terraform init` to initialize terraform. This will pull in the required modules and then setup the state. Next:
* Do `terraform plan` to view the changes you're about to make to the your AWS infra
* Do `terraform apply` to make the changes. This spins up the kubernetes cluster with the vpc, zones etc., sets up remote state and updates your `kubeconfig` with enteries for the newly created cluster. Also from the outputs, check for the `ns` records in the `this_route53_zone_name_servers` output from the created zone and go update it over at your domain registrar in order to have AWS manage that subdomain.

Note: for a first time setup, after doing `terraform apply`, you'll get a `backend.tf` file generated. Please commit this file as this helps with working with the remote state.

## Deploying The Application
First go modify `application -> k8s -> ingress.yaml`, chaning the host from `go.k8s.lawrencetalks.com` to a subdomain of your choice as it relates to the zone you selected when creating the infrastructure and then Simply do `kubectl apply` on the `k8s` folder in the application to apply the kubernetes manifest to the cluster

## Proposed CI/CD setup for application
In order to fully automate the deployment of both application and infrastructure, I would suggest having them both in different repositories, have them run in separate pipelines independent of themselves. For more flexibility, the CI pipeline can use a docker runner with all required tools baked in.
### The Application
The application's pipeline would have the following steps:
* A `quality` step that runs tests and/or linters if any.
* A `build` step that builds and tags the docker image with the current pipeline ID and then pushes it to a container registry.
* A `deploy` step that applies the kubernetes manifests. For a production branch and environment of course, this action has to be manual to avoid automatically deploying changes not ready for production to production.
* An optional `release` step that updates the git tag for the current HEAD using semver, based on the team.

In order to rollback a previous deployment, I would use `helm` for the deployment instead of the regular kubernetes manifests as used in this challenge. Helm helps with rollback out of the box by storing previous releases in configmaps in the cluster and rolling back to them when requested.

### The Infrastructure
The infrastructure's pipeline would have the following steps:
* A `linting` step that would run `terraform validate` to check syntax etc.
* A `apply` step that pushes the validated update to the infrastructure.

## Improvements
* Having the application use helm instead
* Having separate repositories for application and infrastructure
* Having pipelines for both to automatically deploy them
