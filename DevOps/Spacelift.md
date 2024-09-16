# Spacelift
- Spacelift is a CI/CD tool designed specifically for IaC
- Generates and preview plans before applying PRs
- Has access control policies to define what different users can do
- Can be used to manage IaC state
- Built-in drift detection

## Spacelift Concepts
- Stacks - consists of 
    - source code
    - current state of your managed infrastructure (like Terraform state file)
    - environment vairables
- Policy - Spacelift uses Open Policy Agent (OPA) to provide a way to declaratively write policies as code. Eg -
    - Login - who gets to login to Spacelift
    - Access - who gets to access individual stacks
    - Approval - who can approve/reject a run
    - Initialization - which Run & Tasks can be started
    - Notification - routing and filtering notifications
    - Plan - which changes can be applied
    - Push - how git push events are interpreted
    - Task - which on-off commands can be executed
    - Trigger - what happens when blocking runs terminate
- Spacelift uses Rego as its policy language. Spacelift policies can be tested locally using opa test command. Syntax -
    ```
    package spacelift

    permission["message"] {
        conditional
    }

    sample { true } # to get the sample output (for debugging)
    ```
    where `permission` can be deny or allow. `message` is the output message and `conditional` is the condition on which the policy will be applied. Eg -
    ```
    deny["Policy was denied"] {
        instance := input.terraform.resource_changes[_].change.after.instance_type
        instance != sanitized("t2.micro")
    }
    ```
    - To attach a policy with a stack - go to `Policies` tab under the specific tab, select the policy from drop-down and click on `Attach`
- Task - can be used to run commands for troubleshooting

### Mounted Files
- Can we used to import environment variables from files.
> [!NOTE]
> The source code are cloned under `/mnt/workspace/source` directory in Spacelift.
- To mount a file, click on `Environment` tab, choose `Mounted file` and set the path for the file which will contains the variables (eg - `/mnt/workspace/source/terraform.tfvars`)



