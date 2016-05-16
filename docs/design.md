## What?

Cloud-health executes scenarios and reports the execution time so that
operators can monitor the health and ability of a cloud system to
perform useful work.  Scenarios may check on individual services or
check end-to-end functionality of a system as a whole.


## Why?

Cloud systems are composed of multiple services.  Getting work done
requires interaction with multiple APIs, and actions performed by
multiple services.  Operators of IaaS and PaaS require knowledge about
end-user experience that is the result of complex interactions among
services that comprise the system.


### Definitions

- Scenario: an executable module that performs a sequence of steps to
  measure health of a system

- Task: a list of Scenarios to be executed.  Contains parameters for
  Scenarios and a context in which those Scenarios will be executed
  (e.g., details of the system under test).  A Task outputs a boolean
  success indicator and a dictionary of test results.

- Schedule: defines a when a Task will execute.

- Result: output from a Task.  Contains status, start time, duration,
  and any data that the Scenarios of the task produced.

- Publisher: Module that publishes the results of Tasks.  All Results
  are delivered the configured Publisher(s).  A Publisher may push data,
  or may expose an API that can be queried.


### Design

Cloud-health maintains state about Scenarios, Tasks, Schedules.
Cloud-health is an engine for execution of the Tasks and delivery of
Results.

Cloud-health does not store Results.  Instead, it passes the Results to a
configured Publisher.  A Publisher module may store Results or push
results to an external system.


### Sample Usage

The following example shows how to configure cloud-health to execute a
daily check to measure the time to create a Kubernetes pod.

1. Define a task
   POST /v1/tasks/
    { "kind": "Task",
      "apiVersion": "v1",
      "metadata": {
        "name": "CreateAndDeletePod",
        "description": "Check time to create and delete Pod"
      }
      "scenarios": [
        {
          "scenario": "Kube.create_and_delete_pod",
          "name": "Create pod with 2 replicas",
          "args": {
            "image": "test-image",
            "replicas": "2"
          }  
        }
      ]
    }

2. Define a Schedule to execute the Task at 2:00 each night.

    POST /v1/schedules/
    { "kind": "Schedule",
      "apiVersion": "v1",
      "metadata": {
        "name": "NightlyCreateAndDeleteCheck",
      },
      "taskID": "string"
      "when": [
        {
          "minute": "0",
          "hour": "2"
        }
      ]
    }

3. Task runs periodically.  Generates a Result.  An example Result may
   look like the following:

    { "kind": "Result",
      "apiVersion": "v1",
      "metadata": {
        "uid": "string"
      }
      "taskID": <task UID that produced result>,
      "status": <"pass" | "fail">,
      "startTime": 0,
      "durationMilliseconds": 0,
      "data": [
        {
          "type": "type of data",
          "key1": "data1",
          ...
        }
      ]
    }

4. Result is passed to a Publisher module.  Publisher modules are
   statically configured when starting the system.  Publisher acts on
   the Result.
   - Publisher may hold the data, to later be queried.
   - Publisher may push the data to external receiver


### APIs

List of Scenarios: GET /v1/scenarios/
List of Tasks: GET /v1/tasks/
List of Schedules: GET /v1/schedules/

Create Scenario: POST /v1/scenarios/
Create Task: POST /v1/tasks/
Create Schedule: POST /v1/schedules/

Delete Scenario: DELETE /v1/scenarios/<uid>
Delete Task: DELETE /v1/tasks/<uid>
Delete Schedule: DELETE /v1/schedules/<uid>


### Example: HTTP check

Define a Task that uses the Http.check_response Scenario.  The
Http.check_response module performs an GET on a specified URL.  It
compares the response code with an expected value, and also searches for
a string in the response data.  The “status” and “contains” arguments
are optional.

   POST /v1/tasks/
    { "kind": "Task",
      "apiVersion": "v1",
      "metadata": {
        "name": "CheckStatusEndpoint",
        "description": "Check if status endpoint is serving v1.0 API"
      }
      "scenarios": [
        {
          "scenario": "Http.check_response",
          "name": "Check status endpoint",
          "args": {
            "url": "http://example.com:5555/status",
            "status": 200,
            "contains": "\"apiVersion\": \"v1\""
          }
        }
      ]
    }


### Example: OpenStack VM creation
   POST /v1/tasks/
    { "kind": "Task",
      "apiVersion": "v1",
      "metadata": {
        "name": "CreateAndDeleteVM",
        "description": "Check time to create and delete VMs"
      }
      "scenarios": [
        {
          "scenario": "Nova.create_and_delete_instance",
          "name": "Create small instance",
          "args": {
            "flavor": "m1.tiny",
            "image": "test-image"
          }  
        },
        {
          "scenario": "Nova.create_and_delete_instance",
          "name": "Create large instance",
          "args": {
            "flavor": "m1.large",
            "image": "test-image"
          }  
        }
      ]
    }

