# Terraform Runner Container States

> PROPOSAL

This is design doc to outline the current limitations with Terraform Runners and a design to fix them.

## Current Limitations

The way Terraform Runner works today is by executing a bash script in a job pod that has been configured by the controller. When the pod is done (ie the container has exited), the job status goes to `Completed` which lets the controller know the run is complete. However, there is no concept of "SUCCESS" or "FAILURE" of the terraform execution on the controller level. The controller only knows that the pod has exited. The controller doesn't know anything about the run other than that.

Here are some of the known limitations of the Terraform Runner and the communication back to the controller:

- There's no way for the job pod to communicate back to the controller about the state of the run. Whether the pod is setting up, running `terraform init`, running `terraform apply`, or in the middle of the `prerunScript` or `postrunScript` is hard to know.
- There's no way to communicate back to the controller the final status of the terraform execution. What should happen upon `terraform <cmd>` errors? 
- There is no _good_ way to do plan reviews. The only way to do this is a hack of setting `applyOnCreate | applyOnUpdate | applyOnDelete` actions to `false`, reviewing the plan by viewing the pod logs, and then updating a ConfigMap to continue with the plan. This is terribly inefficient, hard to automate, and hard to remember all the steps involved.
- The Terraform Runner naively tries 10 times to run terraform. Terraform could be failing in any `terraform <cmd>` and the only way to know is to tail the pod logs.
- The Bash script of the Terraform Runner is tied to the Docker image-tag which correlates directly to the Terraform version. A user specifies the `spec.terraformVersion` to select the correct Docker image they need. Now, if the Bash script needed to be updated (for new features or any other reason), all the Docker images need to be updated and then re-tagged again with the same tag. Using the same tag ensures all users get the updates. But the new features in the Bash script might break a user's runner for any number of reasons. The bottom line, the Bash script can never be updated because it's impossible to test everyone's terraform setup.

## Proposal

Still working on this... it's late and I need to get to sleep. :tired_face: