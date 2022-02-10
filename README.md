Cloud Custodian
=================

<p align="center"><img src="https://cloudcustodian.io/img/logo_capone_devex_cloud_custodian.svg" alt="Cloud Custodian Logo" width="200px" height="200px" /></p>

Cloud Custodian is a rules engine for managing public cloud accounts and resources. It allows users to define policies to enable a well managed cloud infrastructure.

Custodian can be used to manage AWS, Azure, and GCP environments by ensuring real time compliance to security policies (like encryption and access requirements), tag policies, and cost management via garbage collection of unused resources and off-hours resource management. Custodian policies are written in simple YAML configuration files. It integrates with the cloud native serverless capabilities of each provider to provide for real time enforcement of policies with builtin provisioning.

Links
-----

-   [Homepage](http://cloudcustodian.io)
-   [Docs](http://cloudcustodian.io/docs/index.html)
-   [Developer Install](https://cloudcustodian.io/docs/developer/installing.html)
-   [Presentations](https://www.google.com/search?q=cloud+custodian&source=lnms&tbm=vid)

Quick Install
-------------

```shell
$ python3 -m venv custodian
$ source custodian/bin/activate
(custodian) $ pip install c7n
```


## <a id="concepts_and_terms"></a>Concepts and Terms

- **Policy** Policies first specify a resource type, then filter those resources, and finally apply actions to those selected resources. Policies are written in YML format.
- **Resource** Within your policy, you write filters and actions to apply to different resource types (e.g. EC2, S3, RDS, etc.). Resources are retrieved via the AWS API; each resource type has different filters and actions that can be applied to it.
- **Filter** are used to target the specific subset of resources that you're interested in. Some examples: EC2 instances more than 90 days old; S3 buckets that violate tagging conventions.
- **Action** Once you've filtered a given list of resources to your liking, you apply actions to those resources. Actions are verbs: e.g. stop, start, encrypt.
- **Mode** `Mode` specifies how you would like the policy to be deployed. If no mode is given, the policy will be executed once, from the CLI, and no lambda will be created. (This is often called `pull mode` in the documentation.) If your policy contains a `mode`, then a lambda will be created, plus any other resources required to trigger that lambda (e.g. CloudWatch event, Config rule, etc.).

## <a id="working_with_policies"></a>Working with Policies

The EC2 instance policies are laid out in the `policy.yml` file.

You can validate, test, and run Cloud Custodian with the example policy with these commands:

```shell
# Validate the configuration (note this happens by default on run)
$ custodian validate policy.yml

# Dryrun on the policies (no actions executed) to see what resources
# match each policy.
$ custodian run --dryrun -s out policy.yml

# Run the policy
$ custodian run -s out policy.yml

### Updating policies

The filters and actions available to you vary depending on the targeted resource (e.g. EC2, S3, etc.). Cloud Custodian has good CLI documentation to help you find the right filter or action for your needs. To get started, run

``` bash
custodian schema -h
```

to see all the different resources that Cloud Custodian supports. Then, to see the resource-specific filters/actions, run

``` bash
custodian schema [resourceName]
```

for example:

``` bash
custodian schema ec2
```

will list the filters and actions available for EC2. For details on a specific action or filter, the format is:

``` bash
custodian schema [resourceName].[actions or filters].[action or filter name]
```

for example:

``` bash
custodian schema ec2.filters.instance-age
```

### Configure the IAM role

Before running the policy, you'll need to give the resulting Lambda function the permissions required. Use the IAM policy provided for each Cloud Custodian policy file as a starting place: create the policy, attach it to a new role, and update the Cloud Custodian policy with the ARN of that role.


### Deploy the policies

Once you're satisfied with the results of your filters, deploy the policies with:

``` bash
custodian run -s . policy.yml
```

Cloud Custodian will take care of creating the needed lambda functions, CloudWatch events, etc. Sit back and watch it work!

### Set up the mailer

Some of the policies send notifications via SNS, email, or Slack. To send notifications, you'll need to implement the Mailer tool in your account. Instructions on how to do this are in [usingTheMailer.md](usingTheMailer.md). An IAM policy with permissions required by the mailer is in [mailer-permissions.json](mailer-permissions.json).

## <a id="modes"></a>More About Modes

Modes can be confusing. Here are the different mode types, what they do, and what their `yml` block should look like:

### asg-instance-state

`asg-instance-state` triggers the lambda in response to [Auto Scaling group state events](https://docs.aws.amazon.com/autoscaling/ec2/userguide/cloud-watch-events.html) (e.g. Auto Scaling launched an instance). The four events supported are:

| yml option | AWS ASG event `detail-type` |
| ---------- | --------------------------- |
|`launch-success` | "EC2 Instance Launch Successful" |
|`launch-failure` | "EC2 Instance Launch Unsuccessful" |
|`terminate-success` | "EC2 Instance Terminate Successful" |
|`terminate-failure` | "EC2 Instance Terminate Unsuccessful" |

Example:

``` yml
mode:
    role: #ARN of the IAM role you want the lambda to use
    type: asg-instance-state
    events:
        - launch-success
```

### cloudtrail

`cloudtrail` triggers the lambda in response to [CloudTrail events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference.html). The `cloudtrail` type comes in a couple different flavors: events for which there are shortcuts, and all other events.

#### Shortcuts

For very common API calls, Cloud Custodian has defined some [shortcuts](https://github.com/capitalone/cloud-custodian/blob/master/c7n/cwe.py#L28-L84) to target commonly-used CloudTrail events.

As of this writing, the available shortcuts are:

    - ConsoleLogin
    - CreateAutoScalingGroup
    - UpdateAutoScalingGroup
    - CreateBucket
    - CreateCluster
    - CreateLoadBalancer
    - CreateLoadBalancerPolicy
    - CreateDBInstance
    - CreateVolume
    - SetLoadBalancerPoliciesOfListener
    - CreateElasticsearchDomain
    - CreateTable
    - RunInstances

For those shortcuts, you simply need to specify:

``` yml
mode:
    role: #ARN of the IAM role you want the lambda to use
    type: cloudtrail
    events:
        - RunInstances
```

#### Other CloudTrail Events

You can also trigger your lambda via any other CloudTrail event; you'll just have to add two more pieces of information. First, you need the source API call - e.g. `ec2.amazonaws.com`. Secondly, you need a JMESPath query to extract the resource IDs from the event. For example, if `RunInstances` wasn't already a shortcut, you would specify it like so:

``` yml
mode:
    role: #ARN of the IAM role you want the lambda to use
    type: cloudtrail
    events:
        - source: ec2.amazonaws.com
          event: RunInstances
          ids: "responseElements.instancesSet.items[].instanceId"

```

### config-rule

`config-rule` creates a [custom Config Rule](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config_develop-rules.html) to trigger the lambda. Config rules themselves can only be triggered by configuration changes; triggering rules periodically is not supported. Use the `periodic` mode type instead.

Example:

``` yml
mode:
    role: #ARN of the IAM role you want the lambda to use
    type: config-rule
```

### ec2-instance-state

`ec2-instance-state` triggers the lambda in response to EC2 instance state events (e.g. an instance being created and entering `pending` state).

Available states:

    - pending
    - running
    - stopping
    - stopped
    - shutting-down
    - terminated
    - rebooting

Example:

``` yml
mode:
    role: #ARN of the IAM role you want the lambda to use
    type: ec2-instance-state
    events:
        - pending
```

### guard-duty

With `guard-duty`, your lambda will be triggered by [GuardDuty findings](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_findings.html). The associated filters would then look at the guard duty event detail - e.g. `severity` or `type`.

``` yml
mode:
    role: #ARN of the IAM role you want the lambda to use
    type: periodic
filters:
    - type: event
      key: detail.severity
      op: gte
      value: 4.5
```

### periodic

`periodic` creates a CloudWatch event to trigger the lambda on a given schedule. The `schedule` is specified using [scheduler syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html).

Example:

``` yml
mode:
    role: #ARN of the IAM role you want the lambda to use
    type: periodic
    schedule: "rate(1 day)"
```


Additional Tools
----------------

The Custodian project also develops and maintains a suite of additional
tools here
<https://github.com/cloud-custodian/cloud-custodian/tree/master/tools>:

- [**_Org_:**](https://cloudcustodian.io/docs/tools/c7n-org.html) Multi-account policy execution.

- [**_PolicyStream_:**](https://cloudcustodian.io/docs/tools/c7n-policystream.html) Git history as stream of logical policy changes.

- [**_Salactus_:**](https://cloudcustodian.io/docs/tools/c7n-salactus.html) Scale out s3 scanning.

- [**_Mailer_:**](https://cloudcustodian.io/docs/tools/c7n-mailer.html) A reference implementation of sending messages to users to notify them.

- [**_Trail Creator_:**](https://cloudcustodian.io/docs/tools/c7n-trailcreator.html) Retroactive tagging of resources creators from CloudTrail

- **_TrailDB_:** Cloudtrail indexing and time series generation for dashboarding.

- [**_LogExporter_:**](https://cloudcustodian.io/docs/tools/c7n-logexporter.html) Cloud watch log exporting to s3

- [**_Cask_:**](https://cloudcustodian.io/docs/tools/cask.html) Easy custodian exec via docker

- [**_Guardian_:**](https://cloudcustodian.io/docs/tools/c7n-guardian.html) Automated multi-account Guard Duty setup

- [**_Omni SSM_:**](https://cloudcustodian.io/docs/tools/omnissm.html) EC2 Systems Manager Automation

- [**_Mugc_:**](https://github.com/cloud-custodian/cloud-custodian/tree/master/tools/ops#mugc) A utility used to clean up Cloud Custodian Lambda policies that are deployed in an AWS environment.
