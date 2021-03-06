---
layout: post
title:  "Using AWS Step Functions in CFML: Invoking and Tracking Step Function Workflows from CFML"
date:   2019-05-20 9:57:00 -0400
categories: AWS ColdFusion
---

We are now eight posts in to this series on using Step Functions in your CFML applications, and there hasn't been a whole lot of CMFL. We've looked in detail at [the first example workflow of performing image analysis on a randomly selected image](https://github.com/brianklaas/awsPlaybox/blob/master/stateMachines/choiceDemoStateMachine.json), and the various state types that make up that workflow. Now it's time to see how we invoke a Step Function from CFML and get data back from that Step Function invocation.

As always, [my AWSPlaybox application](https://github.com/brianklaas/awsPlaybox) has all of the code used in this series.

### Where Does CFML Fit in to Step Function Execution?

As we aren't using CFML to write our Lambda functions in this example &mdash; though we could use Pete Frietag's awesome [Fuseless project to write Lambda functions in CFML](https://fuseless.org) &mdash; CFML takes on a different role. CFML becomes the *scheduler* and *executor* of Step Functions workflow executions. Your CFML app will kick off the Step Functions workflow execution, check on its completion, and display the results of the execution (if any).

### Kicking Off Step Function Workflow Executions from CFML

In order to kick off a Step Functions workflow execution, we follow the typical request/reponse object pattern found throughout the [AWS Java SDK](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/index.html): you create a request object of some type, send it to the service, and get a response object back from the service. Here's the relevant code from [stepFunctions.cfm](https://github.com/brianklaas/awsPlaybox/blob/master/stepFunctions.cfm):

{% highlight javascript %}
stepFunctionService = application.awsServiceFactory.createServiceObject('stepFunctions');
stepFunctionARN = application.awsResources.stepFunctionRandomImageARN;
executionRequest = CreateObject('java', 'com.amazonaws.services.stepfunctions.model.StartExecutionRequest').init();
jobName = "AWSPlayboxExecution" & DateDiff("s", DateConvert("utc2Local", "January 1 1970 00:00"), now());
executionRequest.setName(jobName);
executionRequest.setStateMachineArn(stepFunctionARN);
executionType = "Image Description";
executionResult = stepFunctionService.StartExecution(executionRequest);
{% endhighlight %}

The stepFunctionARN property of the StartExecutionRequest object is the ARN ([Amazon Resource Name](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html)) of your Step Function workflow running in AWS. The ARN is the unique identifier, across all of AWS, for your Step Function workflow. It can be found in the Step Functions console.

The jobName of each execution request must be unique. In this case, we set it to "AWSPlayboxExecution" and the number of seconds since Epoch Time. That's good enough for this demo, though in a production application, you may want to use a UUID.

We're not providing any input to this particular workflow, so we only need to set the job name and ARN of our Step Functions workflow, and we can tell the Step Functions service to start the execution of the workflow as described in the executionRequest object.

### Tracking the Progress of a Step Functions Execution Request

While this first example workflow is fairly simple, there's no guarantee that any Step Functions workflow execution will occur immediately. Many Step Functions workflows take minutes or hours to complete. (A single Step Functions execution can run for as long as one whole year!) You don't want to tie up the current request thread by waiting for the Step Functions workflow to complete. For that reason, the "start execution" invocation returns immediately with an object containing information you can use to check on the status of that particular Step Functions execution in the future.

This is reflected in the [AWSPlaybox code](https://github.com/brianklaas/awsPlaybox/blob/master/stepFunctions.cfm) immediately after the above code:

{% highlight javascript %}
executionResult = stepFunctionService.StartExecution(executionRequest);
executionARN = executionResult.getExecutionARN();
executionInfo = { 
	executionType=executionType, 
	executionARN=executionARN,
	timeStarted=Now() 
};
arrayAppend(application.currentStepFunctionExecutions, executionInfo);
{% endhighlight %}

The result object contains a couple of pieces of information. The data we really need is the ARN of the unique execution of the workflow that we just invoked. That data is then appended to a structure in the application scope. We can use that information at any point in the future to check the status of this particular execution of the workflow.

In a production application, you would likely persist the ARN and any other important information about the execution to a database. You would then have a scheduled task running to periodically check the status of each active Step Functions workflow execution.

>Side note: if you are running [ColdFusion 2018](https://www.adobe.com/ColdFusion2018), it's possible that you could use the [support for Java Futures in ColdFusion 2018](https://coldfusion.adobe.com/2018/07/asynchronous-programming-in-coldfusion-2018-release/) to kick off a Step Functions workflow and wait for the result using the runAsync() construct. The problem with this approach is that Futures execute as single threads in ColdFusion 2018. If you have a Step Functions execution that runs for hours or days, that thread is kept alive for hours or days, which can lead to instability. Shorter execution times (seconds, minutes) should not have this problem.

### Checking the Status of an Individual Execution Request

It's simple to check on the status of an individual Step Functions execution. You simply ask AWS to "describe the execution request" via the DescribeExecutionRequest object. That request returns, as usual, a result object which contains details of the individual Step Functions execution. Here's the code for that:

{% highlight javascript %}
stepFunctionService = application.awsServiceFactory.createServiceObject('stepFunctions');
describeExecutionRequest = CreateObject('java', 'com.amazonaws.services.stepfunctions.model.DescribeExecutionRequest').init();
describeExecutionRequest.setExecutionArn(ARNofUniqueStepFunctionExecution);
describeActivityResult = stepFunctionService.describeExecution(describeExecutionRequest);
stepFunctionResult.status = describeActivityResult.getStatus();
{% endhighlight %}

Note that you must have the ARN of the execution that was returned when you originally invoked the Step Functions workflow in order to check on the status of that execution. It's important that you persist this information in some way when you invoke any Step Functions workflow. Without the ARN, you have to manually review all of your Step Functions executions in the Step Functions console to find the information you're looking for.

The status property of the DescribeActivityResult object that is returned from the DescribeExecutionRequest tells you the current status of your Step Functions execution. It can have one of three possible values:

- IN PROGRESS
- SUCCEEDED
- FAILED

If the status is not SUCCEEDED or FAILED, your code will generally not do anything because the workflow execution is still in progress. 

If the status is FAILED, you'll need to handle the failure of your execution appropriately in your calling application. You may need to re-run the workflow or take other action to compensate for the fact that your execution failed.

### Getting Data Back from an Individual Execution Request

If the status is SUCCEEDED, we know the execution is done and can get data back &mdash; if any was passed back to us from the workflow.

Retrieving data returned from a Step Functions execution is a simple call to deserialize the JSON returned from the Step Functions environment:

{% highlight javascript %}
if (stepFunctionResult.status IS "SUCCEEDED") {
	stepFunctionResult.finishedOn = describeActivityResult.getStopDate();
	stepFunctionResult.output = DeserializeJSON(describeActivityResult.getOutput());
	// Delete the this execution from the array of currently running executions in the application scope
	application.currentStepFunctionExecutions.each(function(element, index) {
		if (element.executionARN IS checkStepFunctionARN) {
			stepFunctionResult.invocationType = element.executionType;
			arrayDeleteAt(application.currentStepFunctionExecutions, index);
			break;
		}
	});
}
{% endhighlight %}

The data we receive is the data that was created in the last task state in our Step Functions workflow: the "getImageLabels" task state. The detectLabelsForImage Lambda function that is invoked in this task step returns a JSON structure to its caller (the Step Functions execution environment) which, in turn, returns that data to *its* caller (your CFML app) because the workflow reached a succeed state.

Whew!

That's it for the first example Step Functions workflow. In the next half of this series, I'll cover a second workflow &mdash; transcribing, translating, and speaking the translated content of a video &mdash; which introduces additional state types and demonstrates the power (and simplicity) of retries and error handling in Step Functions workflows.