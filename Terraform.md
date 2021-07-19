# Terraform

## Terraform general considerations

- The possible outputs a resource can output are listed in the documentation as resource attributes.
- Terraform generally loads all the config files within the directory in alphabetical order. The files must end in either `.tf` or `.tf.json`

## Terraform Providers

A provider is responsible form understanding API interactions and exposing resources.
Most of the available providers correspond to one cloud or on-premises infrastructure platform, and offer resource types that correspond to the features of that platform.

You can explicitly set a specific section of the provider within the provider block.
To updgrade to the latests acceptable version of each provider run `terraform init -upgrade`

You can have more than one instance of provider with the help of the `alias` parameter.
The provider without alias is the default provider.

Provider configuration block is not mandatory for all the terrraform configurations.

### Required providers

Each terraform module must declare which providers it requires.
Providers are declared:
```
terraform {
  required_providers {
    mycloud = {
      source = ""
      version = ""
    }
  }
}
```

### Handilng access and secret keys in providers

Never ever have any credentials in a vars or resources file.

For aws, if we have the aws cli with the credentials properly configured, we can remove the access keys from the provider, because terraform will take them from there.

Provider configuration if we want to deploy some stuff in different aws regions or with different accounts. Up to now we've configured those values at provisioner level.

```
provider "aws" {
  region = "us-west-1"
  provider = "aws.east"
}

provider "aws" {
  alias = "east"
  region = "us-east-1"
}

resource "aws_eip" "myeip" {
  vpc = "true"
}


resource "aws_eip" "myeip02" {
  vpc = "true"
  provider = asw.east
}

module "aws_vpc" {
  source = "./aws_vpc"
  providers = {
    aws = aws.east
  }
}
```

Alias allows us to have more than one configuration per provider.

### Terraform configuration language

- A terraform workspace is simply a folder that contains terraform code.
- Terraform files always end in `*.tf` of `*.tfvars`
- Most terraform workspaces contain a minimum of 3 files:
  - *main.tf*: Functional code.
  - *variables.tf*: This file is for storing variables.
  - *outputs.tf*: Define what is shown at the end of a terraform run.

```
# comment

/* Multiline comment
are written this way */
```

## Terraform Commands

### terraform init

You have got to run this command at least once when working with new terraform code.

With this command terraform will download the code for the providers and modules used in terraform code. The code for the providers and modules is stored in a `.terraform` folder. If you add, change or update your modules or providers (or their version requirements) you will need to run it again.

This command is idempotent, you can run it several times in the same directory.

The following lines should be added to the `.gitignore` file in a repo in order NOT to upload those files to the git repository (if the code is stored in git):
```
.terraform
*.tfstate
*.tfstate.backup
```

Terraform lock file to record the provider selection, and avoid mistake deletions.
With `terraform init -upgrade` we can upgrade versions of providers.

### terraform plan

This command does NOT perform any change in the infra, but check **what is the current status of the infra** and the difference with the desired status. 
It shows then what changes would be made in the infra if we would like to apply the terraform configuration file to the infra.

The output meanings:

- `+` something will be added
- `-` something will be deleted
- `~` update in place

We can also output the terraform plan to a file. This allows us to assure only the changes shown in that terraform plan are applied.

```
terraform plan -out=path
#then
terraform apply path
```

We can prevent terraform from queryin the current state during operationes as terraform plan ( with the option `-refresh=false`) if we need to reduce the amount of API calls to a provider when doing some plan and apply.

### terraform apply

This commands first updates the current status, then plans the changes that should be made in infra from the current status Vs the desired state (configuration files) and then, after prompting the user for confirmation, applies the changes to obtain the desired state (with `-auto-approve` option you can skip the program prompting for confirmation `yes`).

You can apply some specific part/resource of the code only with `-target=resource` option.


### terraform graph

Terraform outputs (in DOT languaje) the dependency graph for the terraform file/files.

```
terraform graph > graph.dot
#graphviz package
cat graph.dot | dot -Tsvg > graph.svg

```

### terraform fmt

Linter to set the format of the terraform files properly.

### terraform validate

Validates the configuration files for terraform.

### terraform output

Outputs the output variables for the terraform code taking its value from the state file.

```
terraform output

# or 

terraform output var_name
```

### terraform login

It will save the token you provide in a file so you don't have to deal with the authorization.

### terraform refresh
Updates the terraform status file with the current infra status.

### terraform state

Terraform command to deal with state modifications. Subcommands:
- *list*: list resources within terraform state file
- *mv* (source destination): moves item with terraform state. Used mainly if you want to rename an existing resource without destroying and recreating it. This command outputs a backup copy before doing any change.
- *pull*: manually download and output the state from remote state. Useful for reading values out of a state file.
- *push*: manually upload a local state file to a remote state.
- *rm*: remove items from a terraform state file. Items removed from the Terraform state are NOT physically destroyed. Are no longer managed by terraform. If you do a terraform plan with the resource in the code, it will be recreated again.
- *show*: shows attributes of a single resource in the terraform state

### terraform import

If a resource has been created manually, you can import the resources with import.

We have to create a resource with the same data than the manually created one and then run the terraform import code. 

```
terraform import aws_instance.myec2  INSTANCE_ID_OF_THE_MANUALLY_CREATED_INSTANCE
```

### terraform destroy

Removes the infrastructure.

We can use `-target <PROVIDER>_<RESOURCE>.<NAME>` if we want to delete just some element.

### terraform console

It gives a console for you to check the output of functions, etc.

#### terraform taint

Terraform taint command manually marks a terraform managed resource as tainted, forcing it to be destroyed and recreated on the next apply commnad.

```
terraform taint aws_instance.myec2
```

The only modification that happens is that the status field of the resource in the state file is "tainted".

Note that tainting a resource for recreation may affect resources that depend on the newly tainted resource. It's important to check the dependency graph whe using taint command.

### terraform workspace

Shows and sets the workspace.

Subcommands:
- **show**: shows the available workspaces.
- **new**: creates a new workspace.
- **select**: selects a workspace to be used.

```
terraform workspace show

terraform workspace new my_new_workspace

terraform workspace select my_workspace
```

## Terraform code concepts

### Expressions

Usually they are references, that point to values from other parts of the Terraform code. The reference attribute reference has the following syntax:
```
<PROVIDER>_<TYPE>.<NAME>.<ATTRIBUTE>

#for example
aws_security_group.instance.id
```

### Terraform settings

The configuration of terraform and its providers is set in a `terraform` section in the terraform code.

```
terraform {
  required_version = "> 0.12.0"
  required_providers = {
    aws = " > 0.1.1"
    }
  }
```

### Dealing with larger infrastructure

When dealing with larger infraestructure, you will face issues related to API limits for a provider.

Solutions:
- Switch to smaller configurations where each one can be applied independently.
- we can prevent terraform from queryin the current state during operationes as terraform plan ( `-refresh=false`)
- Apply some specific part only with `-target=resource`

### Terraform variables

#### Input variables
The format of the code to define variables is:

```
variable "NAME" {
    description: "Optional. Some explanation about the variable",
    default: Optional. Default value for the variable. The value set if not provided by command line (with the `-var` option or with a environment variable with the name "TF_VAR_<var_name>").
    type: Optional. Type constraints in the variable. It can be string, number, bool, list, map, set, object, tuple, and any. If nothing is specified it's `any`.
```
If not default value is specified and the value is not set, the user will be prompted to include a value when running apply.

The values can be specified with the following syntax in the CLI:
```
terraform plan -var "server_port=8080"
```
... or with a environment variable of the form `TF_VAR_<var_name>`
... or with a `.tfvars` file.

The reference to access the value of a variable is:
```
var.<VARIABLE_NAME>

# and to reference the value inside a string

${var.<VARIABLE_NAME>}
```

Ways to set variables (from highest precedence to lowest):
1 - Command line flag - run as a command line switch.
```
-var="<VARIABLE_NAME>=variableValue"
#
terraform plan -var "server_port=8080"
```
2 - Configuration file - set in the terraform.tfvars file
```
# .tfvars file
variable_name=value
```
If the file is called something else, the specific vars file has to be specified with a -var-file="filename".

3 - Environment variable - part of your shell environment
4 - Default config - default value in variables.tf
5 - User manual entry - if not specified, prompt the user for entry

##### Variables with undefined values

If you have variables with undefined values it will not directly result in an error.
Terraform will ask you to supply the value associated with them.

Environment variables can be used also to set variable values. TF_VAR_name

#### Count parameter: loops

You can include a count parameter in some resource and it will create that number of resources. A count.index atribute is differente for each instance of the array which can be accessed by `${count.index}`.

The identifier of the resource, then becomes a list automatically.

#### Conditional statements

Ternary operation 
```
condition ? true_val : false_val
```

We can achieve blocks to be executed or not depending on the content of a variable:

```
resource "aws_instance" "test" {

  ...
  count = var.istest == true ? 1 : 0
}
```

#### Local values

Values in the code that can be referenced multiple times.Local value assigns a name to an expression, allowing it to be used multiple times whithin a module without repeating it.

The expression of a local value can refer to other locals, but cycles are not allowed.
It's recommnended to group together locically-related local values into a single block, particularly if they depend on each other.

```
locals {
  common_tags = {
    Owner = "Devops team"
    Service = "backend
  }
}
resource "aws_instance" "test" {

  ...
  tag = local.common_tags

}

Local values can use functions, etc.

```



#### Output variables

Output values from a module or our code.

```
output "<NAME>" {
    value = <VALUE>,
    description = "Blah, blah, blah",
    sensitive = boolean to indicate terraform not to output the value directly on the std output

}
```

### Terraform Functions

There are several built in functions, the available functions can be checked in the terraform official documentation.


Examples:
- file(path_string) -> reads the content of the file in the given path and returns its context as string.

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

- element -> (list, index)  extracts an element from a list corresponding to the index
- lookup -> looks for a key in a map. It receives (map, key, default_value), if the key does not exist in the map, the default value is returned, it's optional-
- timestamp() current timestamp
- formatDate() specify the format  (format, timestamp)
- zipmap It constructs a map from a list of keys and a list of values.
```
zipmap(["uno","dos","tres"], [1,2,3])
```

### Lifecycle setting

In order to manage the dependency tree, it's possible we have to set a `lifecycle` section in order not do delete a resource before creating the new one (aws wouldn't allow it).

```
lifecycle {
    create_before_destroy = true
}
```

We can prevent also the deletion of a resource (this can be useful to specify resources that should NOT be deleted accidentally):
```
lifecycle {
    prevent_destroy = true
}
```

### Datasources

Represents a piece of read-only information that is fetched from a provider.

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

## Terraform logging

Terraform has detailed logs that can be activated setting the TF_LOG env variable to any value. You can set that variable to TRACE,DEBUG,INFO,WARN or ERROR to change the verbosity of the logs.

There's a TF_LOG_PATH variable also to store the terraform log to the specified log.

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

The best way to manage shared storage files is to use Terraform built-in support for remote backends:
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
    dynamodb_table = "my-terraform-state-locks"
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
encrypt        = true
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

## Terraform Modules

Configuration files that can be reused in several environments.
They are a set of terraform configuration files in a folder.

The provider definition should be configured by the user and NOT by the module itself.
Whenever you include a module in your terraform code you have to run `terraform init`

To use a module:

```
module "<NAME>" {
  source = "<SOURCE>"  # can be a path to a file or url to a repo(git@github.com:<OWNER>/<REPO>.git//<PATH>?ref=<VERSION>)

  ...
}
```


In the module you have to set input parameters to be able to modify the module behaviour.

You can define a `locals` section with variables you don't want to expose outside the module.

```
# comment
locals {
  http_port    = 80
  any_port     = 0
  any_protocol = "-1"
  tcp_protocol = "tcp"
  all_ips      = ["0.0.0.0/0"]
}
```

And then use `local.<NAME>` to access the values.

We can define also output values that can be accessed by `output.<MODULE_NAME>.<OUTPUT_NAME>`

It's necesary to be careful about the paths used in the module. There are some path references that can be used:

```
path.module -> path of the module
path.root -> filesystem path of the root module
path.cwd -> current working directory
```


## Terraform providers

From 0.13 version onwards, terraform requires explicit source information for any provider NOT maintained by HashiCorp inside the terraform configuration block:

```
terraform {
  required_providers {
    digitalocean = {
      source = "digitalocean/digitalocean"
    }
  }
}

provider "digitalocean" {
  token = "mydigitaloceantoken"
}
```

It's important  to set the version, the accepted format for version specifying is the following.
```
>=1.0   # greater or equal 1.0
<=1.0   # less than or equal 1.0
~>2.0   # any version in the 2.0 range
>=2.10,<=2.30  # version between those 2
```

Terraform lock file to record the provider selection, and avoid mistake deletions.
With `terraform init -upgrade` we can upgrade versions of providers.



####################

#

## Dynamic block in terraform

Dynamic blocks in terraform allows ut to dynamically construct repeatable nested blocks with is supported inside the resource, data, provider and provisioner blocks.

```

variable "sg_ports" {
  type = list(number)
  description = "List of input ports for the SG"
  default = [8200, 8300, 8400]
}

resource "aws_security_group" "dynamicsg" {
  name = "dynamic-sg"
  description = "ingress for vault"

  dynamic "ingress" {
    for_each = var.ingress_ports
    iterator = port
    content {
      from_port = port.value
      to_port = port.value
      protocol = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
```


## Splat expression

It allows us to get a list of all the attributes or the attributes of a list of elements.

```
resource "aws_iam_user" "lb" {
  name = "iamuser.${count.index}"
  count = 3
  path = "/system/"
}

output "arns" {
  value = aws_iam_user.lb[*].arn
}
```


```
resource "aws_instance" "MYNAME" {
  ami = "asdf"
  ...
  ebs_block_device {
    asldkfjadsf
  }
  ebs_block_device {
    asldkfjadsf
  }
}
```

You can use the splat expression here : aws_instance.MYNAME.ebs_block_device[*].device_name

Terminology:
- Resource type -> aws_instance
- Local name -> "MYNAME"
- Argument name -> ami
- Argument value -> asdfg

### Provisioners 

Up to now we've just created infra with terraform but we've not customized.
Provisioners are used to execute scripts (local or remote) as part of resource creation or deletion.

example:
```
resource "aws_instance" "mytest" {
  ami = "lkadjflkd"



  provisioner "remote-exec" {
    inline = [
      "sudo amazon-linux-extras install nginx -y",
      "sudo another command"
    ]
    connection = {
      type = "ssh"
      host = self.public_ip
      user = "ec2_user"
      key = ${file(./privatekey.pem)}
    }
  }

  provisioner "local-exec" {
    command = "echo ${aws_instante.mytest.private_ip} >> private_ips.txt"
  }
}
```

There are mainly 2 types of provisioners:

- *local-exec*: executes something locally after the resource is created. A typical use-case is the execution on ansible playbooks run after the resource is created.
- *remote-exec*: allows us to execute things in the remote resource.
There are some other provisioners who can be checked in the documentation.

According to some other classification criteria:
- *Creation-Time provisioner* are only run during creation, NOT during update or any other lifecycle change. If the creation provisioner fails, the resource is marked as `tainted`. If the `when` param is not set, it's a creation provisioner.
- *Destroy-Time provisioner* Destroy provisioners are run before the resource is destroyed.

```
 provisioner "local-exec" {
   when = "destroy"
   command = "echo "destroying stuff"
 }
```

By default, provisioners that fail will also cause the terraform apply itself to fail.
The `on_failure` setting can be used to change this behaviour:

- *continue* ignore the error and continue with the creation/destruction.
- *fail* raise an error ans stop applying. This is the default behaviour. If it's a creation provisioner, taint the resource.

### Modules and workspaces

We can refer to different modules in order to keep the DRY principle. We have to install the modules with `terraform init`.

```
module "myec2" {
  source = "..\..\modules\whatever"
}
```

#### Variables and modules

It's required to define variables for the parameters that we want to change.

We can create a varibles.tf file in the module with default values, so we can set different values to some parameters

```
module "ec2module" {
  source = ""
  instance_type = "t2.micro"
}
```

#### Module sources

We can use diferent type of sources for terraform modules:
- *local modules* local relative path. It has to start with `.` or `..`.
```
module "consul" {
  source = "../consul"
}
```
- *git* they can start with `git::https://..` or `git::ssh://username@...` . You can use the ref=somethig to get the code from a branch or a tag. Terraform also supports `github.com` format

### Terraform registry

Repository of modules written by terraform community.
There are verified modules in terraform registry that are maintained by some third party vendors. They are reviewed by terraform (blue verification badge in the web).

The url is https://registry.terraform.io

### Terraform workspaces

Terraform allows us to have multiple workspaces, with each of the workspaces we can have different set of environment variables associated.

```
terraform workspace show  # shows current one
terraform workspace list  # shows available workspaces
terraform workspace new dev # creates dev workspace
terraform workspace select prd # select 

```

You can create a map with a default
the variable is terraform_workspace

we can use a lookup function

```
....
resource "aws_instance" "myinstance" {

  instance_type = lookup(var.instance_type, terraform.workspace)

}

variable "instance_type" {
  type = "map"

  default = {
    default = "t2.nano"
    dev = "t2.micro"
    prd = "t2.large"
  }
}

```


In workspaces terraform states are separate. There is a `terraform.tfstate.d` directory which has different subdirectories one per workspace.
For the default workspace the tfsate file is in the root directory.

### Team collaboration in terraform

#### Git

There are security challenges in committing tfstate into git. It's important NOT to store keys/passwords in git.

We can use ${file(file_path_ouside_the_repository)} BUT if you store the tfstate file into GIT the passwords are stored there in clear text.

Terraform and .gitignore: It's recommended to include these files:
- *.terraform directory*: this dir will be recreated when a `terraform init` is run.
- *terraform.tfvars* : likely to contain sensitive data like usernames / passwords, etc.
- *terraform.tfstate*: should be stoares in the remote side.
- *crash.log*: if terraform crashes there will be a file containing the details there. 

### Terraform remote backend

Is a feature that allows to store the tfstate file in a remote central repository (not in git). The typical backend is S3 in aws.

Supported backend types:
- *standard backend type*: supports state storage and locking.
- *enhanced backend type*: All features of standard + remote management.

```
terraform {
  backend "s3" {
    bucket = "mybucket"
    key = "path/to/my/key"
    region = "us-east-1"
    access_key = ""
    secret_key = ""

  }
}
```
Whenever you are performing a write operation, terraform would lock the state file. Just with S3 backend locking is NOT suported, we have to do it with DynamoDB, then we have to include the following param:

```
...
dynamodb_table = "whatever"
```

The dynamoDB table has to have a key called LockID



You can also use terraform cloud as remote backend. The operations are then executed in terraform cloud instead of the local environment.

```
terraform {
   backend "remote" {}
}

# and then in a backend.hcl
workspaces { name = "demo-repository" }
hostname = "app.terraform.io"
organization = "demo-whatever"
```

Then if we run a `terraform plan` it's actually run in terraform cloud. It will also show the output of the cost estimation and the sentinel policies that are applied.


#### Remote backend

The remote backend stores terraform state and may be used to run operations in terraform cloud.
When using full remote operations, plan or apply can be run in terraform cloud env streaming the output to the local terminal.

#### Local backend

The local backend stores state on the local filesystema, locks that state using system APIs and performs operations locally.

##### Bakend configuration

Backends are configured in the terraform section in terraform config files.
After configuring a backend, it has to be initialized.

When configuring a backend for the first time, terraform will provide the option to migrate the state file from the current backend to the new one. This lets you adopt a new backend without losing the existing state.

You don't need to specify every required argument in the backend configuration. Ommiting certain arguments may be desirable to avoid storing secrets within the main configuration. The remaining arguments must be provided as part of the initialization process:

```
terraform init \
   -backend-config="address=demo.consul.io" \
   -backend-config="path=example_app/terraform_state" \
   ...
```


### Security



#### Handling multiple AWS profiles with terraform providers

You can have more than one credential pair from aws CLI, selected with `--profile`.

You can add the `profile` name in the provider configuration.

#### Terraform with STS

Multiple AWS accounts, use a single identity and you can assume different roles.
 STS user -> assign a role with json policy. We have to give the user the assumeRole policy.

 With aws we do it like:
```
aws sts assume-role --role-arn arn:aws:iam::21389047182343:role/whatevet --role-session-name my_role_session

# then we get an accessKeyId, SecretAccessKey, SessionToken
```

The config for this, we have to set the configuration in the provider itself:
```
provider "aws" {
  region = "us-east-1"
  assume_role {
    role_arn = ""
    session_name = ""
  }
}
```

#### Sensitive parameter

For the outputs, we can set a param as `sensitive = true`

```
output "db_password" {
  value = local.db_password
  description = "description of the parameter"
  sensitive = true

}
```


### Terraform cloud

Manages terraform runs in a consistent and reliable environments with various features like access controls, private registry for sharing modules, policy controls (put some checks in the resources created) and others (comments, etc.).

Based on the terraform plan it provides a monthly/hourly cost estimated.

State files are stored there.
Variables are stored in terraform cloud.

Terraform cloud has a free tier up to 5 users.

### Sentinel

Sentinel is an embedded policy-as-code framework integrated with the Hashicorp enterprise products.

It enables fine-grained, logic-based policy decisions and can be extended to use info from external sources. It's a PAID feature.

```
terraform plan ---> sentinel checks ---> terraform apply
```

A policy is included in a policy-set and then the policy set is associated to the workspace.

This is the web with the documentation for [Sentinel](https://docs.hashicorp.com/sentinel/terraform/)

Example of rule:
```
    import "tfplan"
     
    main = rule {
      all tfplan.resources.aws_instance as _, instances {
        all instances as _, r {
          (length(r.applied.tags) else 0) > 0
        }
      }
    }
```



### Instruqt

It's the hashicorp training platform. [Here is a tutorial](https://instruqt.com/instruqt/tracks/getting-started-with-instruqt)


### Exams

57 questions

questions:
- True - False
- Multichoice
- Fill in the blank



##### Init

Terraform init is used to initializa a working directory containing Terraform configuration files.
During the init, the config is searched for module blocks and the source code for referenced modules is retrieved from the locations given in their source arguments.
Terraform must initialize the provider before it can be used.
Initalization downloads and installs the provider's plugin to that it can later be executed.
It will not create any sample files like example.tf.

##### Plan
Terraform plan command is used to create an execution plan.
It will NOT modify things in infrastructure.

Terraform performs a refresh, unless explicitly disabled, and then determines what actions are necessary to achieve the desired state specified in the configuration files.
This command is a convenient way to check whether the execution plan for a set of changes matches your expectations without making any changes to real resources or to the state.

##### Apply
It's used to apply the changes required to reach the desired state of the configuration.
Terraform apply will also write data to the terraform.tfstate fole.
Once apply is completed, the resources are immediately available.

##### Refresh

Command used to reconcile the state terraforms knows about with the real-world infrastructure.

This does NOT modify infrastructure but the state file.

##### Destroy

This command is used to destroy the terraform managed infra.

##### Format

The `terraform fmt` command is used to rewrite terraform configuration files to a canonical format and style. 

##### Validate

The `terraform validate` command validates the configuration files in a directory.

It runs checks that verify whether a configuration is syntactically valid and thus primarily useful for general verification of reusable modules, including the correctness of attribute names and value types.

It's safe to run this command automatically as a post-save check in a text editor.
Validation requires an initialized working directory with any referenced plugins and modules installed.

##### Provisioners

Provisioners can be used to model specific actions on the local machine or on a remote machine in order de prepare servers or other infrastructure objects for service.

Provisioners should only be used as las resort.

- Provisioner are included inside the resource block
- can be "local-exec" or "remote-exec".

##### Debugging in terraform

- TF_LOG environment variable can be set (TRACE,DEBUG,INFO,WARN...)
- TF_LOG_PATH can be used to store the login to a file.

##### Import

Terraform can import existing infra that was created by other mean and now it will be managed by terraform.
The current implementation can only import resources into the state. It does NOT generate configuration for it.

Because of this, prior to running terraform import, it's necessary to write a resource configuration block manually for the resource, to which the imported object will be mapped.

```
terraform import aws_instance.myec2 instance-id
```




##### Overview of data types
 - string: sequence of unicode characters representing some text.
 - list: sequential list of values identified by their position. Starting by 0.
 - map: group of values identified by named labels
 - number: integer

 ##### Workspaces
Each of the workspaces can have a different set of environment variables associated.

Workspaces allow multiple state files of a single configuration.

##### Modules

We can centralize the terraform resouces and call out from tf files whenever required.

Root and child
Every terraform config has at least one module known as root module, that consists of the resources defined in the tf files in the main working directory.

A module can call other modules, which lets you include the child's module's resources in the configuration in a concise way.

The resources defined in a module are encpasulated, so the calling module cannot access their attributes directly. The child module can define output values to export certain values to be accessed by the calling module.

The `sensitive` parameter can be used not to show the values directly in the std outputs. These value are still recorded in the tfstate file so they are visible to anyone able to access the tfstate info.

It's recommended to explicitly constraint the acceptable version number for each external module to avoid unexpected or unwanted changes. Version constraints are only supported for modules installed from a module registry.

Terraform registry is integrated directly into terraform.
The format to reference a module is `<NAMESPACE>/<NAME>/<PROVIDER>`
For private regitstry -> we have to prepend `<HOSTNAME>/`


##### Functions

Terraform language includes a number of built-in functions that you can use to transform and combine values.

User defined functions are not supported. Beware of the basic functions as lookup, max, min.

##### Count and count.index

Count alllows us to have a set of objects so you can modify the configuration for each instance

#####  Find the issue use case.

Not recommended practices.

##### Terraform lock

If supported by the backend, terraform will lock your state for all the operations that could write the state.
Terraform has a force-unlock command to manually unlock the state if unlocking failed.

`terraform force-unlock LOCK_ID [DIR]`

##### Resources deleted out of terraform

You have created an ec2 instance and someone manually modified the type from t2.micro to t2.large. If you do a terraform plan, terraform will see the current state is t2.large and the desired state is t2.micro. It will try to change the instance back to t2.micro.

##### Resource block

Each resource block describes one or more infrastructure object. They have a "resource type" and "local name".

##### Sentinel 

Sentinel is an embedded policy-as-code framework integrated with the hashicorp enterprise products.

- Verify if the EC2 instance has tags
- Verify a S3 bucket has encryption enabled

Sentinel is a proactive service

##### Sensitive data in state file

If you manage any sensitive data within Terraform, treat the state itself as sensitive data.
 
 Approaches:
 - Terraform cloud always encrypts the state at rest and protects with TLS in transit.
 - S3 backend supports encryption at rest when encrypt option is enabled.

 Hardcoding credentials into any terraform config is NOT recommended.

 You can store the credentials outside the terraform configuration.
 Storing credentials as part of environment variables is much better approach than hard coding it in the system.








##### Output

Terraform output is a command used to extract the value of an output variable from the state file.

##### Benefits of IaC tools
- Automation
- Versioning
- Reusability

##### Refresh

Terraform refresh does NOT modify the infrastructure but it modifies the state file.

##### Strings

Slice function is NOT part of the string function. Others like join, split, chomp are part of it.

##### Module version
It is NOT mandatory to include the module version argument when pulling the code from a terraform registry

##### Dynamic blocks

Overuse of dynamic blocks can make the configuration hard to read and maintain.

##### Apply
Terraform apply can change, destroy and provision resources, but it can NOT import any resource.

##### Terraform enterprise

Provides several added advantages compared to terraform cloud.

Examples:
- Single Sign On
- Auditing
- Private Data Center Networking
- Clustering
- Team and governance features.



##### Structural data types

A structural type allows multiple values of several distinct types to be grouped together as a single value.
A list contains multiple values of the same type.



##### Provisioner

The local-exec provisioner invokes a local executable after the resource is created. This invokes a process on the machine running Terraform, NOT on the resource.

The remote-exec provisioner invokes a script on a remote resource after it is created. It supports both ssh and winrm type connections.

```
resource "aws_instance" "myname" {

  provisioner "remote-exec" {
    inline = [
      "yum -y install nginx",
      "yum -y install nano"
    ]
  }
}
```

Failure behaviour. By default, provisioners that fail will also cause the terraform apply itself to fail.
The on_failure setting can be used to change this, the allowed values are `continue` ignore the error and `fail`. 

2 types of provisioner:
- Creation-time provisioner : are only run during the creation, not during updating or any other lifecycle. If creation-time provisioner fails, the resource is marked as tainted.
- Destroy-time provisioner: Are run BEFORE the resource is destroyed.






##### Required version

for terraform in terraform section.



https://www.youtube.com/watch?v=fOybhcbuxJ0



https://hashicorp.github.io/field-workshops-terraform/slides/aws/terraform-oss/#49


