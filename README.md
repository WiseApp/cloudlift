# Cloudlift

--------------------

Fork by WISE:
This version has cloudlift version frozen on branch "primary", **tag: v1.11.2**

This is branched out from git sha 638de4c3f4aa4fdbd69afffc8ee2a1b1cdc5b266 (v1.10.0) of Rippling fork of Cloudlift, followed by pickle fixes for Python 3.7+.

This is frozen because it is risky to use Rippling fork master due to the active development going on.

# Before you follow steps below, check out tag given above (v1.11.X)

--------------------

Simple to use, heavily opinionated cli for running dockerized services deployed in AWS ECS.

## Installing cloudlift

### 1. Pre-requisites

- pip

```sh
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py | python get-pip.py
```

### Download and Install cloudlift

```sh
git clone git@github.com:jeechu/cloudlift.git
cd cloudlift
./install-cloudlift.sh
```

### 2. Configure AWS

```perl
aws configure
```

Enter the AWS Access Key ID, AWS Secret Access Key. You can find instructions
here on how to get Access Key ID and Secret Access Key here at
[AWS Docs](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)

#### Using AWS Profiles

If you are using [AWS profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html), set the desired profile name in the environment before invoking Cloudlift.

```sh
AWS_DEFAULT_PROFILE=<profile name> cloudlift <command>
```

OR

```sh
export AWS_DEFAULT_PROFILE=<profile name>
cloudlift <command>
cloudlift <command>
```

## Usage

### Create a new environment

Create a new environment for services to be deployed. Cloudlift creates a new
VPC for the given CIDR and sets up the required networking infrastructure for
services to run in ECS.

```sh
cloudlift create_environment -e <environment-name>
```

This starts a prompt for required details to create an environment, which
includes -

- AWS region for the environment
- VPC CIDR
- NAT Elastic IP allocation ID
- 2 Public Subnet CIDRs
- 2 Private Subnet CIDRs
- Minimum instances for cluster
- Maximum instances for cluster
- SSH key name
- SNS ARN for notifications
- AWS ACM ARN for SSL certificate

Once the configuration is saved, this is opened in the default `VISUAL` editor.
Here configurations can be changed if required.

### Update an environment

```sh
cloudlift update_environment -e <environment-name>
```

This opens the environment configuration in the `VISUAL` editor. Update this to
make changes to the environment.

### Create a new service

#### 1. Upload configuration to Parameter Store

During create_service and deployment `cloudlift` pulls the config from AWS
Parameter Store to apply it on the task definition. Configurations are stored in
path with the convention `/<environment>/<service>/<key>`

```sh
cloudlift edit_config -e <environment-name>
```

  _NOTE_: This is *not* required for every deployment. It's required only when
  config needs to be changed.

#### 2. Create service

In the repository for the application, run -

```sh
  cloudlift create_service -e <environment-name>
```

This opens the `VISUAL` editor with default config similar to -

```json5
  {
      "ecr_repo": {
        // ECR Repository to use
        "name": "test-repo"
      },
      "services": {
          "Test123": {
              "command": null,
              "http_interface": {
                  "alb": {
                    "create_new": true
                  },
                  "container_port": 80,
                  "internal": false,
                  "restrict_access_to": [
                      "0.0.0.0/0"
                  ]
              },
              // Use secrets from secrets manager with key test-env
              "secrets_name": "test-env",
              "memory_reservation": 100
          }
      }
  }
```

Definitions -

`services`: Map of all ECS services with configuration for current application

`command`: Override command in Dockerfile

`http_interface`: Configuration for HTTP interface if required, do not include
this if the services does not require a HTTP interface

`container_port`: Port in which the process is exposed inside container

`internal`: Scheme of loadbalancer. If internal, the loadbalancer is accessible
only within the VPC

`restrict_access_to`: List of CIDR to which HTTP interface is restricted to.

`memory_reservation`: Memory size reserved for each task in MBs. This is a soft
limit, i.e. at least this much memory will be available, and upto whatever
memory is free in running container instance. Minimum: 10 MB, Maximum: 8000 MB

`container_health_check` can be used to specify docker container health and maps to `healthCheck`
in ECS container definition. For more information, check [here](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_HealthCheck.html)

An example service using `container_health_check`

```json
{
  "services": {
    "Test123": {
      "command": null,
      "container_health_check": {
        "command": "curl http://localhost/health",
        "start_delay": 30,
        "retries": 10,
        "interval": 10,
        "timeout": 5
      },
      "memory_reservation": 100
    }
  }
}
```

**ALB**

`alb` allows us to configure how traffic can be routed to service

- To create a new ALB for the cloudlift service

```json5
{
  "alb": {
    // This creates a new ALB and attaches the target group to it.
    "create_new": true
  }
}
```

- To reuse an ALB

```json5
{
  "alb": {
    // Setting this to false means, the ALB is managed outside of this service definition.
    // We can use this mode to attach the target group to one of the listeners of an existing ALB
    "create_new": false,

    // Use listener_arn to attach the TargetGroup to an existing ALB's listener ARN.
    // The target group will be added using ListenerRule. Optional.
    // Default: If this is not specified, the environment level loadbalancer_listener_arn will
    // be picked up.
    "listener_arn": "<listener-arn>",

    // Use this to specify the priority of the listener rule. Optional.
    // Default: If this is not specified, a random available priority is used.
    "priority": 2,

    // Use this to specify host based routing. Optional.
    "host": "abc.xyz",

    // Use this to specify path based routing. Optional.
    "path": "/api/*",
  }
}
```

- When reusing ALB, you can configure the following alerts

```json5
{
  "alb": {
    // Setting this to false means, the ALB is managed outside of this service definition.
    // We can use this mode to attach the target group to one of the listeners of an existing ALB
    "create_new": false,

    // Fires when target 5xx is greater than threshold
    // default is 10
    "target_5xx_error_threshold": 10,

    // Fires when TargetResponseTime.p95 is greater than threshold seconds. (default: 15)
    "target_p95_latency_threshold_seconds": 15,
    // number of datapoints to evaluate
    "target_p95_latency_evaluation_periods": 5,
    // number of seconds to evaluate
    "target_p95_latency_period_seconds": 5,

    // Fires when TargetResponseTime.p99 is greater than threshold seconds. (default: 25)
    "target_p99_latency_threshold_seconds": 25,
    // number of datapoints to evaluate
    "target_p99_latency_evaluation_periods": 5,
    // number of seconds to evaluate
    "target_p99_latency_period_seconds": 5
  }
}
```

`autoscaling` allows to configure ScalingPolicy for ECS Service.

Supported autoscaling policies `request_count_per_target`. It works only if there is a `http_interface`

```json
{
  "services": {
    "Test123": {
      "command": null,
      "memory_reservation": 100,
      "http_interface": {},
      "autoscaling": {
        "max_capacity": 10,
        "min_capacity": 5,
        "request_count_per_target": {
          "target_value": 10,
          "scale_in_cool_down_seconds": 120,
          "scale_out_cool_down_seconds": 60
        }
      }
    }
  }
}
```

`container_labels` allows to add docker labels

```json5
{
  "services": {
    "Test123": {
      "command": null,
      "memory_reservation": 100,
      // Gets added as docker container labels
      "container_labels": {"key":  "value"}
    }
  }
}
```

#### 3. Deploy service

This command build the image (only if the version is unavailable in ECR), pushes to ECR and updates the ECS
service task definition. It supports `--build-arg` argument of `docker build` command as well to pass
custom build time arguments

```sh
  cloudlift deploy_service -e <environment-name>
```

For example, you can pass your SSH key as a build argument to docker build

```sh
  cloudlift deploy_service --build-arg SSH_KEY "\"`cat ~/.ssh/id_rsa`\"" -e <environment-name>
```

This example is bit comprehensive to show

- it can execute shell commands with "`".
- It's wrapped with double quotes to avoid line-breaks in SSH keys breaking the command.

### 6. Starting shell on container instance for service

You can start a shell on a container instance which is running a task for given
application using the `start_session` command. One pre-requisite for this is
installing the session manager plugin for `awscli`. To install session manager
plugin follow the [guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html#install-plugin-macos)

```sh
  cloudlift start_session -e <environment-name>
```

MFA code can be passed as parameter `--mfa` or you will be prompted to enter
the MFA code.

## Contributing to cloudlift

### Setup

To ensure the tests use the development version and not the installed version run (refer [here](https://stackoverflow.com/a/20972950/227705))

```sh
pip install -e .
```

### Tests

First level of tests have been added to assert cloudformation template generated
vs expected one.

```sh
py.test test/deployment/
```

To run high level integration tests

```sh
pytest -s test/test_cloudlift.py
```

To run tests inside docker container

```sh
docker build -t cloudlift .
docker run -it cloudlift
```

This tests expects to have an access to AWS console.
Since there's no extensive test coverage, it's better to manually test the
impacted areas whenever there's a code change.
