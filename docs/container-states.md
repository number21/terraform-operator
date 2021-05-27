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

> [scratch ideas..]
>
> - Option #1) Orchestrate a lot of little pods for the different tasks. For example, after the `terraform init` pod runs, it exits and the next pod can start.
> <br/>
> In this scenario, the controller is not only aware of the run's status, it can also customize the next pod. (Not sure what for yet, but that sounds like it'd be kind of cool for something.)
>
> - Option #2) Add multiple containers in a pod with pauses that wait for a trigger to activate. This is somewhat similar to the pod-per-task, but it's all contained in a single pod. 
> <br/>
> The controller can check the exit status of the last finished pod to know where in the run the pod is.
>
> &nbsp;

<!-- First and foremost the Terraform Runner should be as simple as possible. Much of the logic should be handled by the controller where possible. Some concepts are easy to understand, like letting the controller handle saving outputs to a ConfigMap.  -->

After thinking about it a bit, and reading this article https://learn.hashicorp.com/tutorials/terraform/automate-terraform?in=terraform/automation from Hashicorp, I think the best approach is the multi pod approach. This lends way to allow the controller to make decisions each step of the way. 

However, there is still a fundamental issue of the Terraform Runner scripts; this solution doesn't make it easier to update the scripts once users start adopting the project. 

> [terraform runner script improvement ideas...]
>
> - Option #1) Just let it be. The current script isn't all that complicated. Some hacks can actually be programmatically added to the Pod instead of scripted. 
>
> - Option #2) Change the name of the default runner ~~when updates might break something~~ anytime the bash script is updated. Users can pin the runner by defining `spec.terraformRunner` in their tf resource. This means that backwards compatibility needs to be in mind when updating the project with new features. 
> <br/>
> New features should:
>     - safely be ignored by old runners
>     - warn the user that they need to update the runner to a specific image
>
> - Option #3) Change the image tags instead of the image name. This might not be intuitive since the tag is also known as the `terraformVersion` in it's current design. This is the same concept as above and the same limitations apply, but users have to update the `spec.terraformVersion` on everything existing to take the new features. Also, users need to be aware of the tags and what they do... This is a terrible option lol
>
> &nbsp;

For the runner, option #2 sounds the most promising and I'll be moving forward with that unless something else pops up as a better option or if it's a complete blocker in the overall proposal. 

Still working on this... it's late and I need to get to sleep. :tired_face: