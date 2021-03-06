---

title: 'Process History View'
weight: 40

menu:
  main:
    identifier: "user-guide-cockpit-history-view"
    parent: "user-guide-cockpit"

---


{{< enterprise >}}
Please note that this feature is only included in the enterprise edition of the Camunda BPM platform, it is not available in the community edition.
{{< /enterprise >}}

At the top right of the Process Definition View and the Process Instance View, you can hit the History Button to access the history view.


# Process Definition History View

{{< img src="../img/cockpit-history-view-process-definition-history.png" title="Process Definition History" >}}

In the history view of the Process Definition you see an overview of all of the running and completed process instances. On the left side of the screen, a [filter][process-definition-view-filter] can be applied and you have the option of selecting to only see process instances in a specific state. Running and completed instances can be selected.

At the bottom of the screen you can also select the Job Log tab to see jobs related events of all process instances, including state, time, the corresponding activity and job ID, the type, configuration and message. You can also access the stracktrace of a failed job.


# Process Instance History View

{{< img src="../img/cockpit-history-view-process-instance-history.png" title="Process Instance History" >}}

In the history view of the process instance you see instance-specific information. On the left side of the screen, a [filter][process-definition-view-filter] can be applied and you have the option of selecting to only see process instances in specific states. Running, completed and canceled process instances can be viewed as well as task-specific activity states.

You can access various information regarding the specific instance by selecting the applicable tab at the bottom of the screen:

**Audit Log**
In the Audit Log you can find a detailed overview of the activities that took place within the process instance, including start time, end time, activity instance ID and the current state.

**Variables**
In the Variables tab you can see an overview of the variables used within the process instances, including the name, last value, variable type, scope and actions of the variables.

**Called Process Instances**
In the Called Process Instances tab you can find an overview of other process instances which were called by this specific process instance. You can see the Name of the called process instances, the process definition and the activity.

**Incidents**
In the Incidents tab you can find a listing of all incidents related to this process instance and the details thereof. This includes the message type, the time the incident was created, the end time, the actual activity, the cause process instance ID, the root cause process instance ID, the incident type and the current state.

**User Tasks**
In the User Tasks tab you can find an overview of all the user tasks related to this process instance and the details of the specific user tasks, such as the activity, the assignee, owner, creation date, completion date, the duration, due date, follow up date, the priority of the user task and the unique task ID. You can also see the user task log for each specific user task.

**Job Log**
In the Job Log tab you can find an overview of all job related events of this process instance and the details of the specific jobs, such as state, time, the corresponding activity and job ID, the type, configuration and message. You can also access the stracktrace of a failed job.


[process-definition-view-filter]: {{< relref "webapps/cockpit/process-definition-view.md#filter" >}}