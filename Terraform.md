
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

Output (in DOT languaje) of the dependency graph for the terraform file/files.

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
