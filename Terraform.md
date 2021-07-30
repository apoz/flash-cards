# Terraform

## Terraform Language

The main purpose of the terraform language is declaring **resources** that represent infrastructure objects.

A **terraform configuration** is a complete document in the terraform language that tells terraform how to manage a given collection of infrastructure.

The Terraform language is declarative, describing an intended goal rather than the steps to reach that goal. The ordering of blocks and the files they are organized into are generally not significant; Terraform only considers implicit and explicit relationships between resources when determining an order of operations.

Example:

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 1.0.4"
    }
  }
}

variable "aws_region" {}

variable "base_cidr_block" {
  description = "A /16 CIDR range definition, such as 10.1.0.0/16, that the VPC will use"
  default = "10.1.0.0/16"
}

variable "availability_zones" {
  description = "A list of availability zones in which to create subnets"
  type = list(string)
}

provider "aws" {
  region = var.aws_region
}

resource "aws_vpc" "main" {
  # Referencing the base_cidr_block variable allows the network address
  # to be changed without modifying the configuration.
  cidr_block = var.base_cidr_block
}

resource "aws_subnet" "az" {
  # Create one subnet for each given availability zone.
  count = length(var.availability_zones)

  # For each subnet, use one of the specified availability zones.
  availability_zone = var.availability_zones[count.index]

  # By referencing the aws_vpc.main object, Terraform knows that the subnet
  # must be created only after the VPC is created.
  vpc_id = aws_vpc.main.id

  # Built-in functions and operators can be used for simple transformations of
  # values, such as computing a subnet address. Here we create a /20 prefix for
  # each subnet, using consecutive addresses for each availability zone,
  # such as 10.1.16.0/20 .
  cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 4, count.index+1)
}

```

### Files and directories

Code in the Terraform language is stored in plain text files with the `.tf` file extension. There is also a JSON-based variant of the language that is named with the `.tf.json` file extension.

Files containing Terraform code are often called configuration files.

Configuration files must always use UTF-8 encoding, and by convention usually use Unix-style line endings (LF) rather than Windows-style line endings (CRLF), though both are accepted.

#### Directories and modules

A module is a collection of `.tf` and/or `.tf.json` files kept together in a directory.

A Terraform module only consists of the top-level configuration files in a directory; nested directories are treated as completely separate modules, and are not automatically included in the configuration.

Terraform evaluates all of the configuration files in a module, effectively treating the entire module as a single document. Separating various blocks into different files is purely for the convenience of readers and maintainers, and has no effect on the module's behavior.

A Terraform module can use module calls to explicitly include other modules into the configuration. These child modules can come from local directories (nested in the parent module's directory, or anywhere else on disk), or from external sources like the `Terraform Registry`.

#### The root module

Terraform always runs in the context of a single root module. A complete Terraform configuration consists of a root module and the tree of child modules (which includes the modules called by the root module, any modules called by those modules, etc.).

- In Terraform CLI, the root module is the working directory where Terraform is invoked. (You can use command line options to specify a root module outside the working directory, but in practice this is rare. )
- In Terraform Cloud and Terraform Enterprise, the root module for a workspace defaults to the top level of the configuration directory (supplied via version control repository or direct upload), but the workspace settings can specify a subdirectory to use instead.

#### Override files

Terraform normally loads all of the .tf and .tf.json files within a directory and expects each one to define a distinct set of configuration objects. If two files attempt to define the same object, Terraform returns an error.

In some rare cases, it is convenient to be able to override specific portions of an existing configuration object in a separate file. For example, a human-edited configuration file in the Terraform language native syntax could be partially overridden using a programmatically-generated file in JSON syntax.

For these rare situations, Terraform has special handling of any configuration file whose name ends in _override.tf or _override.tf.json. This special handling also applies to a file named literally override.tf or override.tf.json.

Terraform initially skips these override files when loading configuration, and then afterwards processes each one in turn (in lexicographical order). For each top-level block defined in an override file, Terraform attempts to find an already-defined object corresponding to that block and then merges the override block contents into the existing object.

Use override files only in special circumstances. Over-use of override files hurts readability, since a reader looking only at the original files cannot easily see that some portions of those files have been overridden without consulting all of the override files that are present. When using override files, use comments in the original files to warn future readers about which override files apply changes to each block.

### Syntax

#### Terminology
- **Argument**: an argument assigns a value to a particular name. The identifier before the equal is the *argument name* and the part after the equal the *argument value*. It can be also defined as *attribute* in general HCL terminology.
- **Block**: is a container for other content. A block has a *type* (for example resource). Each block type defines how many labels are expected after it. *Block body* is delimited by `{}`
- **Identifier**: identifiers can contain letters, digits, underscores(_) and hyphens(-). The first character can NOT be a digit.
- **Comment**: single line can be specified with `#`or `//`. Multiline are defined by `/* ... */`

#### tf.json syntax example
```json
{
  "resource": {
    "aws_instance": {
      "example": {
        "instance_type": "t2.micro",
        "ami": "ami-abc123"
      }
    }
  }
}
```
#### Style conventions
- Indent 2 spaces for each nesting values
- When multiple arguments appear in consecutive lines, align `=` signs.

### Resources

Resources are the most important element in Terraform language. Each resource block describes one or more infrastructure objects, such as virtual networks, compute instances, or higher-level components such as DNS records.

#### Resource block syntax

Example:
```
resource "aws_instance" "web" {
  ami           = "ami-a1b2c3d4"
  instance_type = "t2.micro"
}
```
A `resource` block declares a resource of a given type (**"aws_instance"**) with a given local name (**"web"**). The name is used to refer to this resource from elsewhere in the same Terraform module, but has no significance outside that module's scope. The name has to be unique within the module.

Within the block body (between { and }) are the configuration arguments for the resource itself. Most arguments in this section depend on the resource type.

#### Resource types

##### Providers

Each resource type is implemented by a **provider**, which is a plugin for Terraform that offers a collection of resource types.A provider usually provides resources to manage a single cloud or on-premises infrastructure platform. Providers are distributed separately from Terraform itself, but Terraform can automatically install most providers when initializing a working directory.

###### Requiring providers

Each Terraform module must declare which providers it requires, so that Terraform can install and use them. Provider requirements are declared in a required_providers block.

```
terraform {
  required_providers {
    mycloud = {
      source  = "mycorp/mycloud"
      version = "~> 1.0"
    }
  }
}
```
The required_providers block must be nested inside the top-level terraform block (which can also contain other settings).

Each argument in the required_providers block enables one provider. The key determines the provider's local name (its unique identifier within this module), and the value is an object with the following elements:
- *source* - the global source address for the provider you intend to use, such as hashicorp/aws. It's comprised of 3 parts `[<HOSTNAME>/]<NAMESPACE>/<TYPE>`. Hostname defaults to `registry.terraform.io`.
- *version* - a version constraint specifying which subset of available provider versions the module is compatible with.

This format was included in Terraform 0.13 version, 0.12v used the format `mycloud = "~=1.0"` and had no way to specify the source address.

###### Built-in providers

While most Terraform providers are distributed separately as plugins, there is currently one provider that is built in to Terraform itself, which provides the `terraform_remote_state` data source.

Because this provider is built in to Terraform, you don't need to declare it in the required_providers block in order to use its features. 

###### In-house providers

Some organizations develop their own providers to configure proprietary systems, and wish to use these providers from Terraform without publishing them on the public Terraform Registry.

One option for distributing such a provider is to run an in-house private registry, by implementing the provider registry protocol.

Terraform also supports other provider installation methods, including placing provider plugins directly in specific directories in the local filesystem, via filesystem mirrors.

For example, you can use:
```
terraform {
  required_providers {
    mycloud = {
      source  = "terraform.example.com/examplecorp/ourcloud"
      version = ">= 1.0"
    }
  }
}
```
And then, in one of the implied local mirror directories, create a structure like this:
```
terraform.example.com/examplecorp/ourcloud/1.0.0
```
Under that 1.0.0 directory, create one additional directory representing the platform where you are running Terraform, such as linux_amd64 for Linux on an AMD64/x64 processor, and then place the provider plugin executable and any other needed files in that directory.

###### Provider configuration

Provider configurations belong in the root module of a Terraform configuration. (Child modules receive their provider configurations from the root module.

A provider configuration is created with a `provider` block:
```
provider "google" {
  project = "acme-app"
  region  = "us-central1"
}
```

You can use expressions in the values of these configuration arguments, but can only reference values that are known before the configuration is applied. This means you can safely reference input variables, but not attributes exported by resources.

There are also two "meta-arguments" that are defined by Terraform itself and available for all provider blocks:
- **alias** for using the same provider with different configurations for different resources.
- **version** no longer recommended

Example of different provider configurations:
```
# The default provider configuration; resources that begin with `aws_` will use
# it as the default, and it can be referenced as `aws`.
provider "aws" {
  region = "us-east-1"
}

# Additional provider configuration for west coast region; resources can
# reference this as `aws.west`.
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
```

Then, if we need to reference that provider from some resource:
```
resource "aws_instance" "foo" {
  provider = aws.west

  # ...
}
```
To select alternate provider configurations for a child module, use its providers meta-argument to specify which provider configurations should be mapped to which local provider names inside the module:
```
module "aws_vpc" {
  source = "./aws_vpc"
  providers = {
    aws = aws.west
  }
}
```
#### Resource meta-arguments

The Terraform language defines several meta-arguments, which can be used with any resource type to change the behavior of resources.

##### depends_on

Use the depends_on meta-argument to handle hidden resource or module dependencies that Terraform can't automatically infer.
Explicitly specifying a dependency is only necessary when a resource or module relies on some other resource's behavior but doesn't access any of that resource's data in its arguments.

The depends_on meta-argument, if present, must be a list of references to other resources or child modules in the same calling module. 

The depends_on argument should be used only as a last resort.

##### count

By default, a resource block configures one real infrastructure object. However, sometimes you want to manage several similar objects (like a fixed pool of compute instances) without writing a separate block for each one. Terraform has two ways to do this: **count** and **for_each**.

If a resource or module block includes a count argument whose value is a whole number, Terraform will create that many instances.

The count meta-argument accepts a whole number, and creates that many instances of the resource or module. Each instance has a distinct infrastructure object associated with it, and each is separately created, updated, or destroyed when the configuration is applied.

```
resource "aws_instance" "server" {
  count = 4 # create four similar EC2 instances

  ami           = "ami-a1b2c3d4"
  instance_type = "t2.micro"

  tags = {
    Name = "Server ${count.index}"
  }
}
```

The count meta-argument accepts numeric expressions. However, unlike most arguments, the count value must be known before Terraform performs any remote resource actions.

If your instances are almost identical, count is appropriate. If some of their arguments need distinct values that can't be directly derived from an integer, it's safer to use for_each.

##### for_each

`for_each` is a meta-argument defined by the Terraform language. It can be used with modules and with every resource type.

The for_each meta-argument accepts a map or a set of strings, and creates an instance for each item in that map or set. Each instance has a distinct infrastructure object associated with it, and each is separately created, updated, or destroyed when the configuration is applied.

Example of map:
```
resource "azurerm_resource_group" "rg" {
  for_each = {
    a_group = "eastus"
    another_group = "westus2"
  }
  name     = each.key
  location = each.value
}
```

Example of set of strings:
```
resource "aws_iam_user" "the-accounts" {
  for_each = toset( ["Todd", "James", "Alice", "Dottie"] )
  name     = each.key
}
```

The `each` object has 2 attributes:
- **key**
- **value**


Sensitive values, such as sensitive input variables, sensitive outputs, or sensitive resource attributes cannot be used as arguments to for_each. 

Example:
```
variable "vpcs" {
  type = map(object({
    cidr_block = string
  }))
}

resource "aws_vpc" "example" {
  # One VPC for each element of var.vpcs
  for_each = var.vpcs

  # each.value here is a value from var.vpcs
  cidr_block = each.value.cidr_block
}

resource "aws_internet_gateway" "example" {
  # One Internet Gateway per VPC
  for_each = aws_vpc.example

  # each.value here is a full aws_vpc object
  vpc_id = each.value.id
}

output "vpc_ids" {
  value = {
    for k, v in aws_vpc.example : k => v.id
  }

  # The VPCs aren't fully functional until their
  # internet gateways are running.
  depends_on = [aws_internet_gateway.example]
}

```

##### provider

The provider meta-argument specifies which provider configuration to use for a resource, overriding Terraform's default behavior of selecting one based on the resource type name. Its value should be an unquoted <PROVIDER>.<ALIAS> reference.

```
# default configuration
provider "google" {
  region = "us-central1"
}

# alternate configuration, whose alias is "europe"
provider "google" {
  alias  = "europe"
  region = "europe-west1"
}

resource "google_compute_instance" "example" {
  # This "provider" meta-argument selects the google provider
  # configuration whose alias is "europe", rather than the
  # default configuration.
  provider = google.europe

  # ...
}

```

##### lifecycle

Example:
```
resource "azurerm_resource_group" "example" {
  # ...

  lifecycle {
    create_before_destroy = true
  }
}
```

Lifecycle is a nested block that can appear within a resource block. The lifecycle block and its contents are meta-arguments, available for all resource blocks regardless of type.

The following arguments can be used inside:
- *create_before_destroy* (bool) By default, when Terraform must change a resource argument that cannot be updated in-place due to remote API limitations, Terraform will instead destroy the existing object and then create a new replacement object with the new configured arguments.The create_before_destroy meta-argument changes this behavior so that the new replacement object is created first, and the prior object is destroyed after the replacement is created.
- *prevent_destroy* (bool) This meta-argument, when set to true, will cause Terraform to reject with an error any plan that would destroy the infrastructure object associated with the resource, as long as the argument remains present in the configuration.
- *ignore_changes* (list of attribute names) The ignore_changes feature is intended to be used when a resource is created with references to data that may change in the future, but should not affect said resource after its creation. In some rare cases, settings of a remote object are modified by processes outside of Terraform, which Terraform would then attempt to "fix" on the next run. Instead of a list, the special keyword all may be used to instruct Terraform to ignore all attributes, which means that Terraform can create and destroy the remote object but will never propose updates to it.
```
resource "aws_instance" "example" {
  # ...

  lifecycle {
    ignore_changes = [
      # Ignore changes to tags, e.g. because a management agent
      # updates these based on some ruleset managed elsewhere.
      tags,
    ]
  }
}
```
##### provisioner and connection

Provisioners can be used to model specific actions on the local machine or on a remote machine in order to prepare servers or other infrastructure objects for service. Check specific section.

##### Timeouts

Some resource types provide a special timeouts nested block argument that allows you to customize how long certain operations are allowed to take before being considered to have failed. For example, aws_db_instance allows configurable timeouts for create, update and delete operations.

```
resource "aws_db_instance" "example" {
  # ...

  timeouts {
    create = "60m"
    delete = "2h"
  }
}

```

### Data sources

Data sources allow Terraform use information defined outside of Terraform, defined by another separate Terraform configuration, or modified by functions.
A data source is accessed via a special kind of resource known as a data resource, declared using a data block:

```
data "aws_ami" "example" {
  most_recent = true

  owners = ["self"]
  tags = {
    Name   = "app-server"
    Tested = "true"
  }
}
```
If the query constraint arguments for a data resource refer only to constant values or values that are already known, the data resource will be read and its state updated during Terraform's "refresh" phase, which runs prior to creating a plan. This ensures that the retrieved data is available for use during planning and so Terraform's plan will show the actual values obtained.

Data resources support count and for_each meta-arguments as defined for managed resources, with the same syntax and behavior.
Data resources support the provider meta-argument as defined for managed resources, with the same syntax and behavior.
Data resources do not currently have any customization settings available for their lifecycle, but the lifecycle nested block is reserved in case any are added in future versions.

Each data instance will export one or more attributes, which can be used in other resources as reference expressions of the form data.<TYPE>.<NAME>.<ATTRIBUTE>. For example:

```
resource "aws_instance" "web" {
  ami           = data.aws_ami.web.id
  instance_type = "t1.micro"
}
```


#### local data sources

##### local_file
local_file reads a file from the local filesystem.

```
data "local_file" "foo" {
    filename = "${path.module}/foo.bar"
}
```


##### template_file

The template_file data source renders a template from a template string, which is usually loaded from an external file.
```
data "template_file" "init" {
  template = "${file("${path.module}/init.tpl")}"
  vars = {
    consul_address = "${aws_instance.consul.private_ip}"
  }
}
```
Although in principle template_file can be used with an inline template string, we don't recommend this approach because it requires awkward escaping. Instead, just use template syntax directly in the configuration. For example:
```
  user_data = <<-EOT
    echo "CONSUL_ADDRESS = ${aws_instance.consul.private_ip}" > /tmp/iplist
  EOT
```

The following attributes are exported:
- template - See Argument Reference above.
- vars - See Argument Reference above.
- rendered - The final rendered template.


### Providers

Terraform relies on plugins called "providers" to interact with cloud providers, SaaS providers, and other APIs.

Terraform configurations must declare which providers they require so that Terraform can install and use them. Additionally, some providers require configuration (like endpoint URLs or cloud regions) before they can be used.

Each provider adds a set of resource types and/or data sources that Terraform can manage.

#### Provider installation

- Terraform Cloud and Terraform Enterprise install providers as part of every run.
- Terraform CLI finds and installs providers when initializing a working directory. It can automatically download providers from a Terraform registry, or load them from a local mirror or cache. If you are using a persistent working directory, you must reinitialize whenever you change a configuration's providers. To save time and bandwidth, Terraform CLI supports an optional plugin cache. You can enable the cache using the plugin_cache_dir setting in the CLI configuration file.

The dependency lock file is a file that belongs to the configuration as a whole, rather than to each separate module in the configuration. For that reason Terraform creates it and expects to find it in your current working directory when you run Terraform, which is also the directory containing the .tf files for the root module of your configuration.

The lock file is always named `.terraform.lock.hcl`, and this name is intended to signify that it is a lock file for various items that Terraform caches in the .terraform subdirectory of your working directory.

Terraform automatically creates or updates the dependency lock file each time you run the terraform init command. You should include this file in your version control repository so that you can discuss potential changes to your external dependencies via code review.

When `terraform init` is working on installing all of the providers needed for a configuration, Terraform considers both the version constraints in the configuration and the version selections recorded in the lock file.

If a particular provider has no existing recorded selection, Terraform will select the newest available version that matches the given version constraint, and then update the lock file to include that selection.

If a particular provider already has a selection recorded in the lock file, Terraform will always re-select that version for installation, even if a newer version has become available. You can override that behavior by adding the -upgrade option when you run terraform init, in which case Terraform will disregard the existing selections and once again select the newest available version matching the version constraint.

#### Provider requirements

Each Terraform module must declare which providers it requires, so that Terraform can install and use them. Provider requirements are declared in a required_providers block.

A provider requirement consists of a local name, a source location, and a version constraint:
```
terraform {
  required_providers {
    mycloud = {
      source  = "mycorp/mycloud"
      version = "~> 1.0"
    }
  }
}
```

### Variables and Outputs

#### Input variables

Input variables serve as parameters for a Terraform module, allowing aspects of the module to be customized without altering the module's own source code, and allowing modules to be shared between different configurations.

When you declare variables in the root module of your configuration, you can set their values using CLI options and environment variables. When you declare them in child modules, the calling module should pass values in the module block.


```
variable "image_id" {
  type = string
}

variable "availability_zone_names" {
  type    = list(string)
  default = ["us-west-1a"]
}

variable "docker_ports" {
  type = list(object({
    internal = number
    external = number
    protocol = string
  }))
  default = [
    {
      internal = 8300
      external = 8300
      protocol = "tcp"
    }
  ]
}
```

Parameters:

- default - A default value which then makes the variable optional.
- type - This argument specifies what value types are accepted for the variable.
The type constructors allow you to specify complex types such as collections:
  - string
  - number
  - bool
  - list(<TYPE>)
  - set(<TYPE>)
  - map(<TYPE>)
  - object({<ATTR NAME> = <TYPE>, ... })
  - tuple([<TYPE>, ...])

- description - This specifies the input variable's documentation.
- validation - A block to define validation rules, usually in addition to type constraints.

```
variable "image_id" {
  type        = string
  description = "The id of the machine image (AMI) to use for the server."

  validation {
    condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "The image_id value must be a valid AMI id, starting with \"ami-\"."
  }
}
```
- sensitive - Limits Terraform UI output when the variable is used in configuration.

```
variable "user_information" {
  type = object({
    name    = string
    address = string
  })
  sensitive = true
}

resource "some_resource" "a" {
  name    = var.user_information.name
  address = var.user_information.address
}

```

Terraform loads variables in the following order, with later sources taking precedence over earlier ones:
- Environment variables
- The terraform.tfvars file, if present.
- The terraform.tfvars.json file, if present.
- Any *.auto.tfvars or *.auto.tfvars.json files, processed in lexical order of their filenames.
- Any -var and -var-file options on the command line, in the order they are provided. (This includes variables set by a Terraform Cloud workspace.)


#### Output variables

Output values are like the return values of a Terraform module, and have several uses:

- A child module can use outputs to expose a subset of its resource attributes to a parent module.
- A root module can use outputs to print certain values in the CLI output after running terraform apply.
- When using remote state, root module outputs can be accessed by other configurations via a terraform_remote_state data source.

```
output "instance_ip_addr" {
  value = aws_instance.server.private_ip
}
```

In a parent module, outputs of child modules are available in expressions as module.`<MODULE NAME>.<OUTPUT NAME>`. For example, if a child module named web_server declared an output named instance_ip_addr, you could access that value as module.web_server.instance_ip_addr.

#### local variables

```
locals {
  service_name = "forum"
  owner        = "Community Team"
}


# to use the local value
resource "aws_instance" "example" {
  # ...

  tags = local.common_tags
}


```

### Modules

Modules are containers for multiple resources that are used together. A module consists of a collection of .tf and/or .tf.json files kept together in a directory.

Modules are the main way to package and reuse resource configurations with Terraform.

Every Terraform configuration has at least one module, known as its root module, which consists of the resources defined in the .tf files in the main working directory.

A Terraform module (usually the root module of a configuration) can call other modules to include their resources into the configuration. A module that has been called by another module is often referred to as a child module.

Child modules can be called multiple times within the same configuration, and multiple configurations can use the same child module.

In addition to modules from the local filesystem, Terraform can load modules from a public or private registry. This makes it possible to publish modules for others to use, and to use modules that others have published.

#### Module blocks

To call a module means to include the contents of that module into the configuration with specific values for its input variables. Modules are called from within other modules using module blocks:

```
module "servers" {
  source = "./app-cluster"

  servers = 5
}
```

The label immediately after the module keyword is a local name, which the calling module can use to refer to this instance of the module.

##### Source

All modules require a source argument, which is a meta-argument defined by Terraform. Its value is either the path to a local directory containing the module's configuration files, or a remote module source that Terraform should download and use. This value must be a literal string with no template sequences; arbitrary expressions are not allowed.

The same source address can be specified in multiple module blocks to create multiple copies of the resources defined within, possibly with different variable values.

After adding, removing, or modifying module blocks, you must re-run terraform init to allow Terraform the opportunity to adjust the installed modules. 

##### Version

The version argument accepts a version constraint string. Terraform will use the newest installed version of the module that meets the constraint; if no acceptable versions are installed, it will download the newest version that meets the constraint.

##### Accessing module output values

The resources defined in a module are encapsulated, so the calling module cannot access their attributes directly. However, the child module can declare output values to selectively export certain values to be accessed by the calling module.

For example, if the ./app-cluster module referenced in the example above exported an output value named instance_ids then the calling module can reference that result using the expression module.servers.instance_ids:

```
resource "aws_elb" "example" {
  # ...

  instances = module.servers.instance_ids
}
```

When refactoring an existing configuration to split code into child modules, moving resource blocks between modules causes Terraform to see the new location as an entirely different resource from the old. Always check the execution plan after moving code across modules to ensure that no resources are deleted by surprise.

If you want to make sure an existing resource is preserved, use the terraform state mv command to inform Terraform that it has moved to a different module.

When passing resource addresses to terraform state mv, resources within child modules must be prefixed with module.<MODULE NAME>.. If a module was called with count or for_each, its resource addresses must be prefixed with module.<MODULE NAME>[<INDEX>]. instead, where <INDEX> matches the count.index or each.key value of a particular module instance.

The taint command can be used to taint specific resources within a module:
```
terraform taint module.salt_master.aws_instance.salt_master
```
It is not possible to taint an entire module. Instead, each resource within the module must be tainted separately.

#### Module sources

The source argument in a module block tells Terraform where to find the source code for the desired child module.

Terraform uses this during the module installation step of terraform init to download the source code to a directory on local disk so that it can be used by other Terraform commands.

Source types supported:
- local paths: specified starting by `./` or `../`
- terraform registry: format `<NAMESPACE>/<NAME>/<PROVIDER>`. Specify `<HOSTNAME>/` if the registry is NOT `registry.terraform.io`
- github repos: `github.com/...` or `git@github.com:` (for ssh cloning)
- bitbucket repos: `bitbucket.org/...`
- git repo: `git::https://whatever.com/vpc.git` or `git::ssh://username@...`  `?ref=LOQUESEA` for a tag or branch.
  scp-like address: `git::username@example.com:storage.git`
- mercurial repo : `hg::http://...`
- generic http server: terraform will make a GET with `terraform-get=1` to the URL and then look for the module in `X-Terraform-Get` header or in a `meta-element` with the name `terraform-get`.
If the URL has a common file extension for modules(["zip","tar.bz2","tar.gz","tar.xz"]), it will download directly the file and use it as source code.
- s3 bucket: `s3::https://...` it will look for aws credentiasl in `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` env variables, or `.aws/credentials` in your home dir.
- cgs bucket: `cgs://`

When the source of a module is a version control repository or archive file (generically, a "package"), the module itself may be in a sub-directory relative to the root of the package.

A special double-slash syntax is interpreted by Terraform to indicate that the remaining path after that point is a sub-directory within the package. For example:

```
 hashicorp/consul/aws//modules/consul-cluster 
```

#### meta-arguments

##### provider

In a module call block, the optional providers meta-argument specifies which provider configurations from the parent module will be available inside the child module.

```
# The default "aws" configuration is used for AWS resources in the root
# module where no explicit provider instance is selected.
provider "aws" {
  region = "us-west-1"
}

# An alternate configuration is also defined for a different
# region, using the alias "usw2".
provider "aws" {
  alias  = "usw2"
  region = "us-west-2"
}

# An example child module is instantiated with the alternate configuration,
# so any AWS resources it defines will use the us-west-2 region.
module "example" {
  source    = "./example"
  providers = {
    aws = aws.usw2
  }
}
```

If the child module does not declare any configuration aliases, the providers argument is optional. If you omit it, a child module inherits all of the default provider configurations from its parent module. (Default provider configurations are ones that don't use the alias argument.)

If you specify a providers argument, it cancels this default behavior, and the child module will only have access to the provider configurations you specify.

The value of providers is a map, where:
- The keys are the provider configuration names used inside the child module.
- The values are provider configuration names from the parent module.

There are two main reasons to use the providers argument:
- Using different default provider configurations for a child module.
- Configuring a module that requires multiple configurations of the same provider.

##### depends_on

Use the depends_on meta-argument to handle hidden resource or module dependencies that Terraform can't automatically infer.

##### count

count is a meta-argument defined by the Terraform language. It can be used with modules and with every resource type.

The count meta-argument accepts numeric expressions. However, unlike most arguments, the count value must be known before Terraform performs any remote resource actions. This means count can't refer to any resource attributes that aren't known until after a configuration is applied (such as a unique ID generated by the remote API when an object is created).

##### for_each

The for_each meta-argument accepts a map or a set of strings, and creates an instance for each item in that map or set. Each instance has a distinct infrastructure object associated with it, and each is separately created, updated, or destroyed when the configuration is applied.

The keys of the map (or all the values in the case of a set of strings) must be known values, or you will get an error message that for_each has dependencies that cannot be determined before apply, and a -target may be needed.

for_each keys cannot be the result (or rely on the result of) of impure functions, including uuid, bcrypt, or timestamp, as their evaluation is deferred during the main evaluation step.

The for_each meta-argument accepts map or set expressions. However, unlike most arguments, the for_each value must be known before Terraform performs any remote resource actions. This means for_each can't refer to any resource attributes that aren't known until after a configuration is applied (such as a unique ID generated by the remote API when an object is created).

[flattening structs](https://www.terraform.io/docs/language/functions/flatten.html#flattening-nested-structures-for-for_each)

#### module development

##### standard module structure

- Root module. This is the only required element for the standard module structure. Terraform files must exist in the root directory of the repository. This should be the primary entrypoint for the module and is expected to be opinionated.
- README. The root module and any nested modules should have README files. 
- LICENSE. The license under which this module is available.
- `main.tf`, `variables.tf`, `outputs.tf`. These are the recommended filenames for a minimal module, even if they're empty. main.tf should be the primary entrypoint. For a complex module, resource creation may be split into multiple files but any nested module calls should be in the main file. variables.tf and outputs.tf should contain the declarations for variables and outputs, respectively.
- Variables and outputs should have descriptions. All variables and outputs should have one or two sentence descriptions that explain their purpose.
-  Nested modules. Nested modules should exist under the modules/ subdirectory. Any nested module with a README.md is considered usable by an external user. If a README doesn't exist, it is considered for internal use only. 

##### providers inside modules

Alternative configurations can be used with the `configuration_aliases` param:
```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 2.7.0"
      configuration_aliases = [ aws.alternate ]
    }
  }
}

```
### Expressions

#### Types and values

The Terraform language uses the following types for its values:

- string: a sequence of Unicode characters representing some text, like "hello".
- number: a numeric value. The number type can represent both whole numbers like 15 and fractional values like 6.283185.
- bool: a boolean value, either true or false. bool values can be used in conditional logic.
- list (or tuple): a sequence of values, like ["us-west-1a", "us-west-1c"]. Elements in a list or tuple are identified by consecutive whole numbers, starting with zero.
- map (or object): a group of values identified by named labels, like {name = "Mabel", age = 52}.

Strings, numbers, and bools are sometimes called primitive types. Lists/tuples and maps/objects are sometimes called complex types, structural types, or collection types.

- null: a value that represents absence or omission. If you set an argument of a resource or module to null, Terraform behaves as though you had completely omitted it

##### Literal expresions

- **Lists/Tuples**
Lists/tuples are represented by a pair of square brackets containing a comma-separated sequence of values, like ["a", 15, true].

List literals can be split into multiple lines for readability, but always require a comma between values. A comma after the final value is allowed, but not required. Values in a list can be arbitrary expressions.

- **Maps/Objects**
Maps/objects are represented by a pair of curly braces containing a series of <KEY> = <VALUE> pairs:
```
{
  name = "John"
  age  = 52
}
```
Key/value pairs can be separated by either a comma or a line break.

The values in a map can be arbitrary expressions.

The keys in a map must be strings; they can be left unquoted if they are a valid identifier, but must be quoted otherwise. You can use a non-literal string expression as a key by wrapping it in parentheses, like (var.business_unit_tag_name) = "SRE".

Where possible, Terraform automatically converts values from one type to another in order to produce the expected type. If this isn't possible, Terraform will produce a type mismatch error and you must update the configuration with a more suitable expression.

Terraform automatically converts number and bool values to strings when needed. It also converts strings to numbers or bools, as long as the string contains a valid representation of a number or bool value.


##### Strings and Templates
Sequence 	Replacement
$${ 	Literal ${, without beginning an interpolation sequence.
%%{ 	Literal %{, without beginning a template directive sequence.

**Heredoc** strings

```
<<EOT
hello
world
EOT
```
To improve on this, Terraform also accepts an indented heredoc string variant that is introduced by the <<- sequence, that removes extra indentation.

Don't use "heredoc" strings to generate JSON or YAML. Instead, use the jsonencode function or the yamlencode function so that Terraform can be responsible for guaranteeing valid JSON or YAML syntax.
```
  example = jsonencode({
    a = 1
    b = "hello"
  })
```

- **Interpolation**
A ${ ... } sequence is an interpolation, which evaluates the expression given between the markers, converts the result to a string if necessary, and then inserts it into the final string:
```
"Hello, ${var.name}!"
```
In the above example, the named object var.name is accessed and its value inserted into the string, producing a result like "Hello, Juan!".

- **Directives**
A %{ ... } sequence is a directive, which allows for conditional results and iteration over collections, similar to conditional and for expressions.

The following directives are supported:

- The %{if <BOOL>}/%{else}/%{endif} directive chooses between two templates based on the value of a bool expression:
```
 "Hello, %{ if var.name != "" }${var.name}%{ else }unnamed%{ endif }!"
```
The else portion may be omitted, in which case the result is an empty string if the condition expression returns false.

- The %{for <NAME> in <COLLECTION>} / %{endfor} directive iterates over the elements of a given collection or structural value and evaluates a given template once for each element, concatenating the results together:
```
    <<EOT
    %{ for ip in aws_instance.example.*.private_ip }
    server ${ip}
    %{ endfor }
    EOT
```

The name given immediately after the for keyword is used as a temporary variable name which can then be referenced from the nested template.

To allow template directives to be formatted for readability without adding unwanted spaces and newlines to the result, all template sequences can include optional strip markers (~), immediately after the opening characters or immediately before the end. When a strip marker is present, the template sequence consumes all of the literal whitespace (spaces and newlines) either before the sequence (if the marker appears at the beginning) or after (if the marker appears at the end):

```
<<EOT
%{ for ip in aws_instance.example.*.private_ip ~}
server ${ip}
%{ endfor ~}
EOT
```

##### References to named values

The main kinds of named values available in Terraform are:

- **Resources**

`<RESOURCE TYPE>.<NAME>` represents a managed resource of the given type and name.


- If the resource doesn't use count or for_each, the reference's value is an object. The resource's attributes are elements of the object, and you can access them using dot or square bracket notation.
- If the resource has the count argument set, the reference's value is a list of objects representing its instances.
- If the resource has the for_each argument set, the reference's value is a map of objects representing its instances.


- **Input variables**


`var.<NAME>` is the value of the input variable of the given name.

- **Local values**

`local.<NAME>` is the value of the local value of the given name.`

Local values can refer to other local values, even within the same locals block, as long as you don't introduce circular dependencies.

- **Child module outputs**

`module.<MODULE NAME>` is an value representing the results of a module block.

If the corresponding module block does not have either count nor for_each set then the value will be an object with one attribute for each output value defined in the child module. To access one of the module's output values, use `module.<MODULE NAME>.<OUTPUT NAME>`.

If the corresponding module uses for_each then the value will be a map of objects whose keys correspond with the keys in the for_each expression, and whose values are each objects with one attribute for each output value defined in the child module, each representing one module instance.

If the corresponding module uses count then the result is similar to for for_each except that the value is a list with the requested number of elements, each one representing one module instance.

- **Data sources**
`data.<DATA TYPE>.<NAME>` is an object representing a data resource of the given data source type and name.
If the resource has the count argument set, the value is a list of objects representing its instances. If the resource has the for_each argument set, the value is a map of objects representing its instances.

- **Filesystem and workspace info**

  - *path.module* is the filesystem path of the module where the expression is placed.
  - *path.root* is the filesystem path of the root module of the configuration.
  - *path.cwd* is the filesystem path of the current working directory. In normal use of Terraform this is the same as path.root, but some advanced uses of Terraform run it from a directory other than the root module directory, causing these paths to be different.
  - *terraform.workspace* is the name of the currently selected workspace. 



Use the values in this section carefully, because they include information about the context in which a configuration is being applied and so may inadvertently hurt the portability or composability of a module.

For example, if you use path.cwd directly to populate a path into a resource argument then later applying the same configuration from a different directory or on a different computer with a different directory structure will cause the provider to consider the change of path to be a change to be applied, even if the path still refers to the same file.

- **Block-local values**

Within the bodies of certain blocks, or in some other specific contexts, there are other named values available beyond the global values listed above. These local names are described in the documentation for the specific contexts where they appear. Some of most common local names are:

- count.index, in resources that use the count meta-argument.
- each.key / each.value, in resources that use the for_each meta-argument.
- self, in provisioner and connection blocks.


##### Operators

When multiple operators are used together in an expression, they are evaluated in the following order of operations:

1. `!, - (multiplication by -1)`
2. `*, /, %`
3. `+, - (subtraction)`
4. `>, >=, <, <=`
5. `==, !=`
6. `&&`
7. `||`

The arithmetic operators all expect number values and produce number values as results:

-  a + b returns the result of adding a and b together.
-  a - b returns the result of subtracting b from a.
-  a * b returns the result of multiplying a and b.
-  a / b returns the result of dividing a by b.
-  a % b returns the remainder of dividing a by b. This operator is generally useful only when used with whole numbers.
-  -a returns the result of multiplying a by -1.

Terraform supports some other less-common numeric operations as functions. For example, you can calculate exponents using the pow function.

The equality operators both take two values of any type and produce boolean values as results.

- a == b returns true if a and b both have the same type and the same value, or false otherwise.
- a != b is the opposite of a == b.

The comparison operators all expect number values and produce boolean values as results.

- a < b returns true if a is less than b, or false otherwise.
- a <= b returns true if a is less than or equal to b, or false otherwise.
- a > b returns true if a is greater than b, or false otherwise.
- a >= b returns true if a is greater than or equal to b, or false otherwise.

The logical operators all expect bool values and produce bool values as results.

- a || b returns true if either a or b is true, or false if both are false.
- a && b returns true if both a and b are true, or false if either one is false.
- !a returns true if a is false, and false if a is true.

Terraform does not have an operator for the "exclusive OR" operation. If you know that both operators are boolean values then exclusive OR is equivalent to the != ("not equal") operator.



##### Function calls

The Terraform language has a number of built-in functions that can be used in expressions to transform and combine values. These are similar to the operators but all follow a common syntax:

```
<FUNCTION NAME>(<ARGUMENT 1>, <ARGUMENT 2>)
```

f the arguments to pass to a function are available in a list or tuple value, that value can be expanded into separate arguments. Provide the list value as an argument and follow it with the ... symbol:

```
min([55, 2453, 2]...)
```

When using sensitive data, such as an input variable or an output defined as sensitive as function arguments, the result of the function call will be marked as sensitive.

```
> local.baz
{
  "a" = (sensitive)
  "b" = "dog"
}
> keys(local.baz)
(sensitive)

```
###### Special functions

A small subset of functions interact with outside state and so for those it can be helpful to know when Terraform will call them in relation to other events that occur in a Terraform run.

- **file** or **templatefile** functions are intended for reading files that are included as a static part of the configuration and so Terraform will execute these functions as part of initial configuration validation, before taking any other actions with the configuration. That means you cannot use either function to read files that your configuration might generate dynamically on disk as part of the plan or apply steps.
- **timestamp** functions return a representation of the current system type at the point where Terraform calls it.
- **uuid** function returns a random result which differs on each call. W

Without any special behavior these would would both cause the final configuration during the apply step not to match the actions shown in the plan, which violates the Terraform execution model.

For that reason, Terraform arranges for both of those functions to produce unknown value results during the plan step, with the real result being decided only during the apply step. 

##### Conditional expressions

A conditional expression uses the value of a bool expression to select one of two values.

```
condition ? true_val : false_val
```

The two result values may be of any type, but they must both be of the same type so that Terraform can determine what type the whole conditional expression will return without knowing the condition value.


##### For expressions

A for expression creates a complex type value by transforming another complex type value. Each element in the input value can correspond to either one or zero values in the result, and an arbitrary expression can be used to transform each input element into an output element.

```
[for s in var.list : upper(s)]
```

A for expression's input (given after the in keyword) can be a list, a set, a tuple, a map, or an object.

```
[for k, v in var.map : length(k) + length(v)]
# or with a list
[for i, v in var.list : "${i} is ${v}"]
```

The type of brackets around the for expression decide what type of result it produces.

The above example uses [ and ], which produces a tuple. If you use { and } instead, the result is an object and you must provide two result expressions that are separated by the => symbol:


```
{for s in var.list : s => upper(s)}

#
{
  foo = "FOO"
  bar = "BAR"
  baz = "BAZ"
}

```
A for expression alone can only produce either an object value or a tuple value, but Terraform's automatic type conversion rules mean that you can typically use the results in locations where lists, maps, and sets are expected.

A for expression can also include an optional if clause to filter elements from the source collection, producing a value with fewer elements than the source value:

```
[for s in var.list : upper(s) if s != ""]
```

If the result type is an object (using { and } delimiters) then normally the given key expression must be unique across all elements in the result, or Terraform will return an error.

Sometimes the resulting keys are not unique, and so to support that situation Terraform supports a special grouping mode which changes the result to support multiple elements per key.

To activate grouping mode, add the symbol ... after the value expression. For example:

```
variable "users" {
  type = map(object({
    role = string
  }))
}

locals {
  users_by_role = {
    for name, user in var.users : user.role => name...
  }
}
```

```
{
  "admin": [
    "ps",
  ],
  "maintainer": [
    "am",
    "jb",
    "kl",
    "ma",
  ],
  "viewer": [
    "st",
    "zq",
  ],
}
```

##### Splat expressions

A splat expression provides a more concise way to express a common operation that could otherwise be performed with a for expression.


```
[for o in var.list : o.id]

# = 
var.list[*].id
```

The splat expression patterns shown above apply only to lists, sets, and tuples. To get a similar result with a map or object value you must use for expressions.

Resources that use the for_each argument will appear in expressions as a map of objects, so you can't use splat expressions with those resources. 



##### Dynamic blocks

A dynamic block acts much like a for expression, but produces nested blocks instead of a complex typed value. It iterates over a given complex value, and generates a nested block for each element of that complex value.

- The label of the dynamic block ("setting" in the example above) specifies what kind of nested block to generate.
- The for_each argument provides the complex value to iterate over.
- The iterator argument (optional) sets the name of a temporary variable that represents the current element of the complex value. If omitted, the name of the variable defaults to the label of the dynamic block ("setting" in the example above).
- The labels argument (optional) is a list of strings that specifies the block labels, in order, to use for each generated block. You can use the temporary iterator variable in this value.
- The nested content block defines the body of each generated block. You can use the temporary iterator variable inside this block.

ince the for_each argument accepts any collection or structural value, you can use a for expression or splat expression to transform an existing collection.

The iterator object (setting in the example above) has two attributes:

- key is the map key or list element index for the current element. If the for_each expression produces a set value then key is identical to value and should not be used.
- value is the value of the current element.

A dynamic block can only generate arguments that belong to the resource type, data source, provider or provisioner being configured. It is not possible to generate meta-argument blocks such as lifecycle and provisioner blocks, since Terraform must process these before it is safe to evaluate expressions.

##### Type constraints

Terraform module authors and provider developers can use detailed type constraints to validate user-provided values for their input variables and resource arguments. 

###### Primitive types
- **string**: a sequence of unicode characters representing some text.
- **number**: a numeric value.
- **bool**: true or false

The Terraform language will automatically convert number and bool values to string values when needed, and vice-versa as long as the string contains a valid representation of a number or boolean value.

###### Complex types

- **Collection types**
A collection type allows multiple values of one other type to be grouped together as a single value. 

The three kinds of collection type in the Terraform language are:

  - list(...): a sequence of values identified by consecutive whole numbers starting with zero. The keyword list is a shorthand for list(any), which accepts any element type as long as every element is the same type. This is for compatibility with older configurations; for new code, we recommend using the full form.

  - map(...): a collection of values where each is identified by a string label. The keyword map is a shorthand for map(any), which accepts any element type as long as every element is the same type. This is for compatibility with older configurations; for new code, we recommend using the full form.

    Maps can be made with braces ({}) and colons (:) or equals signs (=): { "foo": "bar", "bar": "baz" } OR { foo = "bar", bar = "baz" }. Quotes may be omitted on keys, unless the key starts with a number, in which case quotes are required. Commas are required between key/value pairs for single line maps. A newline between key/value pairs is sufficient in multi-line maps.

    Note: although colons are valid delimiters between keys and values, they are currently ignored by terraform fmt (whereas terraform fmt will attempt vertically align equals signs).

  - set(...): a collection of unique values that do not have any secondary identifiers or ordering.

- **Structural types**

  - object(...): a collection of named attributes that each have their own type.

    The schema for object types is { <KEY> = <TYPE>, <KEY> = <TYPE>, ... } — a pair of curly braces containing a comma-separated series of <KEY> = <TYPE> pairs. Values that match the object type must contain all of the specified keys, and the value for each key must match its specified type. (Values with additional keys can still match an object type, but the extra attributes are discarded during type conversion.)

  - tuple(...): a sequence of elements identified by consecutive whole numbers starting with zero, where each element has its own type.

    The schema for tuple types is [<TYPE>, <TYPE>, ...] — a pair of square brackets containing a comma-separated series of types. Values that match the tuple type must have exactly the same number of elements (no more and no fewer), and the value in each position must match the specified type for that position.



##### Version constraints

- = (or no operator): Allows only one exact version number. Cannot be combined with other conditions.

- !=: Excludes an exact version number.

- >, >=, <, <=: Comparisons against a specified version, allowing versions for which the comparison is true. "Greater-than" requests newer versions, and "less-than" requests older versions.

- ~>: Allows only the rightmost version component to increment. For example, to allow new patch releases within a specific minor release, use the full version number: ~> 1.0.4 will allow installation of 1.0.5 and 1.0.10 but not 1.1.0. This is usually called the pessimistic constraint operator.




### Functions

#### Numeric functions

- **abs**:
- **ceil**:
- **floor**:
- **log**:
- **max**:
- **min**:
- **parseint**:
- **pow**:
- **signum**:

#### String functions

- **chomp**:
- **format**:  `format("Hello, %s!", "Ander")`
- **formatlist**:  `format("Hello, %s!", ["Ander","Lisa", "Bart"])`
- **indent**: indent adds a given number of spaces to the beginnings of all but the first line in a given multi-line string.
- **join**: `join(separator, list)`
- **lower**: lower converts all cased letters in the given string to lowercase.
- **regex**:
- **regexall**:
- **replace**:
- **split**:
- **strrev**: strrev reverses the characters in a string.
- **substr**: substr extracts a substring from a given string by offset and length.
- **title**: title converts the first letter of each word in the given string to uppercase.
- **trim**: trim removes the specified characters from the start and end of the given string.
- **trimprefix**:
- **trimsuffix**:
- **trimspace**:
- **upper**:

#### Filesystem functions

- **abspath** takes a string containing a filesystem path and converts it to an absolute path. That is, if the path is not absolute, it will be joined with the current working directory.
- **file** reads the contents of a file at the given path and returns them as a string.
- **fileexists**
- **templatefile**  reads the file at the given path and renders its content as a template using a supplied set of template variables.




### Terraform settings

Terraform settings are gathered together into terraform blocks:

´´´
terraform {
  # ...
}
´´´

- **required_version** specifies which versions of Terraform can be used with your configuration.
- **required_providers** specifies all of the providers required by the current module mapping each local provider name ot a source address and a version constraint.

```
terraform {
  required_providers {
    aws = {
      version = ">= 2.7.0"
      source = "hashicorp/aws"
    }
  }
}
```

- **experiments** The Terraform team will sometimes introduce new language features initially via an opt-in experiment, so that the community can try the new feature and give feedback on it prior to it becoming a backward-compatibility constraint.

- **backend** Backend configuration.

```
terraform {
  backend "remote" {
    organization = "example_corp"

    workspaces {
      name = "my-app-prod"
    }
  }
}

```

#### Terraform backends

There are some important limitations on backend configuration:
- A configuration can only provide one backend block.
- A backend block cannot refer to named values (like input variables, locals, or data source attributes).

Whenever a configuration's backend changes, you must run terraform init again to validate and configure the backend before you can perform any plans, applies, or state operations.

Terraform's backends are divided into two main types, according to how they handle state and operations:

- Enhanced backends can both store state and perform operations. There are only two enhanced backends: local and remote.
- Standard backends only store state, and rely on the local backend for performing operations.


##### Default backend

If a configuration includes no backend block, Terraform defaults to using the local backend, which performs operations on the local system and stores state as a plain file in the current working directory.

##### Partial configuration

You do not need to specify every required argument in the backend configuration. Omitting certain arguments may be desirable if some arguments are provided automatically by an automation script running Terraform. 


- *File*: A configuration file may be specified via the init command line. To specify a file, use the -backend-config=PATH option when running terraform init. If the file contains secrets it may be kept in a secure data store, such as Vault, in which case it must be downloaded to the local disk before running Terraform.
- *Command-line key/value pairs*: Key/value pairs can be specified via the init command line. Note that many shells retain command-line flags in a history file, so this isn't recommended for secrets. To specify a single key/value pair, use the -backend-config="KEY=VALUE" option when running terraform init.
- *Interactively*: Terraform will interactively ask you for the required values, unless interactive input is disabled. Terraform will not prompt for optional values.


##### Configuration changes

You can change your backend configuration at any time. You can change both the configuration itself as well as the type of backend (for example from "consul" to "s3").

Terraform will automatically detect any changes in your configuration and request a reinitialization. As part of the reinitialization process, Terraform will ask if you'd like to migrate your existing state to the new configuration. This allows you to easily switch from one backend to another.

If you're using multiple workspaces, Terraform can copy all workspaces to the destination. If Terraform detects you have multiple workspaces, it will ask if this is what you want to do.



#### Enhanced backends: local

Example config
```
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

Data source configuration
```
data "terraform_remote_state" "foo" {
  backend = "local"

  config = {
    path = "${path.module}/../../terraform.tfstate"
  }
}
```

#### Enhanced backends: remote


The remote backend stores Terraform state and may be used to run operations in Terraform Cloud.

When using full remote operations, operations like terraform plan or terraform apply can be executed in Terraform Cloud's run environment, with log output streaming to the local terminal. Remote plans and applies use variable values from the associated Terraform Cloud workspace.

Terraform Cloud can also be used with local operations, in which case only state is stored in the Terraform Cloud backend.

```
# Using a single workspace:
terraform {
  backend "remote" {
    hostname = "app.terraform.io"
    organization = "company"

    workspaces {
      name = "my-app-prod"
    }
  }
}

# Using multiple workspaces:
terraform {
  backend "remote" {
    hostname = "app.terraform.io"
    organization = "company"

    workspaces {
      prefix = "my-app-"
    }
  }
}

```

Datasource
```
data "terraform_remote_state" "foo" {
  backend = "remote"

  config = {
    organization = "company"

    workspaces = {
      name = "workspace"
    }
  }
}
```

#### State

Terraform must store state about your managed infrastructure and configuration. This state is used by Terraform to map real world resources to your configuration, keep track of metadata, and to improve performance for large infrastructures.

This state is stored by default in a local file named "terraform.tfstate", but it can also be stored remotely, which works better in a team environment.

State is a necessary requirement for Terraform to function.
Alongside the mappings between resources and remote objects, Terraform must also track metadata such as resource dependencies.

Terraform typically uses the configuration to determine dependency order. However, when you delete a resource from a Terraform configuration, Terraform must know how to delete that resource. Terraform can see that a mapping exists for a resource not in your configuration and plan to destroy. However, since the configuration no longer exists, the order cannot be determined from the configuration alone.

To ensure correct operation, Terraform retains a copy of the most recent set of dependencies within the state. Now Terraform can still determine the correct order for destruction from the state when you delete one or more items from the configuration.

In addition to basic mapping, Terraform stores a cache of the attribute values for all resources in the state. This is the most optional feature of Terraform state and is done only as a performance improvement.

For larger infrastructures, querying every resource is too slow. Many cloud providers do not provide APIs to query multiple resources at once, and the round trip time for each resource is hundreds of milliseconds. On top of this, cloud providers almost always have API rate limiting so Terraform can only request a certain number of resources in a period of time. Larger users of Terraform make heavy use of the -refresh=false flag as well as the -target flag in order to work around this. In these scenarios, the cached state is treated as the record of truth.


##### Remote State datasource

The terraform_remote_state data source retrieves the root module output values from some other Terraform configuration, using the latest state snapshot from the remote backend.
This data source is built into Terraform, and is always available; you do not need to require or configure a provider in order to use it.

Sharing data with root module outputs is convenient, but it has drawbacks. Although terraform_remote_state only exposes output values, its user must have access to the entire state snapshot, which often includes some sensitive information.

When possible, we recommend explicitly publishing data for external consumption to a separate location instead of accessing it via remote state. This lets you apply different access controls for shared information and state snapshots.

A key advantage of using a separate explicit configuration store instead of terraform_remote_state is that the data can potentially also be read by systems other than Terraform, such as configuration management or scheduler systems within your compute instances. For that reason, we recommend selecting a configuration store that your other infrastructure could potentially make use of. For example:

- If you wish to share IP addresses and hostnames, you could publish them as normal DNS A, AAAA, CNAME, and SRV records in a private DNS zone and then configure your other infrastructure to refer to that zone so you can find infrastructure objects via your system's built-in DNS resolver.
- If you use HashiCorp Consul then publishing data to the Consul key/value store or Consul service catalog can make that data also accessible via Consul Template or the HashiCorp Nomad template stanza.
- If you use Kubernetes then you can make Config Maps available to your Pods.

Argument reference:
- **backend**
- **workspace**
- **config**

Attributes reference:
- (v0.12+) outputs - An object containing every root-level output in the remote state.
- (<= v0.11) <OUTPUT NAME> - Each root-level output in the remote state appears as a top level attribute on the data source.

```
data "terraform_remote_state" "vpc" {
  backend = "remote"

  config = {
    organization = "hashicorp"
    workspaces = {
      name = "vpc-prod"
    }
  }
}

# Terraform >= 0.12
resource "aws_instance" "foo" {
  # ...
  subnet_id = data.terraform_remote_state.vpc.outputs.subnet_id
}

# Terraform <= 0.11
resource "aws_instance" "foo" {
  # ...
  subnet_id = "${data.terraform_remote_state.vpc.subnet_id}"
}

```

Only the root-level output values from the remote state snapshot are exposed for use elsewhere in your module. Resource data and output values from nested modules are not accessible.

If you wish to make a nested module output value accessible as a root module output value, you must explicitly configure a passthrough in the root module.


##### State storage and Locking

Backends are responsible for storing state and providing an API for state locking. State locking is optional.

Backends determine where state is stored. For example, the local (default) backend stores state in a local JSON file on disk. The Consul backend stores the state within Consul. Both of these backends happen to provide locking: local via system APIs and Consul via locking APIs.
When using a non-local backend, Terraform will not persist the state anywhere on disk except in the case of a non-recoverable error where writing the state to the backend failed. This behavior is a major benefit for backends: if sensitive values are in your state, using a remote backend allows you to use Terraform without that state ever being persisted to disk.


You can still manually retrieve the state from the remote state using the `terraform state pull` command. This will load your remote state and output it to stdout. You can choose to save that to a file or perform any other operations.

You can also manually write state with `terraform state push`. This is *extremely dangerous* and should be avoided if possible. This will overwrite the remote state. This can be used to do manual fixups if necessary.

##### Locking

If supported by your backend, Terraform will lock your state for all operations that could write state. This prevents others from acquiring the lock and potentially corrupting your state.

State locking happens automatically on all operations that could write state. You won't see any message that it is happening. If state locking fails, Terraform will not continue. You can disable state locking for most commands with the -lock flag but it is not recommended.

To protect you, the force-unlock command requires a unique lock ID. Terraform will output this lock ID if unlocking fails. This lock ID acts as a nonce, ensuring that locks and unlocks target the correct lock.







## Terraform CLI




The usual way to run Terraform is to first switch to the directory containing the .tf files for your root module (for example, using the cd command), so that Terraform will find those files automatically without any extra arguments.

In some cases though — particularly when wrapping Terraform in automation scripts — it can be convenient to run Terraform from a different directory than the root module directory. To allow that, Terraform supports a global option -chdir=... which you can include before the name of the subcommand you intend to run:

```
terraform -chdir=environments/production apply

```


### Initializing working directories

Terraform expects to be invoked from a working directory that contains configuration files written in the Terraform language. Terraform uses configuration content from this directory, and also uses the directory to store settings, cached plugins and modules, and sometimes state data.

A Terraform working directory typically contains:
- A Terraform configuration describing resources Terraform should manage. This configuration is expected to change over time.
- A hidden .terraform directory, which Terraform uses to manage cached provider plugins and modules, record which workspace is currently active, and record the last known backend configuration in case it needs to migrate state on the next run. This directory is automatically managed by Terraform, and is created during initialization.
- State data, if the configuration uses the default local backend. This is managed by Terraform in a terraform.tfstate file (if the directory only uses the default workspace) or a terraform.tfstate.d directory (if the directory uses multiple workspaces).


Run the terraform init command to initialize a working directory that contains a Terraform configuration. After initialization, you will be able to perform other commands, like terraform plan and terraform apply.
Initialization performs several tasks to prepare a directory, including accessing state in the configured backend, downloading and installing provider plugins, and downloading modules. 
In fact, you can reinitialize at any time; the init command is idempotent, and will have no effect if no changes are required.

After successful installation, Terraform writes information about the selected providers to the dependency lock file. You should commit this file to your version control system to ensure that when you run terraform init again in future Terraform will select exactly the same provider versions. Use the -upgrade option if you want Terraform to ignore the dependency lock file and consider installing newer versions.


The `terraform get` command is used to download and update modules mentioned in the root module.

### Provisioning Infrastructure with Terraform

Terraform's primary function is to create, modify, and destroy infrastructure resources to match the desired state described in a Terraform configuration.

#### planning

The terraform plan command evaluates a Terraform configuration to determine the desired state of all the resources it declares, then compares that desired state to the real infrastructure objects being managed with the current working directory and workspace. It uses state data to determine which real objects correspond to which declared resources, and checks the current state of each resource using the relevant infrastructure provider's API.

The terraform plan command creates an execution plan. By default, creating a plan consists of:

- Reading the current state of any already-existing remote objects to make sure that the Terraform state is up-to-date.
- Comparing the current configuration to the prior state and noting any differences.
- Proposing a set of change actions that should, if applied, make the remote objects match the configuration.

The plan command alone will not actually carry out the proposed changes, and so you can use this command to check whether the proposed changes match what you expected before you apply the changes or share your changes with your team for broader review.


#### apply

The terraform apply command performs a plan just like terraform plan does, but then actually carries out the planned changes to each resource using the relevant infrastructure provider's API. It asks for confirmation from the user before making any changes, unless it was explicitly told to skip approval.

#### destroying

The terraform destroy command destroys all of the resources being managed by the current working directory and workspace, using state data to determine which real world objects correspond to managed resources. Like terraform apply, it asks for confirmation before proceeding.

A destroy behaves exactly like deleting every resource from the configuration and then running an apply, except that it doesn't require editing the configuration. This is more convenient if you intend to provision similar resources at a later date.

### Authentication

Commands to authenticate with terraform cloud/enterprise
```
terraform login
terraform logout
```



 # Terraform study guide

 https://learn.hashicorp.com/tutorials/terraform/associate-study?in=terraform/certification

 
## Tutorials

https://learn.hashicorp.com/collections/terraform/configuration-language
https://learn.hashicorp.com/tutorials/terraform/variables?in=terraform/configuration-language




####################
https://learn.hashicorp.com/tutorials/terraform/associate-study?in=terraform/certification


################





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

We can prevent terraform from queryin the current state during operations as terraform plan ( with the option `-refresh=false`) if we need to reduce the amount of API calls to a provider when doing some plan and apply.

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

Terraform also automatically loads a number of variable definitions files if they are present:

    Files named exactly terraform.tfvars or terraform.tfvars.json.
    Any files with names ending in .auto.tfvars or .auto.tfvars.json.


3 - Environment variable - part of your shell environment
4 - Default config - default value in variables.tf
5 - User manual entry - if not specified, prompt the user for entry


You can include also validation rules for the variable inputs:
```
variable "resource_tags" {
  description = "Tags to set for all resources"
  type        = map(string)
  default     = {
    project     = "my-project",
    environment = "dev"
  }

  validation {
    condition     = length(var.resource_tags["project"]) <= 16 && length(regexall("[^a-zA-Z0-9-]", var.resource_tags["project"])) == 0
    error_message = "The project tag must be no more than 16 characters, and only contain letters, numbers, and hyphens."
  }

  validation {
    condition     = length(var.resource_tags["environment"]) <= 8 && length(regexall("[^a-zA-Z0-9-]", var.resource_tags["environment"])) == 0
    error_message = "The environment tag must be no more than 8 characters, and only contain letters, numbers, and hyphens."
  }
}
```

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
terraform workspace new dev # creates dev workspace and switched into
terraform workspace select prd # select 

```

You can create a map with a default
the variable is terraform_workspace

The state file directory is `terraform.tfstate.d`

They DON'T provide high isolation between environments.

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

#Then to access the value you can use:
var.instance_type["dev"]
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

You can use `terraform plan -destroy`to preview the behaviour of a terraform destroy command.

##### Apply
It's used to apply the changes required to reach the desired state of the configuration.
Terraform apply will also write data to the terraform.tfstate fole.
Once apply is completed, the resources are immediately available.

##### Refresh

Command used to reconcile the state terraforms knows about with the real-world infrastructure.

This does NOT modify infrastructure but the state file.

Some commands run terraform refresh implicitly, like plan, destroy, apply.
Some others don't as init or import.

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

Array IS NOT SUPOPRTED

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


