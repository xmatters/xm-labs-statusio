# Status.io Outbound (from xMatters) Integration
With this Outbound Integration, components on Status.io affected by one or more xMatters events have incidents created for them unless they are in an active maintenance window. When the last xMatters event affecting a component has been terminated, the incident for the component is updated to show that the component is now healthy again but being monitored.

The concept behind this integration is to allow the Status.io dashboard to be updated automatically from events happening on xMatters.

<kbd>
  <a href="https://support.xmatters.com/hc/en-us/community/topics"><img src="https://github.com/xmatters/xMatters-Labs/raw/master/media/disclaimer.png"></a>
</kbd>

# Pre-Requisites
* [Status.io](https://status.io) account.
* xMatters account - If you don't have one, [get one](https://www.xmatters.com)!

# Files
* [StatusioSteps.zip](StatusioSteps.zip) - This is an example workflow. Note that it does not include any inbound integration to trigger the creation of an xMatters event.

# Included Custom Steps

The attached [StatusioSteps.zip](StatusioSteps.zip) file contains the custom steps for use in other Workflows. The steps included:

<kbd>
  <img src="/media/customsteps.png">
</kbd>

# How it works
This integration is intended to be added to an existing workflow that is triggered when there is a problem and when that problem has been resolved, e.g. the AWS CloudWatch Integration.

The logic of the workflow is expected to follow something like the following:

1. Inbound trigger fires, providing the following information:
   - the component that is affected by the trigger
   - the severity of the problem
   - (optional) a description of the problem for internal consumption (e.g. a Jira ticket for IT Support)

2. If the trigger indicates an alarm status, the first step used is `Status.io component in maintenance?` in order to decide whether or not an incident should be created.

3. If the output from that step is `false`, a xMatters event is created, followed by `Status.io: status mapper` and `Status.io: create incident`.

4. If the trigger indicates a non-alarm status, the workflow runs a `Get x Events for y` xMatters step followed by `Status.io: update incdent`. The workflow should only attempt to terminate the underlying xMatters event as the final step. This will be explained in more detail below.

The following image shows an example complete workflow that started from the CloudWatch Integration. The boxes marked in yellow have been added to incorporate the Status.io integration.

<img src="media/example-workflow.png" width="50%">

# Installation
## Alarm Planning
As noted above, when the inbound trigger fires the workflow, there is a need to determine the affected component, the severity of the problem and, optionally, a description of the problem. How that is dealt with is entirely dependent on your workflow and the trigger.

Taking the CloudWatch Integration as an example, one approach is to set the CloudWatch Alarm Description to a string of a defined format. For example:

`fqdn.example.com|DEGRADE|This service is currently experiencing high CPU load.`

A custom step can then parse that string, divide it into the appropriate parts and store it in event values for later re-use.

## Get Status.io API values
1. Go to https://status.io and log in.

2. Click on `API` on the left-hand navbar.

3. Make a note of your Status Page ID.

4. Click on `Display API Credentials`.

5. Make a note of your API ID and API Key.

## xMatters initial set up
1. Log in to your xMatters instance.

2. Import the `StatusioIntegration.zip` file.

3. In the workflow where you want to use this integration, click on the `Integration Builder` tab.

4. Click on `Edit Constants`.
   - Click on `Add Constant`. Set the name to `Status.io Page ID` and set the value to the Status Page ID previously noted. 
   - Click on `Save Changes`.
   - Click on `Add Constant`. Set the name to `Status.io API ID` and set the value to the API ID previously noted. Click on `Save Changes`.
   - Click on `Add Constant`. Set the name to `Status.io API Key` and set the value to the API Key previously noted. Click on `Save Changes`.

5. Click on `Edit Endpoints`.

6. If there isn't already an endpoint called `Status.io`, follow these steps:
   - Click on `Add Endpoint`.
   - Set the name to `Status.io`.
   - Set the base URL to `https://api.status.io`.
   - Set the authentication type to `None`.
   - Click on `Save Changes`.

7. In the workflow, click on the `Properties` tab.
   - Click on `Create Property`.
   - Click on `Text` then click on `Next`.
   - Enter `AffectedComponent` as the name.
   - Change the `Maximum Size` to 256 characters.
   - Click on `Create Property`.

8. In the workflow, click on the `Forms` tab.
   - Against the form used to notify agents when an alarm has occurred, click on `Edit` > `Layout`.
   - Drag `AffectedComponent` from the `Properties` list on the right-hand side into the `Custom Section`.
   - Click on `Save Changes`.


## Adding the integration
To add the integration to your workflow, follow these steps:

1. After the trigger, add a custom step to define `Component`, `Alarm Severity` and, optionally, `Alarm reason`.

2. In the part of the workflow that handles a new alarm going off:
   - Add the `Status.io: component in maintenance?` step.
     - Set the `Status.io Component` field to the Component value from step #1.
     - Set the `Status.io Page ID` field to the Page ID constant.
     - Set the `Status.io API ID` field to the API ID constant.
     - Set the `Status.io API Key` field to the API Key constant.
     - Click on the `Endpoint` tab. xMatters should automatically select the `Status.io` endpoint.
     - Click on `OK`.
   - Add a `Switch` step and connect it to the output from the maintenance step.
     - Select `Status.io: component in maintenance?.Is in maintenance` as the property.
     - Set the switch value to `false`.
   - From the output of the `false` box, add `xMatters Create Event`, `Status.io: status mapper` and `Status.io: create incident`.
     - In the `Create Event` step, set `AffectedComponent` to the Component value from step #1. Set any other event values as desired.
     - In the `Status Mapper` step, set the `Status` field to the `Alarm Severity` field from step #1 then enter appropriate values to map your trigger severity thresholds to the four different levels supports by Status.io.
     - In the `Create Incident` step:
       - Set the `Status.io Component` field to the Component value from step #1
       - Set the `Status.io Page ID` field to the Page ID constant.
       - Set the `Status.io API ID` field to the API ID constant.
       - Set the `Status.io API Key` field to the API Key constant.
       - Set the `Status description` to any desired value.
       - Set the `Status code` to the output from `Status.io: status mapper> Status.io status`
       - In each of the `Notify` fields, set the string to `1` if you want Status.io to use that notification channel/mechanism.

3. In the part of the workflow that handles an alarm being cleared, **before** the step that terminates events, do the following:
   - Add a `Get Events` step from the `Tools` section.
     - Set the `Step Label` to `Get ACTIVE Events for Affected Component`.
     - Set `Status` to `ACTIVE`.
     - Set `Property Name` to `AffectedComponent#en`.
     - Set the `Property Value` field to the Component value from step #1.
     - Set `Exact Property Value Match` to `TRUE`.
     - Click `OK`.
   - Add a `Status.io: update incident` step and connect it to the output from the `Get Events` step.
     - Set the `Status.io Component` field to the Component value from step #1.
     - Set the `xMatters Event Count` field to the `Get ACTIVE Events for Affected Component > Total Event Count` value.
     - Set the `Status.io Page ID` field to the Page ID constant.
     - Set the `Status.io API ID` field to the API ID constant.
     - Set the `Status.io API Key` field to the API Key constant.
     - Set `New incident state` to an appropriate value, typically 300 which means monitoring. Alternatively, use 200 which means that the problem has been identified.
     - Set `Update message` to an appropriate message to show on the incident.
     - In each of the `Notify` fields, set the string to `1` if you want Status.io to use that notification channel/mechanism.
   - Connect the output from the `Update Incident` step to the input of the `Terminate Events` step.

# Testing
Testing your workflow will require the triggering of alarms and then clearing them again. When the first alarm for a given component triggers the workflow, you should see an incident appearing in the Status.io dashboard for that component. Subsequent alarms for the same component should not create additional incidents but, if they have a higher severity than earlier alarms, the incident will be updated to reflect that.

When clearing the alarms, only the last alarm to be cleared for the component should result in the incident being updated. The incident should show that the component is now considered to be operational and you should see the state and message defined in step #3 above.

# Troubleshooting
One of the best ways to troubleshoot the workflow steps is to use the `Activity` button in the top right corner of the Workflow Designer. When you click on that, the Activity pane appears in the bottom part of the Designer. If the `Logging` option is not turned on, turn it on and then try to reproduce the problem. Once you've done that, click on the name in the Activity pane to look at the logs.

When you click on a request, xMatters highlights the steps that have been executed. You can click on individual steps to look at the inputs, parameters, headers, body and outputs. If checking each of those steps doesn't show what is going wrong, the next thing to look at is the `Log` tab. In addition to the HTTP traffic between xMatters and external systems, the various Status.io steps themselves also emit additional logging information to show what is being done.

# Known Issues
There is a race condition opportunity when multiple alarms go off for the same component within a narrow time window. If this happens, multiple incidents can be created for the same component. When the alarms get cleared, only one of the incidents will get updated (the first one that was created).
