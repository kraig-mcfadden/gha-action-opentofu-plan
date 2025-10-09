# gha-action-opentofu-plan
Runs an OpenTofu plan. Assumes you're using AWS.

Inputs to the apply are not configurable at the moment. They may be in a future release.

Provides default names for the infrastructure folder as well as the AWS role to assume to allow creating and deleting resources. Those can be overridden.
