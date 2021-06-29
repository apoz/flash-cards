
## Commands

### init

You have got to run this command once when working with new terraform code.

With this command terraform will:
- download the code for the providers used in terraform config file. The code for the providers is stored in a `.terraform` folder.

This command is idempotent, you can run it several times in the same directory.

The following lines should be added to the `.gitignore` file in a repo:
```
.terraform
*.tfstate
*.tfstate.backup
```

### plan

This command does NOT perform any change in the infra, but check what is the current status of the infra and the desired status. It shows then what changes would be made in the infra if we would like to apply the terraform configuration file to the infra.

### apply

This commands first plans the changes in infra Vs the configuration file and then, after prompting the user for confirmation, applies the changes to obtain the desired state.

### graph

Outputs (in DOT languaje) of the dependency graph for the terraform file/files.

### output

Outputs the output variables for the terraform code.
```
terraform output

# or 

terraform output var_name
```

### refresh
Updates the terraform status file with the current infra status.

### destroy

Removes the infrastructure.

### workspace

Shows and sets the workspace.

```
terraform workspace show

terraform workspace new my_new_workspace

terraform workspace select my_workspace
```

## Concepts

### Expressions

Usually they are references, that point to values from other parts of the Terraform code. The reference attribute reference has the following syntax:
```
<PROVIDER>_<TYPE>.<NAME>.<ATTRIBUTE>

#for example
aws_security_group.instance.id
```

### Variables

#### Input variables
```
variable "NAME" {
    description: "Optional. Some explanation about the variable",
    default: Optional. Default value for the variable. The value set if not provided by command line (with the `-var` option or with a environment variable with the name "TF_VAR_<var_name>").
    type: Optional. Type constraints in the variable. It can be string, number, bool, list, map, set, object, tuple, and any. If nothing is specified it's `any`.
```
If not default value is specified and the value is not set, the user will be prompted to include a value when running apply.

The values can be specified with the following syntax:
```
terraform plan -var "server_port=8080
```

The reference to access the value of a variable is:
```
var.<VARIABLE_NAME>

# and to reference the value inside a string

${var.<VARIABLE_NAME>}
```

#### Output variables

```
output "<NAME>" {
    value = <VALUE>,
    description = "Blah, blah, blah",
    sensitive = boolean to indicate terraform not to outpu the value

}
```

### Lifecycle setting

In order to manage the dependency tree, it's possible we have to set a `lifecycle` section in order not do delete a resource before creating the new one (aws wouldn't allow it).

```
lifecycle {
    create_before_destroy = true
}
```

We can prevent also the deletion of a resource:
```
lifecycle {
    prevent_destroy = true
}
```

### Datasources

Represents a piece of read-only information that is fetched from the provider.

```
data "<PROVIDER>_<TYPE>" "<NAME> {
    config...
}

# for example
data "aws_vpc" "default" {
  default = true
}

# To reference the data:
data.<PROVIDER>_<TYPE>.<NAME>.<ATTRIBUTE>
```

## Terraform state

Everytime we run terraform, it records information about what infra it created in a terraform state file (typically `terraform.tfstate` in the current directory).

When more than one person is using terraform (a team) there are some issues with terraform state file that we need to address:

- Shared storage of terraform state files.
- Locking state files, to avoid race conditions.
- Isolating state files per environment.

It's  a bad idea to store terraform state files in version control:
- Manual errors forgetting to upload the file after applying some change
- Lack of locking
- Secrets: all data in terraform state files is stored in plain text.

The best way to manage shared storage files is to use Terraform build-in support for remote backends:
- Terraform will load the state after a plan or apply command.
- It will include locking
- Remote backends usually support encryption
- Supports versioning

Recommended option, if you a re using AWS, is to use S3 for storage with DynamoDB for locking.

Example of creating, with terraform, the S3 bucket and DynamoDB db for locking:

```
provider "aws" {
  region = "us-east-2"
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"

  # Prevent accidental deletion of this S3 bucket
  lifecycle {
    prevent_destroy = true
  }

  # Enable versioning so we can see the full revision history of our
  # state files
  versioning {
    enabled = true
  }

  # Enable server-side encryption by default
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "my-terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

```

Then, we have to configure terraform to make use of that remote backend. With a `init` command the local state file will be uploaded to the S3 bucket.
```
terraform {
  backend "s3" {
    # Replace this with your bucket name!
    bucket         = "my-terraform-state"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-2"

    # Replace this with your DynamoDB table name!
    dynamodb_table = "y-terraform-state-locks"
    encrypt        = true
  }
}
```

There's the chicken-egg problem with the creation of the S3 bucket and the dynamoDB table. First we've got to create that infra with the local backend, then apply the config to use the remote backend and run an `init` command to upload the state to the bucket. For the deletion, we have to delete the remote backend config, run init to copy the state to the local disk and then run `destroy` to remove the infra.

The `backend` configuration code does not allow you to use any variable or reference.
The `key` parameter in the s3 has to be unique per module!

It's better to use a partial configuration strategy, where you can pass a file with some variable definitions  in the form:
```
# backend.hcl
bucket         = "my-terraform-state"
region         = "us-east-2"
dynamodb_table = "my-terraform-locks"
enc

enrypt        = true
```
and pass it as parameter when running terraform with `terraform init -backend-config=backend.hcl`.


Regarding isolation, there are 2 ways to perform isolation:
- Vía workspaces
Not recommended as they are error prone.
- Via file layout
Put the terraform config files in a folder per environment.
Configure different backend per environment (with different authentication mechanisms)

### Terraform remote state datasource

There's an special data source called `terraform_remote_state` used for when we need to check some data from a remote terraform status file (read-only).

We have to define the values as outputs in the terraform state file and then:

```
data "terraform_remote_state" "db" {
  backend = "s3"

  config = {
    bucket = "(YOUR_BUCKET_NAME)"
    key    = "stage/data-stores/mysql/terraform.tfstate"
    region = "us-east-2"
  }
}

# Then we will be able to access the data as
data.terraform_remote_state.db.outputs.ATTRIBUTE

```

## Built-in functions

Terraform provides some functions that can be accessed fron the config files.

For example, a file can be loaded using the `file` function.

```
file("user-data.sh")
```

That feature can be combined with the `template_file` data source. Example:

```
data "template_file" "user_data" {
  template = file("user-data.sh")

  vars = {
    server_port = var.server_port
    db_address  = data.terraform_remote_state.db.outputs.address
    db_port     = data.terraform_remote_state.db.outputs.port
  }
}
```

...where the script has the following format...

```
#!/bin/bash

cat > index.html <<EOF
<h1>Hello, World</h1>
<p>DB address: ${db_address}</p>
```

...and we can access the vaue as...

```
user_data       = data.template_file.user_data.rendered
```


# Pag 192
