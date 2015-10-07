---

title: 'Transactions in Processes'
weight: 160

menu:
  main:
    identifier: "user-guide-process-engine-transactions-in-processes"
    parent: "user-guide-process-engine"

---


The process engine is a piece of passive Java Code which works in the Thread of the client. For instance, if you have a web application allowing users to start a new process instance and a user clicks on the corresponding button, some thread from the application server's http-thread-pool will invoke the API method `runtimeService.startProcessInstanceByKey(...)`, thus *entering* the process engine and starting a new process instance. We call this "borrowing the client thread".

On any such *external* trigger (i.e. start a process, complete a task, signal an execution), the engine runtime will advance in the process until it reaches wait states on each active path of execution. A wait state is a task which is performed *later*, which means that the engine persists the current execution to the database and waits to be triggered again. For example in case of a user task, the external trigger on task completion causes the runtime to execute the next bit of the process until wait states are reached again (or the instance ends). In contrast to user tasks, a timer event is not triggered externally. Instead it is continued by an *internal* trigger. That is why the engine also needs an active component, the [job executor]({{< relref "user-guide/process-engine/the-job-executor.md" >}}), which is able to fetch registered jobs and process them asynchronously.


# Wait States

 We talked about wait states as transaction boundaries where the process state is stored to the database, the Thread returns to the client and the transaction is committed. The following BPMN elements are always wait states:

<div data-bpmn-symbol="receivetask" data-bpmn-symbol-name="Receive Task">
	<a href="ref:/api-references/bpmn20/#tasks-receive-task">
	  <div id="1" title="go to Receive Task"></div>
	</a>
</div>
<br>
<div data-bpmn-symbol="usertask" data-bpmn-symbol-name="User Task">
	<a href="ref:/api-references/bpmn20/#tasks-user-task">
	  <div id="1" title="go to User Task"></div>
	</a>
</div>
<br>
<a href="ref:/api-references/bpmn20/#events-message-events"><div data-bpmn-symbol="intermediatecatchevent/message"  data-bpmn-symbol-name="Message"></div> Message Event</a><br>
<a href="ref:/api-references/bpmn20/#events-timer-events"><div data-bpmn-symbol="intermediatecatchevent/timer"  data-bpmn-symbol-name="Timer"></div> Timer Event</a><br>
<a href="ref:/api-references/bpmn20/#events-signal-events"><div data-bpmn-symbol="intermediatecatchevent/signal"  data-bpmn-symbol-name="Signal"></div> Signal Event</a><br><br>


The [Event Based Gateway]({{< relref "reference/bpmn20/gateways/event-based-gateway.md" >}}):

<div data-bpmn-diagram="../bpmn/event-based-gateway" > </div>

Keep in mind that [Asynchronous Continuations]({{< relref "#asynchronous-continuations" >}}) can add transaction boundaries to other tasks as well.


# Transaction Boundaries

The transition from one such stable state to another stable state is always part of one transaction, meaning that it succeeds as a whole or is rolled back on any kind of exception occuring during its execution. This is illustrated in the following example:

{{< img src="../img/transactions-1.png" title="Transaction Boundaries" >}}

We see a segment of a BPMN process with a user task, a service task and a timer event. The timer event marks the next wait state. Completing the user task and validating the address is therefore part of the same unit of work, so it should succeed or fail atomically. That means that if the service task throws an exception we want to roll back the current transaction, so that the execution tracks back to the user task and the user task is still present in the database. This is also the default behavior of the process engine.

In **1**, an application or client thread completes the task. In that same thread the engine runtime is now executing the service task and advances until it reaches the wait state at the timer event (**2**). Then it returns the control to the caller (**3**) potentially committing the transaction (if it was started by the engine).


# Asynchronous Continuations

## Why Asynchronous Continuations?

In some cases the synchronous behavior is not desired. Sometimes it is useful to have custom control over transaction boundaries in a process.
The most common motivation is the requirement to scope *logical units of work*. Consider the following process fragment:

{{< img src="../img/transactions-2.png" title="Asynchronous Continuations" >}}

We are completing the user task, generating an invoice and then sending that invoice to the customer. It can be argued that the generation of the invoice is not part of the same unit of work: we do not want to roll back the completion of the usertask if generating an invoice fails.
Ideally, the process engine would complete the user task (**1**), commit the transaction and return
control to the calling application (**2**). In a background thread (**3**), it would generate the invoice.
This is the exact behavior offered by asynchronous continuations: they allow us to scope transaction
boundaries in the process.


## Configure Asynchronous Continuations

Asynchronous Continuations can be configured *before* and *after* an activity. Additionally, a
process instance itself may be configured to be started asynchronously.

An asynchronous continuation before an activity is enabled using the `camunda:asyncBefore` extension
attribute:

```xml
<serviceTask id="service1" name="Generate Invoice" camunda:asyncBefore="true" camunda:class="my.custom.Delegate" />
```

An asynchronous continuation after an activity is enabled using the `camunda:asyncAfter` extension
attribute:

```xml
<serviceTask id="service1" name="Generate Invoice" camunda:asyncAfter="true" camunda:class="my.custom.Delegate" />
```

Asynchronous instantiation of a process instance is enabled using the `camunda:asyncBefore`
extension attribute on a process-level start event.
On instantiation, the process instance will be created and persisted in the database, but execution
will be deferred. Also, execution listeners will not be invoked synchronously. This can be helpful
in various situations such as [heterogeneous clusters]({{< relref "user-guide/process-engine/the-job-executor.md#cluster-setups" >}}),
when the execution listener class is not available on the node that instantiates the process.

```xml
<startEvent id="theStart" name="Invoice Received" camunda:asyncBefore="true" />
```


## Asynchronous Continuations of Multi-Instance Activities

A [multi-instance activity]({{< relref "reference/bpmn20/tasks/task-markers.md#multiple-instances" >}}) can be configured for asynchronous continuation like other activities. Declaring asynchronous continuation of a multi-instance activity makes the multi-instance body asynchronous, that means, the process continues asynchronously *before* the instances of that activity are created or *after* all instances have ended.

Additionally, the inner activity can also be configured for asynchronous continuation using the `camunda:asyncBefore` and `camunda:asyncAfter` extension attributes on the `multiInstanceLoopCharacteristics` element:

```xml
<serviceTask id="service1" name="Generate Invoice" camunda:class="my.custom.Delegate">
	<multiInstanceLoopCharacteristics isSequential="false" camunda:asyncBefore="true">
 		<loopCardinality>5</loopCardinality>
	</multiInstanceLoopCharacteristics>
</serviceTask>
```

Declaring asynchronous continuation of the inner activity makes each instance of the multi-instance activity asynchronous. In the above example, all instances of the parallel multi-instance activity will be created but their execution will be deferred. This can be useful to take more control over the transaction boundaries of the multi-instance activity or to enable true parallelism in case of a parallel multi-instance activity.


## Understand Asynchronous Continuations

In order to understand how asynchronous continuations work, we first need to understand how an activity is
executed:

{{< img src="../img/process-engine-activity-execution.png" title="Asynchronous Continuations" >}}

The above illustration shows how a regular activity which is entered and left by a sequence flow is
executed:

1. the "TAKE" listeners are invoked on the sequence flow entering the activity.
2. the "START" listeners are invoked on the activity itself.
3. the behavior of the activity is executed: the actual behavior depends on the type of the
   activity: in case of a `Service Task` the behavior consists in invoking [Delegation Code]({{< relref "user-guide/process-engine/delegation-code.md" >}}), in case of a `User Task`, the behavior consists in creating a `Task` instance in the task list etc...
4. the "END" listeners are invoked on the activity.
5. The "TAKE" listeners of the outgoing sequence flow are invoked.

Asynchronous Continuations allow putting break points between the execution of the sequence flows
and the execution of the activity:

{{< img src="../img/process-engine-async.png" title="" >}}

The above illustration shows where the different types of asynchronous continuations break the
execution flow:

* an asynchronous continuation BEFORE an activity breaks the execution flow between the invocation
  of the incoming sequence flow's TAKE listeners and the execution of the activity's START
listeners.
* an asynchronous continuation AFTER an activity breaks the execution flow after the invocation of
  the activity's END listeners and the outgoing sequence flow's TAKE listeners.

Asynchronous continuations directly relate to transaction boundaries: putting an asynchronous
continuation before or after an activity creates a transaction boundary before or after an activity:

{{< img src="../img/process-engine-async-transactions.png" title="" >}}

What's more, asynchronous continuations are always executed by the [Job
Executor]({{< relref "user-guide/process-engine/the-job-executor.md" >}}).


# Rollback on Exception

We want to emphasize that in case of a non handled exception the current transaction gets rolled back and the process instance is in the last wait state (safe point). The following image visualizes that.

{{< img src="../img/transactions-3.png" title="Rollback" >}}

If an exception occurs when calling `startProcessInstanceByKey` the process instance will not be saved to the database at all.


# Reasoning for This Design

The above sketched solution normally leads to discussion as people expect the process engine to stop in case the task caused an exception. Also, other BPM suites often implement every task as a wait state. But this approach has a couple of **advantages**:

 * In Testcases you know the exact state of the engine after the method call, which makes assertions on process state or service call results easy.
 * In production code the same is true; allowing you to use synchronous logic if required, for example because you want to present a synchronous user experience in the front-end as shown in the tutorial "UI Mediator".
 * The execution is plain Java computing which is very efficient in terms of performance.
 * You can always switch to 'asyncBefore/asyncAfter=true' if you need different behavior.

But there are consequences which you should keep in mind:

 * In case of Exceptions the state is rolled back to the last persistent wait state of the process instance. It might even mean that the process instance will never be created! You cannot easily trace the exception back to the node in the process causing the exception. You have to handle the exception in the client.
 * Parallel process paths are not executed in parallel in terms of Java Threads, the different paths are executed sequentially, since we only have and use one Thread.
 * Timers cannot fire before the transaction is committed to the database. Timers are explained in more detail later, but they are triggered by the only active part of the Process Engine where we use own Threads: The Job Executor. Hence they run in an own thread which receives the due timers from the database. But in the database the timers are not visible before the current transaction is visible. So the following timer will never fire:

{{< img src="../img/NotWorkingTimerOnServiceTimeout.png" title="Not Workin Tiemout" >}}


# Transaction Integration

The process engine can either manage transactions on its own ("Standalone" transaction management)
or integrate with a platform transaction manager.


## Standalone Transaction Management

If the process engine is configured to perform standalone transaction management, it always opens a
new transaction for each command which is executed. To configure the process engine to use
standalone transaction management, use the
`org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration`:

```java
ProcessEngineConfiguration.createStandaloneProcessEngineConfiguration()
  ...
  .buildProcessEngine();
```

The usecases for standalone transaction management are situations where the process engine does not
have to integrate with other transactional resources such as secondary datasources or messaging
systems.

{{< note title="" class="info" >}}
  In the tomcat distribution the process engine is configured using standalone transaction management.
{{< /note >}}


## Transaction Manager Integration

The process engine can be configured to integrate with a transaction manager (or transaction
management systems). Out of the box, the process engine supports integration with Spring and JTA
transaction management. More information can be found in the following chapters:

* [Section on Spring Transaction Management][tx-spring]
* [Section on JTA Transaction Management][tx-jta]

The usecase for transaction manager integration are situations where the process engine needs to
integrate with

* transaction focused programming models such as Java EE or Spring (think about transaction scoped
  JPA entity managers in Java EE),
* other transactional resources such as secondary datasources, messaging systems or other
  transactional middleware like the web services stack.

[tx-spring]: {{< relref "user-guide/spring-framework-integration/index.md#spring-transaction-integration" >}}
[tx-jta]: {{< relref "user-guide/cdi-java-ee-integration/index.md" >}}
