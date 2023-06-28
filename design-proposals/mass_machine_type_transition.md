# Overview
With the release of RHEL 10, machine types prior to RHEL 9 will no longer be compatible. Currently we can support manually changing the machine type of individual VMs, but we need an automated process that will be able to change the machine types of multiple (e.g. thousands of) VMs to limit workload interruptions and to prevent the user from having to manually change every VM's machine type individually.

## Goals
* Creating an automated method for mass converting machine types of VMs
	* if currently running, update VM machine type and add a label to alert the user the VM must be restarted for the change to take effect
	* if currently running, take VM offline immediately and update machine type
	* if not currently running, update machine type immediately
* Allow user to specify certain VMs for updating machine type automation (e.g. by namespace)

## Non Goals
While these additions could be beneficial to cluster-admins who wish to use the mass machine type transition, these are not covered in the scope of this design proposal: 
* Allowing user to filter VMs by running state: a cluster admin may want to only update VMs that are offline.
* Allowing user to filter VMs in ways other than by namespace; e.g. with label-selector

Also not covered in this design proposal is the method in which users/cluster admins will be informed that they have VMs with unsupported machine types.

## User Stories
As a developer, I want to be notified that there are VMs created with RHEL < 9 machine types so I can update their machine type to get ready for RHEL 10.
As a developer, I want an automated way to update the machine type of all VMs to be compatible with RHEL 10 without interrupting my workflow.

## Repos
[kubevirt/kubevirt](https://github.com/kubevirt/kubevirt)

# Design
Create a new virtctl command that will automate the process of updating the machine type of the specified VMs. This command invokes a Kubernetes job that iterates through all VMs within a namespace, determines if the machine type is no longer supported, and updates it to the latest supported version. By default, it will update the machine type of VMs in all namespaces, and add the label `restart-vm-required` to any currently running VMs. However, the user can configure the environment variables `NAMESPACE` to update the machine type of VMs in a specific namespace and `RESTART_NOW` to have the job itself restart all running VMs automatically. 

The machine types that we will be updating will either be in the format `pc-q35-rhelx.x.x` or `q35`. If the VM's machine type is the former, it will simply be parsed and compared to the minimum supported machine type version, and updated or not updated accordingly. When the machine type is `q35`, it will be using whatever the latest machine type version is at the time of VMI creation. In the case of VMs that are not running, nothing needs to be updated here. However, for VMs that are currently running, it is possible that since VMI creation, the machine type of that VMI has become outdated (e.g. a VM was started when `pc-q35-rhel8.2.0` was the latest machine type version, but has not been restarted since), so the machine type in the VM Spec will also be updated to the latest version. 

## API Examples
(tangible API examples used for discussion)

## Scalability
(overview of how the design scales)

## Update/Rollback Compatibility
As both the minimum supported machine type version and the latest machine type version change in the future, these values are global constants in the mass machine type transition package. As the versions are updated in the future, these global constants can also be updated accordingly, allowing for easy maintainability.

## Functional Testing Approach
* Functional tests will follow the same basic procedure:
	* Create (a) VM(s) with the necessary machine type for the test case; start the VM if testing functionality of running VMs
	* Configure environment variables `NAMESPACE` and `RESTART_NOW` as necessary for the test case
	* Ensure VM spec has the correctly updated machine type version
	* Ensure the VM has/doesn't have the `restart-vm-required` label
	* Ensure all VMs have been updated accordingly
* Test cases:
	* Single VM (each of these will be their own individual test)
		* Running
			* VM machine type version less than the minimum supported version
				* `RESTART_NOW` is **false**
				* `RESTART_NOW` is **true**
			* VM machine type is `q35`
				* `RESTART_NOW` is **false**
				* `RESTART_NOW` is **true**
			* VM machine type version is greater than or equal to the minimum supported version
		* Not running
			* VM machine type version less than the minimum supported version
			* VM machine type version is greater than or equal to the minimum supported version
			* VM machine type is equal to `q35`
	* Multiple VMs (these cases will be split into 4 functional tests based on the environment variable configuration: `NAMESPACE`  specified/unspecified and `RESTART_NOW` **true**/**false**)
		* `NAMESPACE` is specified
			* `RESTART_NOW` is **true**
				* Running
					* VM machine type version less than the minimum supported version
					* VM machine type version is greater than or equal to the minimum supported version
					* VM machine type is equal to `q35`
				* Not running
					* VM machine type version less than the minimum supported version
					* VM machine type version is greater than or equal to the minimum supported version
					* VM machine type is equal to `q35`
			* `RESTART_NOW` is **false**
				* Running
					* VM machine type version less than the minimum supported version
					* VM machine type version is greater than or equal to the minimum supported version
					* VM machine type is equal to `q35`
				* Not running
					* VM machine type version less than the minimum supported version
					* VM machine type version is greater than or equal to the minimum supported version
					* VM machine type is equal to `q35`
		* `NAMESPACE` is not specified
			* `RESTART_NOW` is **true**
				* Running
					* VM machine type version less than the minimum supported version
					* VM machine type version is greater than or equal to the minimum supported version
					* VM machine type is equal to `q35`
				* Not running
					* VM machine type version less than the minimum supported version
					* VM machine type version is greater than or equal to the minimum supported version
					* VM machine type is equal to `q35`
			* `RESTART_NOW` is **false**
				* Running
					* VM machine type version less than the minimum supported version
					* VM machine type version is greater than or equal to the minimum supported version
					* VM machine type is equal to `q35`
				* Not running
					* VM machine type version less than the minimum supported version
					* VM machine type version is greater than or equal to the minimum supported version
					* VM machine type is equal to `q35`

# Implementation Phases

 - [ ] Create mass machine type transition package that Kubernetes job will use as an image
 - [ ] Create Kubernetes job yaml to be invoked with mass machine type transition subcommand
 - [ ] Insert package and job into kubevirt/pkg/virtctl and implement the subcommand (name TBD)
