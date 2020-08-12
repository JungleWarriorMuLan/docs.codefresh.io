---
title: "Hooks"
description: "Execute commands before/after each pipeline or step"
group: codefresh-yaml
toc: true
---

Pipeline hooks allow you to run specific actions at end and the beginning of the pipeline as well as before/after a step

## Pipeline hooks

Codefresh allows you to run a specific step before each pipeline as well as after it has finished. Each hook is similar to a [freestyle step]({{site.baseurl}}/docs/codefresh-yaml/steps/freestyle/) as you need to define:

1. A Docker image that will be used to run specific commands
1. One or more commands to run within the context of that docker image.

For simple commands we suggest you use a small image such as alpine, but any Docker image can be used in hooks.

### Running a step at the end of the pipeline

You can easily run a step at end of pipeline, that will execute even if one of the steps have failed

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
hooks: 
 on_finish:
   exec:
     image: alpine:3.9
     commands:
       - echo "cleanup after end of pipeline"

steps:
  step1:
    title: "Step 1"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Hello world"
  step2:
    title: "Step 2"
    type: "freestyle" 
    image: node:10-buster
    commands:
      - echo "There was an error"
      - exit 1   
{% endraw %}
{% endhighlight %}

In the example above we define a hook for the whole pipeline that will run a step (the `exec` keyword) inside `alpine:3.9` and will simply execute an `echo` command. Because we have used the `on_finish` keyword, this step will execute even if the pipeline fails.

This scenario is very common if you have a cleanup step or a notification step that you always want to run at the end of the pipeline. You will see the cleanup logs in the top pipeline step.

 {% include 
image.html 
lightbox="true" 
file="/images/codefresh-yaml/hooks/cleanup-step.png" 
url="/images/codefresh-yaml/hooks/cleanup-step.png" 
alt="Running a cleanup step" 
caption="Running a cleanup step" 
max-width="80%" 
%}

Apart from the `on_finish` keyword you can also use `on_success` and `on_fail` if you want the step to only execute according to a specific workflow. It is also possible to use multiple hooks.

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
hooks: 
 on_finish:
   exec:
     image: alpine:3.9
     commands:
       - echo "cleanup after end of pipeline"
 on_success:
   exec:
     image: alpine:3.9
     commands:
       - echo "Send a notification only if pipeline was successful"
 on_fail:
   exec:
     image: alpine:3.9
     commands:
       - echo "Send a notification only if pipeline has failed"       
steps:
  step1:
    title: "Step 1"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Hello world"
  step2:
    title: "Step 2"
    type: "freestyle" 
    image: node:10-buster
    commands:
      - echo "There was an error"
      - exit 1 #Comment this line out to see how hooks change

{% endraw %}
{% endhighlight %}

Note that if you have multiple hooks like the example above, the `on_finish` segments will always execute after any `on_success`/`on_fail` segments (if they are applicable).


### Running a step at the start of the pipeline

Similar to the end of the pipeline, you can also execute a step in the beginning of the pipeline with the `on_elected` keyword:

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
hooks: 
 on_elected:
   exec:
     image: alpine:3.9
     commands:
       - echo "Creating an adhoc test environment"
 on_finish:
   exec:
     image: alpine:3.9
     commands:
       - echo "Destroying test environment"
steps:
  step1:
    title: "Step 1"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Running Integration tests on test environment"
  step2:
    title: "Step 2"
    type: "freestyle" 
    image: node:10-buster
    commands:
      - echo "Running acceptance tests on test environment"

{% endraw %}
{% endhighlight %}

All pipeline hooks will be shown in the "initializing process" logs:

 {% include 
image.html 
lightbox="true" 
file="/images/codefresh-yaml/hooks/before-pipeline.png" 
url="/images/codefresh-yaml/hooks/before-pipeline.png" 
alt="Hooks before a pipeline" 
caption="Hooks before a pipeline" 
max-width="80%" 
%}

It is possible to define all possible hooks (`on_elected`, `on_finish`, `on_success`, `on_fail`) in a single pipeline, if this is required by your workflow.

## Step hooks

Hooks can also be defined for individual steps inside a pipeline. This capability allows you for more granular control on defining prepare/cleanup phases for specific steps. 

The syntax for step hooks is the same as pipeline hooks (`on_elected`, `on_finish`, `on_success`, `on_fail`), you just need to put the respective segment under a step instead of the root of the pipeline.

For example, this pipeline will always run a cleanup step after integration tests (even if the tests fail).

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
steps:
  step1:
    title: "Compile application"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Building application"
  step2:
    title: "Unit testing"
    type: "freestyle" 
    image: node:10-buster
    commands:
      - echo "Running unit tests"
    hooks:
      on_finish:
        exec:
          image: alpine:3.9
          commands:
          - echo "Create test report"
  step3:
    title: "Uploading artifact"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Upload to artifactory"
{% endraw %}
{% endhighlight %}


Logs for steps hooks are shown in the GUI window of the step itself.

 {% include 
image.html 
lightbox="true" 
file="/images/codefresh-yaml/hooks/step-after.png" 
url="/images/codefresh-yaml/hooks/step-after" 
alt="Hooks before a pipeline" 
caption="Hooks before a pipeline" 
max-width="80%" 
%}

As with pipeline hooks, it is possible to define multiple hook conditions for each step.

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
steps:
  step1:
    title: "Compile application"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Building application"
  step2:
    title: "Security scanning"
    type: "freestyle" 
    image: node:10-buster
    commands:
      - echo "Running Security scan"
    hooks:
      on_elected:
        exec:
          image: alpine:3.9
          commands:
          - echo "Authenticating to security scanning service"    
      on_finish:
        exec:
          image: alpine:3.9
          commands:
          - echo "Uploading security scan report"
      on_fail:
        exec:
          image: alpine:3.9
          commands:
          - echo "Sending slack notification"          

{% endraw %}
{% endhighlight %}

The order of events is the following.

1. The `on_elected` segment executes first (authentication)
1. The step itself executes (the security scan)1
1. The `on_fail` segment executes (only if the step throws an error code)
1. The `on_finish` segment always executes at the end


## Controlling errors inside pipeline/step hooks

By default if a step fails within a pipeline, the whole pipeline will stop and be marked as failed. This is also true for `on_elected` segments as well. If they fail then the whole pipeline will fail (regardless of the position of the segment in a pipeline or step).

For example the following pipeline will fail right away, because the pipeline hook fails at the beginning.

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
hooks: 
 on_elected:
   exec:
     image: alpine:3.9
     commands:
       - echo "failing on purpose"
       - exit 1 
steps:
  step1:
    title: "Step 1"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Running Integration tests on test environment"
{% endraw %}
{% endhighlight %}

You can change this behavior by using the familiar [fail_fast property]({{site.baseurl}}/docs/codefresh-yaml/what-is-the-codefresh-yaml/#execution-flow) inside an `on_elected` hook.

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
hooks: 
 on_elected:
   exec:
     image: alpine:3.9
     fail_fast: false
     commands:
       - echo "failing on purpose"
       - exit 1 
steps:
  step1:
    title: "Step 1"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Running Integration tests on test environment"
{% endraw %}
{% endhighlight %}

This pipeline will now execute successfully and `step1` will still run as normal, because we have used the `fail_fast` property. You can also use the `fail_fast` property on step hooks as well:

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
steps:
  step1:
    title: "Step 1"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Running Integration tests on test environment"
    hooks: 
     on_elected:
       exec:
         image: alpine:3.9
         fail_fast: false
         commands:
         - echo "failing on purpose"
         - exit 1 
{% endraw %}
{% endhighlight %}


>Notice that the `fail_fast` property is only available for `on_elected` hooks. The other types of hooks (`on_finish`, `on_success`, `on_fail`) do not affect the outcome of the pipeline in any way. Even if they fail, the pipeline will continue running to completion. This behavior is not configurable.


## Using multiple steps for hooks

In all the previous examples, each hook was a single step running on a single Docker image. You can also defined multiple steps for each hook. This is possible by inserting an extra `steps` keyword inside the hook:

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
hooks: 
 on_finish:
   steps:
     mycleanup:
       image: alpine:3.9
       commands:
         - echo "echo cleanup step"
     mynotification:
       image: cloudposse/slack-notifier
       commands:
         - echo "Notify slack"
steps:
  step1:
    title: "Step 1"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Running Integration tests on test environment"
{% endraw %}
{% endhighlight %}

By default all steps in a single hook segment are executed one after the other. But you can also run them in [parallel]({{site.baseurl}}/docs/codefresh-yaml/advanced-workflows/#inserting-parallel-steps-in-a-sequential-pipeline):

`codefresh.yml`
{% highlight yaml %}
{% raw %}
version: "1.0"
steps:
  step1:
    title: "Compile application"
    type: "freestyle"
    image: node:10-buster
    commands:
      - echo "Building application"
  step2:
    title: "Unit testing"
    type: "freestyle" 
    image: node:10-buster
    commands:
      - echo "Running Integration tests"
      - exit 1 
    hooks:
      on_fail:
        mode: parallel
        steps:
          upload-my-artifact:
            image: maven:3.5.2-jdk-8-alpine
            commands:
            - echo "uploading artifact" 
          my-report:
            image: alpine:3.9
            commands:
            - echo "creating test report" 
{% endraw %}
{% endhighlight %}

You can use multiple steps in a hook in both the pipeline and the step level. 


## Using annotations and labels in hooks

## Syntactic sugar syntax

## Limitations of pipeline/step hooks

With the current implementation of hooks, the following limitations are known:

* [Codefresh variables]({{site.baseurl}}/docs/codefresh-yaml/variables/) are not interpolated inside hook segments
* The [debugger]({{site.baseurl}}/docs/configure-ci-cd-pipeline/debugging-pipelines/) cannot inspect commands inside hook segments
* Hooks are not supported for [parallel steps]({{site.baseurl}}/docs/codefresh-yaml/advanced-workflows/)



## What to read next

* [Conditional Execution of Steps]({{site.baseurl}}/docs/codefresh-yaml/conditional-execution-of-steps/)
* [Condition Expression Syntax]({{site.baseurl}}/docs/codefresh-yaml/condition-expression-syntax/)
* [Working Directories]({{site.baseurl}}/docs/codefresh-yaml/working-directories/)
* [Annotations]({{site.baseurl}}/docs/codefresh-yaml/annotations/)


