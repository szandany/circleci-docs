---
layout: classic-docs
title: Config Policies
categories: []
description: How to leverage CircleCI containers
version:
- Cloud
- Beta
---

## Introduction
{: #introduction }

CircleCI uses `config.yml` files to define CI pipelines at the project level. For most use cases this is a convenient way of developing and iterating quickly as each pipeline can be made to meet the needs of the project as it grows. However, from an organizational standpoint, it can be tricky to manage and enforce organization-wide conventions and security policies. 

Config Policy Management uses a decision engine leveraging OPA (Open-Policy-Agent) to allow users to specify policies, and return a decision about if a Pipeline's Config complies with those policies. In the strictest case, if a pipeline configuration does not comply with the organization's policies, that pipeline will be blocked from running until it does comply. Decision's are also stored and can be audited allowing for useful feedback about what pipeline defintions are being run in your organization.

Policies are written in the `rego` query language defined by OPA, and following certain circleci conventions / semantics, rule violations are surfaced when pipelines are triggered.

The following sections will describe the CircleCI specific semantics, what is a CircleCI Decision, and how to use the CircleCI-CLI to manage your policies.

#TODO: A section on the architecture/process - how a config is evaluated 

## Writing Rego Policies using CircleCI Domain-Specific Language
{: #writing-rego-policies }

# TODO: re-read to ensure no confusion between rego rules and CircleCI violations

In order for CircleCI to make decisions about configs, it needs to be able to interpret the output 
generated by policy evaluation. To do this, a policy must be written to meet CircleCI specifications. 
Policies must be written in `rego`, a purpose-built declarative policy language that supports Open Policy 
Agent (OPA). You can find more information about `rego` [here](https://www.openpolicyagent.org/docs/latest/policy-language/).

All policies must belong to the `org` package. All policy rego files should include:

```shell
package org
```

After declaring the `org` package, policies can then be defined as a list of "rules". Each rule is composed of 
three parts:

    1. Evaluation - This is the Rego that evaluates if the config contains the policy violation.
    2. Enforcement status - This determines how a violation should be enforced.
    3. Enablement - This determines if a policy violation should be enabled.

Using this format allows policy writers to create custom helper functions without impacting CircleCI's ability to
parse policy evaulation output. Policies all have access to config data through the `input` variable. The `input` is the project config being evaluated. Since the `input` matches the CircleCI config, you can write rules to enforce a desired state on any any available config strucutre i.e. `jobs`, `workflows`, etc.

```shell
input.workflows     # an array of nested structures mirroring workflows in the CircleCI config
input.jobs          # an array of nested structures mirroring jobs in the CircleCI config
```

### Rule Definition
{: #rule-definition }
#TODO: update "evaluation name" to "rule definition"

This is how the decision engine determines if a config violates the given policy. The evaluation defines the name and id of the rule, checks a condition, and returns a user-friendly string describing the violation. Rule evaluations include the **rule name** and an **optional rule id**. The rule name will be used to enable and set the enforcement level for a rule.

#TODO: specify expected outputs

```shell
RULE_NAME = reason {
  ... # some comparison
  reason := "..."
}

RULE_NAME[RULE_ID] = reason {
  ... # some comparison
  reason := "..."
}
```

Here is an example of a simple evaluation that checks that a config includes at least one workflow:
```shell
contains_workflows = reason {
    count(input.workflows) > 0
    reason := "config must contain at least one workflow"
}
```

The rule id can be used to differentiate between multiple violations of the same rule. For example, if a config uses multiple unofficial docker images, this might lead to multiple violations of a `use_official_docker_image` rule. Rule ids should only be used when multiple violations are expected. In some cases, the customer may only need to know if a rule passes or not. In this case, the rule will not need a rule id.

```shell
use_official_docker_image[image] = reason {
  some image in docker_images   # docker_images are parsed below
  not startswith(image, "circleci")
  reason := sprintf("%s is not an approved Docker image", [image])
}

# helper to parse docker images from the config
docker_images := {image | walk(input, [path, value])  # walk the entire config tree
                          path[_] == "docker"         # find any settings that match 'docker'
                          image := value[_].image}    # grab the images from that section

```

### Enforcement
{: #enforcement }

The policy service allows rules to be enforced at different levels.

#TODO: add default level which is soft fail

```shell
ENFORCEMENT_STATUS["RULE_NAME"]
```

The two available enforcement levels are:
* `hard_fail` - If the `policy-service` detects that the config violated a rule set as `hard_fail`, the build will fail.
* `soft_fail` - If the `policy-service` detects that the config violated a rule set as `soft_fail`, the build will continue but the violation will be logged in the `policy-service` decision log.

```shell
hard_fail["use_official_docker_image"]
```

### Enablement
{: #enablement }

A rule must be enabled for it to be inspected for policy violations. Rules that are not enabled do not need to match CircleCI violation output formats, and can be used as helpers for other rules. 

```shell
enable_rule["RULE_NAME"]
```

To enable a rule, add the rule as a key in the `enable_rule` object.

```shell
enable_rule["use_official_docker_image"]
```

### Example Policy
{: #example-policy }

The following is an example of a complete policy with one rule, `use_official_docker_image`, which checks that
all docker images in a config are prefixed by `circleci`. It uses some helper code to find all the `docker_images`
in the config. It then sets the enforcement status of `use_official_docker_image` to `hard_fail` and enables the rule.

```shell
package org


use_official_docker_image[image] = reason {
  some image in docker_images   # docker_images are parsed below
  not startswith(image, "circleci")
  reason := sprintf("%s is not an approved Docker image", [image])
}

# helper to parse docker images from the config
docker_images := {image | walk(input, [path, value])  # walk the entire config tree
                          path[_] == "docker"         # find any settings that match 'docker'
                          image := value[_].image}    # grab the images from that section

hard_fail["use_official_docker_image"]

enable_rule["use_official_docker_image"]
```

## CircleCI Rego Helpers
{: #rego-helpers }

# TODO: Could this be separated into a separate page?

The `circle-policy-agent` package includes built-in functions for common config policy
use cases. All policies evaluated by the `policy-service`, the `circle-cli`, or the `circle-policy-agent`
will be able to access these functions. This also means the package name `circleci.config` is
reserved.

### `jobs`
{: #rego-helpers-jobs }

`jobs` is a Rego object containing jobs that are present in the given CircleCI config file. It 
can be utilized by policies related to jobs.

#### Definition
{: #rego-helpers-jobs-defintion }

```
jobs = []string
```

Example `jobs` object:
```json
[
    "job-a",
    "job-b",
    "job-c"
]
```

#### Usage
{: #rego-helpers-jobs-usage }

```rego
package org

import future.keywords
import data.circleci.config

jobs := config.jobs
```


### `require_jobs`
{: #rego-helpers-require-jobs }

This function requires a config to contain jobs based on the job names. Each job in the list of 
required jobs must be in at least one workflow within the config.

#### Definition
{: #rego-helpers-require-jobs-definition }

```rego
require_jobs([string])
returns { string }
```

#### Usage
{: #rego-helpers-require-jobs-usage }

```rego
package org

import future.keywords
import data.circleci.config

require_security_jobs = config.require_jobs(["security-check", "vulnerability-scan"])

enable_rule["require_security_jobs"]

hard_fail["require_security_jobs"] {
	require_security_jobs
}
```

### `orbs`
{: #rego-helpers-orbs }

`orbs` is a Rego object containing orbs and versions present in the given config file. It 
can be utilized by policies related to orbs.

#### Definition
{: #rego-helpers-orbs-definition }

```rego
orbs[string] = string
```

Example `orbs` object:
```json
{
    "circleci/security": "1.2.3",
    "circleci/foo": "3.2.1"
}
```

#### Usage
{: #rego-helpers-orbs-usage }
```rego
package org

import future.keywords
import data.circleci.config

my_orbs := config.orbs
```


### `require_orbs`
{: #rego-helpers-require-orbs }

This function requires a config to contain orbs based on the orb names. Versions should not 
be included in the provided list of orbs.

#### Definition
{: #rego-helpers-require-orbs-definition }

```
require_orbs([string])
returns { string: string }
```

#### Usage
{: #rego-helpers-require-orbs-usage }

```rego
package org

import future.keywords
import data.circleci.config

require_security_orbs = config.require_orbs(["circleci/security", "foo/bar"])

enable_rule["require_security_orbs"]

hard_fail["require_security_orbs"] {
	require_security_orbs
}
```

### `require_orbs_version`
{: #rego-helpers-require-orbs-version }

This function requires a policy to contain orbs based on the orb name and version.

#### Definition
{: #rego-helpers-require-orbs-version-definition }

```
require_orbs_version([string])
returns { string: string }
```

#### Usage
{: #rego-helpers-require-orbs-version-usage }

```rego
package org

import future.keywords
import data.circleci.config

require_orbs_versioned = config.require_orbs_version(["circleci/security@1.2.3", "foo/bar@4.5.6"])

enable_rule["require_orbs_versioned"]

hard_fail["require_orbs_versioned"] {
	require_orbs_versioned
}
```

### `ban_orbs`
{: #rego-helpers-ban-orbs }

This function violates a policy if a config includes orbs based on the orb name. Versions should not 
be included in the provided list of orbs.

#### Definition
{: #rego-helpers-ban-orbs-defintion }

```rego
ban_orbs_version([string])
returns { string: string }
```

#### Usage
{: #rego-helpers-ban-orbs-usage }

```rego
package org

import future.keywords
import data.circleci.config

ban_orbs = config.ban_orbs(["evilcorp/evil"])

enable_rule["ban_orbs"]

hard_fail["ban_orbs"] {
	ban_orbs
}
```

### `ban_orbs_version`
{: #rego-helpers-ban-orbs-version }

This function violates a policy if a config includes orbs based on the orb name and version.

#### Definition
{: #rego-helpers-ban-orbs-version-definition }

```rego
ban_orbs_version([string])
returns { string: string }
```

#### Usage
{: #rego-helpers-ban-orbs-version-usage }

```rego
package org

import future.keywords
import data.circleci.config

ban_orbs_versioned = config.ban_orbs_version(["evilcorp/evil@1.2.3", "foo/bar@4.5.6"])

enable_rule["ban_orbs_versioned"]

hard_fail["ban_orbs_versioned"] {
	ban_orbs_versioned
}
```

## Leveraging the CLI for Config and Policy Development

### Developing Configs

The over arching goal of policies for CircleCI configs is to detect violations in configs and stop builds that do not comply
with your organization's policies. However, this raises an issue for local development of circleci.yml files: modifications to your config.yml
may cause your pipeline to be blocked. This slows down development time and can be frustrating in certain situations.

It is possible to run your config.yml against your organization's policies outside of CI using the CircleCI-CLI to get immediate feedback on config compliance.

The following command will request a decision for the provided config input and return a Circle Decision containing the status of the decision
and any violations that may have occurred. 

# TODO: add links to circleci-cli/references

__Remote Decision Command__
```bash
circleci policy decide --owner-id $ORG_ID --input $PATH_TO_CONFIG
```

__Example Resulting Decision__
```json
{
    "status": "HARD_FAIL",
    "hard_failures": [
        {
            "rule": "custom_rule",
            "reason": "custom failure message"
        }
    ],
    "soft_failures": [
        {
            "rule": "other_rule",
            "reason": "other failure message"
        }
    ]
}
```

### Developing Policies

The CLI provides a language agnostic way of evaluating local policies against arbitrary config inputs. It is the recommended
way of developing and testing policies. It is similar to the previous command except that it provides a path to the local policies.
The path can be to a policy file, or to a directory of files. If it is a directory, files will be bundled into a policy non-recursively.

```bash
circleci policy decide --owner-id $ORG_ID --input $PATH_TO_CONFIG --policy $PATH_TO_POLICY_FILE_OR_DIR
```

It is recommended that users build a test suite of policy/config combinations and run them locally or in CI before pushing them to their organization's active policies.

# TODO: further discuss testing recommendations (maybe a separate written section)

### Get Policy Decision Audit logs

Audit logs provide documentary evidence for a policy decision being performed at certain point of time.
These include the inputs which influenced the decision of the policy decision, as well as the outcome of the decision.

The CLI provides `policy logs` command to fetch the policy decision logs for your organization. 

Following is the output of this command when run with `--help` flag:

```shell
circleci logs --help

# Returns the following:
Get policy (decision) logs


Usage:
  circleci policy logs [flags]

Examples:
policy logs  --owner-id 462d67f8-b232-4da4-a7de-0c86dd667d3f --after 2022/03/14
--out output.json

Flags:
      --after string        filter decision logs triggered AFTER this datetime
      --before string       filter decision logs triggered BEFORE this datetime
      --branch string       filter decision logs based on branch name
  -h, --help                help for logs
      --out string          specify output file name
      --project-id string   filter decision logs based on project-id
```

- The organization ID information is required, which can be provided with `--owner-id` flag.
- The command currently accepts following filters for the logs: `--after`, `--before`, `--branch` and `--project-id`.
- These filters are optional. Also, any combination of filters can be used to suit your auditing needs.

#### output
- stdout - by default, the decision logs are printed as a list of logs to the standard output.
- file - output can be written to a file (instead of stdout). This can be done by providing filepath using `--out` flag

## Using the CLI Policy Management

The CircleCI-CLI can be leveraged as a tool to manage your organization's policies programmatically.

The commands to perform policy management are group under `policy` command. 
Following sub-commands are currently supported within the CLI for configuration policy management:
- `create` - creates a new policy
- `list` - fetches a list of all the policies within your org
- `get` - fetches a given policy along with the policy content
- `update` - updates (one of the attributes of) given policy
- `delete` - deletes the given policy

The above list are "sub-commands" in the CLI, which would be executed like so:

```shell
circleci policy list --help

# Returns the following:
List all policies

Usage:
  circleci policy list [flags]

Examples:
policy list --owner-id 516425b2-e369-421b-838d-920e1f51b0f5 --active=true

Flags:
      --active   (OPTIONAL) filter policies based on active status (true or false)
  -h, --help     help for list
```

- The organization ID information is required, which can be provided with `--owner-id` flag.
- As with most of the CLI's commands, you will need to have properly authenticated your version of the CLI with a token to enable performing context related actions.


### Putting it all together

Config Policy Management is a beta feature. If this feature interests you please contact us to participate in the beta. 

#### Create your first a policy file 

The first step is to create a policy file. We recommend storing it in a repository.
Let's create a policy that checks the version of our circleci config and ensure that it is greater than or equal to `2.1`.

Create `version.rego` with the following content:

```rego
# All policies start with the org package definition
package org

# signal to circleci that check_version is enabled and must be included when making a decision
enable_rule["check_version"]

# signal to circleci that check_version is a hard_failure condition and that builds should be
# stopped if this rule is not satisfied.
hard_fail["check_version"]

# define check version
check_version = reason {
    not input.version # check the case where version is not in the input
    reason := "version must be defined"
} {
    not is_number(input.version) # check that version is number
    reason := "version must be a number"
} {
    not input.version >= 2.1 # check that version is at least 2.1
    reason := sprintf("version must be at least 2.1 but got %s", [input.version])
}
```

#### Upload the new policy using the CircleCI-CLI

```bash
circleci-cli policy create --context config --owner-id $ORG_ID --policy ./version.rego --name version_check
```

That is it! Now when a pipeline is triggered, the project's config will be validated against this policy.

#### Updating the policy

Suppose you made an error when creating that policy, and that configs in your organization are using
circleci config version `2.0` and that you want your policy to reflect this.

Simple change the rule definition in your `version.rego` file:

```rego
{
    not input.version >= 2.0 # check that version is at least 2.0
    reason := sprintf("version must be at least 2.0 but got %s", [input.version])
}
```

and update the policy using the CLI (the policy id was part of the response from the create command):

```bash
circleci-cli policy update $POLICY_ID --owner-id $ORG_ID --policy ./version.rego
```