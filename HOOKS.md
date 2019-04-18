# Hooks

A hook is a script in any language you prefer with an added events configuration code.

Addon-operator has two types of hooks:
- global hooks
- module hooks

First of all Addon-operator executes global hooks. After that, Addon-operator executes modules according their order. If module has hooks, Addon-operator executes them according to their order in a module.

> A corresponding runtime environment must be added into the image to execute a hook, e.g. [python](https://github.com/flant/shell-operator/tree/master/examples/startup-python).

Hooks are executable files lying under `$ADDON_OPERATOR_WORKING_DIR` path and each hook has two execution phases:
* [configuration](#hook-configuration)
* [event triggered execution](#event-triggered-execution)

## How hooks are run

Addon-operator searches and runs hooks going through the following steps:
- On start it recursively searches hooks in a working directory. A working directory is `/addons` by default and can be specified with a `ADDON_OPERATOR_WORKING_DIR` environment variable or with a `--working-dir` argument:
    - WORKING_DIR/global-hooks directory is examined for global hooks
    - WORKING_DIR/modules directory is examined for module hooks
    - directories and files starting with the dot symbol are excluded
    - directories and files are sorted alphabetically
    - every **executable** file counts as a hook.
- Every found hook is executed to get a binding configuration:
    - hooks are executed in alphabetical order
    - each hook is executed in the hook directory in which it is located with the `--config` argument and with the `WORKING_DIR` environment variable set
    - hook should print a JSON definition of its [bindings](#bindings) to stdout.
- Addon-operator starts its main loop:
    - hooks are added to the working queue to execute them sequentially
    - if there is more than one hook bound to an event, hooks are added in alphabetical order
    - hook is executed until success. It is re-added to the queue if hook is failed.
- Execution errors and working queue length can be scraped by Prometheus, see [METRICS](METRICS.md).


### What happens if hook fails?

Hook fail is a common UNIX execution error when the script exits with code not equal to 0. In this case Addon-operator has this scenario:
- writes the corresponding message to stdout (use e.g. `kubectl logs` to see it)
- increases the `addon_operator_hook_errors` counter
- re-adds hook into a working queue
- waits for 3 seconds
- executes the hook again.

If you want to skip failed hooks, use the `allowFailure: yes` option of `schedule` and `onKubernetesEvent` binding types. This is a simple scenario:

- writes the corresponding message to stdout (use e.g. `kubectl logs` to see it)
- increases the `addon_operator_hook_allowed_errors` counter.


## Hook configuration

Hook must print to stdout its binding configuration when the hook is executed with the `--config` argument.

Binding configuration is the following JSON structure:

```
{
  "BINDING_TYPE": "BINDING_PARAMETERS",
  ...
  "BINDING_TYPE": "BINDING_PARAMETERS",
  "BINDING_TYPE": [
       BINDING_PARAMETERS,
       BINDING_PARAMETERS
  ]
}
```

Where:
- `BINDING_TYPE` is a string with [binding type](#bindings).
- `BINDING_PARAMETERS` is a simple parameter for binding or a JSON array with binding parameters

### Bindings

Addon-operator provides the following types of bindings:
- **onStartup** — to run hooks on start;
- **schedule** — to run hooks periodically;
- **onKubernetesEvent** — to run hooks on changes of Kubernetes objects or their properties;
- **beforeAll** — to run global hooks after `onStartup` but before all modules;
- **afterAll** — to run global hooks after all modules have been started;
- **beforeHelm** — to run module hooks before helm install;
- **afterHelm** —  to run module hooks after helm install;
- **afterDeleteHelm** — to run module hooks after helm delete.

#### onStartup

This type of binding can be used in **global** or **module** hooks. It tells Addon-operator to execute hook once at Addon-operator start. The `onStartup` binding requires only a single parameter — a hook execution order. This value is used to sort hooks before execute them.

If an Addon-operator pod was recreated (e.g. pod in a deployment was deleted, or pod was evicted and sheduled on an another node) all global hooks with the `onStartup` binding will be reexecuted.

Module hook with the `onStartup` binding will be executed when the module is started or enabled (if you have enabled a previously disabled module).

Syntax:
```
{
  "onStartup": ORDER
}
```

#### schedule

This type of binding can be used in **global** or **module** hooks. It tells Addon-operator to execute a hook on a specified schedule or schedules. You can bind one hook for any number of schedules. The `schedule` binding requires a JSON array of event descriptors.

Syntax:
```
{
  "schedule": [
     {
      "name": "EVENT_NAME",
      "crontab": "CRON_EXPRESSION",
      "allowFailure": true|false,
    },
    {
      "crontab": "CRON_EXPRESSION",
      "allowFailure": true|false,
    },
    ...
  ]
}
```

The `name` is an optional parameter. It specifies a binding name, which hook can get in the execution time from the [binding context](#binding-context) file.

The `CRON_EXPRESSION` is a string with advanced crontab syntax: 6 values separated by spaces, each of them can be:
- a number
- a set of numbers separated by commas
- a range: two numbers separated by a hyphen
- expression with a [special characters](https://godoc.org/github.com/robfig/cron#hdr-Special_Characters): `*`, `/`
- [predefined values](https://godoc.org/github.com/robfig/cron#hdr-Predefined_schedules)
- [intervals](https://godoc.org/github.com/robfig/cron#hdr-Intervals).


Fields meaning:
```
* * * * * *
| | | | | |
| | | | | +---- Day of the Week   (allowed: 1-7, 1 standing for Monday)
| | | | +------ Month of the Year (allowed: 1-12)
| | | +-------- Day of the Month  (allowed: 1-31)
| | +---------- Hour              (allowed: 0-23)
| +------------ Minute            (allowed: 0-59)
+-------------- Second            (allowed: 0-59)
```

Examples:

```
*/10 * * * * *   Every 10 seconds

* 30 12 * * *    Everyday at 12:30

* * * * * 1      Every Monday
```

The `allowFailure` is an optional field, it is false by default. When set to true, Addon-operator will not try to re-execute hook if it fails and will skip it. See [what happens if hook fails](#what-happens-if-hook-fails).

#### beforeAll

This binding can be used only in **global** hooks and is triggered after all onStartup hooks have been triggered but before running modules.
Hooks with this binging are also executed when global values in a ConfigMap have changed.

```yaml
{
  "beforeAll": ORDER
}
```

#### afterAll

This binding can be used only in **global** hooks and is triggered after all modules have been started.
Hooks with this binging are also executed when global values in a ConfigMap have changed.

```yaml
{
  "afterAll": ORDER
}
```

#### beforeHelm

This binding can be used only in **module** hooks and is triggered before the module deploys.
Hooks with this binging are also executing when global values in a ConfigMap have changed or module values in a ConfigMap have changed.


```yaml
{
  "beforeHelm": ORDER
}
```

#### afterHelm

This binding can be used only in **module** hooks and is triggered after the module deploys.
Hooks with this binging are also executing when global values in a ConfigMap have changed or module values in a ConfigMap have changed.

```yaml
{
  "afterHelm": ORDER
}
```

#### afterDeleteHelm

This binding can be used only in **module** hooks and is triggered after deleting the module.

```yaml
{
  "afterDeleteHelm": ORDER
}
```

Module can be deleted in the following cases:
- module has been disables in a ConfigMap (`MODULENAME: "false"`);
- module has been disables in an `enabled` script.

> In case of deletion module directory  (`/modules/MODULE`) the module will be purged but not deleted ­— the `afterDeleteHelm` hook will not be triggered.

#### onKubernetesEvent

This type of binding tells Addon-operator to execute hook on a specified event from a Kubernetes cluster.

The `onKubernetesEvent` binding requires a JSON array of event descriptors. Each event descriptor assumes monitoring events related to objects of only one kind (see below).

Syntax:
```
{
  "onKubernetesEvent": [
    {
      "name": "BINDING_NAME",
      "kind": "KUBERNETES_OBJECT_KIND",
      "event": [ EVENT_TYPE_LIST ],
      "selector": {
        "matchLabels": {
          "myLabel": "myLabelValue",
          "someKey": "someValue",
          ...
        },
        "matchExpressions": [
          {
            "key": "tier",
            "operation": "In",
            "values": ["cache"],
          },
          ...
        ],
      },
      "namespaceSelector": {
        "matchNames": ["somenamespace", "proj-production", "proj-stage"],
        "any": true|false,
      },
      "jqFilter": ".metadata.labels",
      "allowFailure": true|false,
      "disableDebug": true|false,
    },
    {"name":"monitor Pods", "kind": "pod", ...},
    ...
    {...}
  ]
}
```

Where:
- `name` — is an optional parameter, specifies a binding name, which hook can get in the execution time from the [binding context](#binding-context) file.
- `kind` — is a case-insensitive kind of a Kubernetes objects you need to monitor. Can be one of the following:
    - namespace
    - cronjob
    - daemonset
    - deployment
    - job
    - pod
    - replicaset
    - replicationcontroller
    - statefulset
    - endpoints
    - ingress
    - service
    - configmap
    - secret
    - persistentvolumeclaim
    - storageclass
    - node
    - serviceaccount
- `event` — is an optional parameter, by default equals to `["add", "update", "delete"]`. Specifies an array of event types to monitor and can be any combination of:
    - `add` — an event of adding a Kubernetes resource of a specified `kind`
    - `update` — an event of updating a Kubernetes resource of a specified `kind`
    - `delete` — an event of deleting a Kubernetes resource of a specified `kind`.
- `selector` — is an optional parameter, specifies the condition for selecting a subset of objects. The `selector.matchLabels` is a `{key,value}` Kubernetes filter by label/selector. The `selector.matchExpressions` is a standard Kubernetes list of pod selector conditions (learn more about Kubernetes labels and selectors [here](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels)). If you don't specify a `selector`, all objects of specified `kind` are assumed.
- `namespaceSelector` — is an optional parameter, by defaul equals to `namespaceSelector.any=true`. Specifies a filter for namespaces for selecting objects. `namespaceSelector.matchNames` can contain an array of namespaces for selecting objects.
- `jqFilter` — is an optional parameter that specifies additional event filtering. Since a Kubernetes object has many events during its lifecycle, binding on the `update` event may give you a lot of events that you may not need. The `jqFilter` option allows trigger a hook only on object properties changes. `jqFilter` is applied to JSON manifest of changed object and hook is triggered only if output is changed from last execution. Thus `"jqFilter":".metadata.labels"` will trigger hook only on labels changes.
- `allowFailure` — is an optional field, it is false by default. When set to true, Addon-operator will not try to re-execute hook if it fails and will skip it. See [what happens if hook fails](#what-happens-if-hook-fails).


Example:
```
{
  "onKubernetesEvent": [
    {
      "name": "Trigger on labels changes of Pods with myLabel:myLabelValue in any namespace",
      "kind": "pod",
      "event": ["update"],
      "selector": {
        "matchLabels": {
          "myLabel": "myLabelValue"
        }
      },
      "namespaceSelector": {
        "any": true
      },
      "jqFilter": ".metadata.labels",
      "allowFailure": true
    }
  ]
}
```

> Note: Addon-operator Pod must have a `serviceAccount` with the appropriate `RoleBinding` or `ClusterRoleBinding` to monitor desired resources in a Kubernetes cluster. See [monitor namespaces](examples/monitor-namespaces) example.


## Event triggered execution

When one or more events are triggered, Addon-operator executes the hook without arguments and with the following environment variables set:
- `BINDING_CONTEXT_PATH` — a path to a file with information about an event that was triggered — binding context.
- `WORKING_DIR` — a path to a hooks directory. It can be helpful when you use common libraries in your hooks (`source $WORKING_DIR/lib/functions.sh`). See [common library](https://github.com/flant/shell-operator/tree/master/examples/common-library) example.
- `CONFIG_VALUES_PATH`, `VALUES_PATH` — input values for hooks
- `CONFIG_VALUES_JSON_PATCH_PATH`, `VALUES_JSON_PATCH_PATH` - hook can write a json patch into these files to change values


### Binding context

As hook can have more than one bindings, it should identify a binding (or in other words — an event) which was triggered. When a hook is executing, the BINDING_CONTEXT_PATH environment variable contains a path to a JSON file with information about the binding that was triggered.

The binding context contains the following fields:
- `binding` — contains the name of the binding type or a `name` value from the binding configuration (if it was set)
- `resourceEvent` — an event type that was triggered: add, update or delete
- `resourceNamespace`, `resourceKind`, `resourceName` — corresponding values for identifying a Kubernetes object ("Kubernetes coordinates").

For example, you have the following binding configuration of a hook:
```json
{
  "schedule": [
    {
      "name": "incremental",
      "crontab": "* 2 */3 * * *",
      "allowFailure": true
    },
    {
      "crontab": "* 59 23 * * *",
      "allowFailure": false
    }
  ]
}
```

At 12:02 this hook will be executed with the following binding context:
```json
[{ "binding": "incremental"}]
```

At 23:59 this hook will be executed with the following binding context:
```json
[{ "binding": "schedule"}]
```

## Values

> TODO: How to work with input values and generate json patches.

## Debug

For debugging purposes you can use the following:
- get logs of a Addon-operator pod by using `kubectl -n addon-operator logs addon-operator`
- get a hook queue content in the `/tmp/addon-operator-queue` file of a Addon-operator pod
- set RLOG_LOG_LEVEL=DEBUG environment to get DEBUG messages in log
- use `--debug=yes` argument to get even more messages
- get a hook queue content by requesting a `/queue` endpoint:
   ```
   kubectl port-forward po/addon-operator 9115:9115
   curl localhost:9115/queue
   ```
