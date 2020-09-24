# terraform-aws-backend
A Terraform module which enables you to create and manage your [Terraform AWS Backend resources](https://www.terraform.io/docs/backends/types/s3.html), _with terraform_ to achieve a best practice setup.

More info on aws (s3/dynamo) backend supported by this module is found here:

https://www.terraform.io/docs/backends/types/s3.html

Note: This fork uses PAY-PER-REQUEST for dynamodb and add a default programatic name for s3 backend

# Bootstrapping your project

[This module has a few options which are documented below. They allow you to change the behavior of this module.](#module-options)

**_Why does this exist?_**

If your project [specifies an AWS/S3 backend](https://www.terraform.io/docs/backends/types/s3.html), Terraform requires existence of an S3 bucket in which to store _state_ information and a DynamoDB table for state locking.

This terraform module creates/manages those resources:

* Versioned S3 bucket for state
* Properly configured DynamoDB lock table

**If you follow this README carefully, you should be able to avoid circular dependency which is inherent to the problem at hand.**


**_What circular dependency?_**

Your resulting terraform configuration block will refer to resources created by this module. You wouldn't be able to `plan` or `apply` if your state bucket and lock table don't exist. The details which make this work can be seen under [the section which encourages you to postpone writing your terraform configuration block](#postpone-writing-your-terraform-configuration-block) and [specific options used in the commands section below](#commands-are-the-fun-part).


### Note on state bucket and s3 key naming

Can be specified in module block with:
```
  backend_bucket = "terraform-state-bucket"
```

Otherwise it's generated from:
```
   state_bucket   = "${var.account_alias}-${var.identifer}"
```
which are also specified as variables in module block

### Postpone writing your terraform configuration block

In order to bootstrap your project with this module/setup, wait until **after** Step 4 (below) to write [terraform configuration block](https://www.terraform.io/docs/configuration/terraform.html) into one of your `.tf` files. (Your "terraform configuration block" is the one that looks like this `terraform {}`.)

If you are updating an existing terraform-managed project, or you already wrote your `terraform {...}` block into one of your `.tf` files, you will run into the following error on Step 3 (`terraform plan`):

![reinit required error](http://g.samstav.xyz/av5vyblbwq.png)


### describe your terraform backend resources

```hcl
module "backend" {
  source = "github.com/AnthonyWC/terraform-aws-backend"

  account_alias = "hashicorp"
  identifer = "prod"

  # Optional to override generated name
  # backend_bucket = "terraform-state-bucket"

  # using options, e.g. if you dont want a dynamodb lock table, uncomment this:
  # dynamodb_lock_table_enabled = false
}
```

### if using _existing_ backend resources (instead of creating new ones)

#### re-using a DynamoDB lock table across multiple terraform-managed projects

One of the resources created and managed by this module is the DynamoDB Table for [terraform locking](https://www.terraform.io/docs/state/locking.html). This module provides a default name: `terraform-lock`. This table may actually be re-used across multiple different projects. In the case that you already have a DynamoDB table you would like to use for locking (or perhaps you are already using this module in another project), you can simply import that dynamodb table:

```bash
# If you are running Terraform at >=0.12.*...
$ terraform import module.backend.aws_dynamodb_table.tf_backend_state_lock_table terraform-lock

# ...or if you are running it at <0.12.*
$ terraform import module.backend.aws_dynamodb_table.tf_backend_state_lock_table[0] terraform-lock
```

_(The `[0]` is needed in <0.12.* because it is a "conditional resource" and you must refer to the 'count' index when importing, which is always `[0]`)_

Where `backend` is your chosen `terraform-aws-backend` module instance name, and `terraform-lock` is the DynamoDB table name you use for tf state locking.

If you attempt to apply this module without importing the existing DynamoDB table with the same name, you will run into the following error:

```
Error: Error applying plan:

1 error(s) occurred:

* module.backend.aws_dynamodb_table.tf_backend_state_lock_table: 1 error(s) occurred:

* aws_dynamodb_table.tf_backend_state_lock_table: ResourceInUseException: Table already exists: terraform-lock
	status code: 400, request id: F35KO0U78JJOIWEJFNJNJHSLDBFF66Q9ASUAAJG
 ```

### commands to run:

```bash
# Step 1: Download modules
terraform get -update

# Step 2: Initialize your directory/project for use with terraform
# Note: Use of -backend=false here is important: Avoid backend configuration initially as the backend resources is not created yet
terraform init -backend=false

# Step 3: Create infrastructure plan for just the tf backend resources
# Target only resources needed for aws backend for terraform state/locking
terraform plan -out=backend.plan -target=module.backend

# Step 4: Apply the infrastructure plan
terraform apply backend.plan

# Step 5: Only after applying (building) the backend resources, write our terraform config.
# Now write terraform backend configuration into our project

# Instead of this command, can instead write terraform config block in a .tf file
# Please see "writing your terraform configuration" below for more info
echo 'terraform { backend "s3" {} }' > conf.tf

# Step 6: Reinitialize terraform to use your newly provisioned backend
terraform init -reconfigure \
    -backend-config="bucket=terraform-state-bucket" \
    -backend-config="key=states/terraform.tfstate" \
    -backend-config="encrypt=1" \
    # leave this next line out if you dont want to use a tf lock
    -backend-config="dynamodb_table=terraform-lock"
```

### writing your terraform configuration

https://www.terraform.io/docs/configuration/terraform.html

Instead of using the `echo` command above in Step 5 (provided only for proof of concept), you can just write your terraform config into one of your \*.tf files. Otherwise you'll end up needing to provide the `-backend-config` [parameters partial configuration](https://www.terraform.io/docs/backends/config.html#partial-configuration) every single time you run `terraform init` (which might be often).

```hcl
terraform {
  backend "s3" {
    bucket = "terraform-state-bucket"
    key = "states/terraform.tfstate"
    dynamodb_table = "terraform-lock"
    encrypt = "true"
  }
}
```

### reconfiguring terraform after building your backend resources

Terraform might ask you if you want to copy your existing state. You probably do:

![yes](http://g.samstav.xyz/bgs7hwsiqa.png)

## Module options

Options and configuration for this module are exposed via terraform variables.


#### `backend_bucket`

Create the s3 backend bucket; default as:
```
 state_bucket   = "${var.account_alias}-${var.identifer}"
```

Can be explicitly defined to override default value in your terraform-aws-backend module block. e.g.


```hcl
module "backend" {
  source = "github.com/AnthonyWC/terraform-aws-backend"

  account_alias = "hashicorp"
  identifer = "prod"

  # Optional to override generated name
  # backend_bucket = "terraform-state-bucket"
}
```

OR

```hcl
variable "backend_bucket" {
  default = "terraform-state-bucket"
}

module "backend" {
  source = "github.com/AnthonyWC/terraform-aws-backend"
  backend_bucket = "${var.backend_bucket}"
}
```

#### `dynamodb_lock_table_enabled`

_Defaults to `true`._

- Set to false or 0 to prevent this module from creating the DynamoDB table to use for terraform state locking and consistency. More info on locking for aws/s3 backends: https://www.terraform.io/docs/backends/types/s3.html. More information about how terraform handles booleans here: https://www.terraform.io/docs/configuration/variables.html"
}

#### `dynamodb_lock_table_stream_enabled`

_Defaults to `false`._

Affects terraform-aws-backend module behavior. Set to false or 0 to disable DynamoDB Streams for the table. More info on DynamoDB streams: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html. More information about how terraform handles booleans here: https://www.terraform.io/docs/configuration/variables.html


#### `dynamodb_lock_table_stream_view_type`

_Defaults to `NEW_AND_OLD_IMAGES`_

Only applies if `dynamodb_lock_table_stream_enabled` is true.

#### `dynamodb_lock_table_name`

_Defaults to `terraform-lock`_

Name of your [terraform state locking](https://www.terraform.io/docs/state/locking.html) DynamoDB Table.

#### `lock_table_read_capacity`

_Defaults to `1` Read Capacity Unit._

More on DynamoDB Capacity Units: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/CapacityUnitCalculations.html


#### `lock_table_write_capacity`
_Defaults to `1` Write Capacity Unit._

More on DynamoDB Capacity Units: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/CapacityUnitCalculations.html

#### `kms_key_id`
_Defaults to ``._

Encryption key to use for encrypting the terraform remote state s3 bucket. If not specified, then AWS-S3 encryption key management method will be used, which uses keys derived from the account master kms key. If specified, then AWS-KMS encryption key management method will be used. If the kms_key_id is specified, then you must specify the backend config option `kms_key_id`. More on s3 bucket server side encryption: https://docs.aws.amazon.com/AmazonS3/latest/dev/serv-side-encryption.html and https://docs.aws.amazon.com/AmazonS3/latest/dev/bucket-encryption.html

### terraform-aws-backend terraform variables

See variables available for module configuration

https://github.com/AnthonyWC/terraform-aws-backend/blob/master/variables.tf
