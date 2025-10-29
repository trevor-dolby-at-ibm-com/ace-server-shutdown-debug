# ace-server-shutdown-debug

This repo contains an example flow that will sleep upon startup in order to cause flow shutdown 
to hang. The goal is to allow experiments with various debugging strategies and commands that can
help show where issues are occurring.

One important command is
```
mqsireportproperties ACEv12_Broker -e default -o FlowThreadReporter/IgnoreLocks -r
```
as it will bypass the usual REST API locking and cause the server to dump the state of the various
flows (including the one that is failing to shut down). See [flow-thread-reporter](#flow-thread-reporter)
for details.

Using `kill -3` on a DataFlowEngine is also helpful if the flow is stuck in a Java call as it causes the 
DataFlowEngine to produce a javacore, but will not always show which message flow node is calling Java.
See [javacore-limitations](#javacore-limitations) for details.

## Problems during shutdown

ACE message flows will normally try to shut down gracefully, completing in-flight work before exiting.
This can be a problem if flows get stuck during message flow operation, as it can prevent shutdown and
in some cases cause issues with other administrative activities: if a flow is in the process of being
redeployed, the server's administration REST API threads will be busy with the redeploy and will not
normally process any other work. This can lead to commands timing out and the server generally appearing
unresponsive.

The application in this repo looks like this:

![flow](/ShutdownTestApp/HangOnTimer.png)

and prints to the console (syslog for node-associated servers) on startup
```
2025-10-29 14:19:27.983292: BIP3051E: Error message '

Sleeping for 1800 seconds


' from trace node 'HangOnTimer.Trace'.
```

The flow uses Java to implement the sleep call because ESQL SLEEP() is shutdown-aware and will wake
up if the flow is being shut down due to redeploy or server shutdown. 

Server shutdown differs from redeploy in that the admin REST API threads have shut down early in the
server shutdown process, so REST API calls to the flow thread reporter (for example) are not possible.
The admin REST API threads will be running during a flow redeploy, so in that case it is possible to
call the flow thread reporter as long as the query is told to ignore the REST API locks.

## Flow Thread Reporter

When the `mqsireportproperties ACEv12_Broker -e default -o FlowThreadReporter/IgnoreLocks -r` command
is run, it will send the request to the DataFlowEngine in a way that skips the usual admin REST API
locking, but a consequence of this is that the output is written to stdout for the server and is not
returned back to the command:
```
$ mqsireportproperties ACEv12_Broker -e default -o FlowThreadReporter/IgnoreLocks -r

Avoiding deploy chain locks and writing report to stdout for the integration server at /var/mqsi/components/ACEv12_Broker/servers/default/stdout; examine the system event log to find a BIP2889 acknowledgement and see integration server stdout log for flow stack and history information.

BIP8071I: Successful command completion.
```
The output is visible in the stdout file named in the response (`/var/mqsi/components/ACEv12_Broker/servers/default/stdout`):
```
FlowThreadReporter command received; dump follows:
applicationCount: 1
  ShutdownTestApp
    HangOnTimer
      tid: 100418
        currentFlowMessageId: (00018842-6902853C-00000001)
        stack:
          Timeout Notification[ComIbmTimeoutNotificationNode],FlowThreadPool,0,2025-10-29T21:21:00Z,(00000000-00000000-00000000)
          Trace[ComIbmTraceNode],Timeout Notification/out,0,2025-10-29T21:21:00Z,(00018842-6902853C-00000001)
          Java Compute[ComIbmJavaComputeNode],Trace/out,1,2025-10-29T21:21:00Z,(00018842-6902853C-00000001)
        history:
          Timeout Notification[ComIbmTimeoutNotificationNode],FlowThreadPool,0,2025-10-29T21:21:00Z,0,(00000000-00000000-00000000)
          Trace[ComIbmTraceNode],Timeout Notification/out,0,2025-10-29T21:21:00Z,0,(00018842-6902853C-00000001)
          Java Compute[ComIbmJavaComputeNode],Trace/out,1,2025-10-29T21:21:00Z,0,(00018842-6902853C-00000001)
```
and the last item on the stack is the currently-running message flow node, which is presumably the one 
that is stuck. If the flow is stuck in a loop, then that will be visible from the history section for the flow.


## Walkthrough

This walkthrough (with matching syslog below) shows the server getting stuck on the flow redeploy, the
mqsireportproperties command timing out, and then the successful use of `FlowThreadReporter/IgnoreLocks` 
to show the location of the hang:

```
tdolby@IBM-7NGKB54:~$ mqsistart ACEv12_Broker
BIP8873I: Starting the component verification for component 'ACEv12_Broker'.
BIP8096I: Successful command initiation, check the system log to ensure that the component started without problem and that it continues to run without problem.
tdolby@IBM-7NGKB54:~$ date ; mqsideploy ACEv12_Broker -e default -a /mnt/c/Users/TrevorDolby/git/ace-server-shutdown-debug/ShutdownTestApp/ShutdownTestApp.bar
Wed Oct 29 16:20:59 CDT 2025
BIP1039I: Deploying BAR file '/mnt/c/Users/TrevorDolby/git/ace-server-shutdown-debug/ShutdownTestApp/ShutdownTestApp.bar' to integration node 'ACEv12_Broker' (integration server 'default') ...
BIP9332I: Application 'ShutdownTestApp' has been created successfully.
BIP1092I: The deployment request was processed successfully.
tdolby@IBM-7NGKB54:~$ date ; mqsideploy ACEv12_Broker -e default -a /mnt/c/Users/TrevorDolby/git/ace-server-shutdown-debug/ShutdownTestApp/ShutdownTestApp.bar
Wed Oct 29 16:21:09 CDT 2025
BIP1039I: Deploying BAR file '/mnt/c/Users/TrevorDolby/git/ace-server-shutdown-debug/ShutdownTestApp/ShutdownTestApp.bar' to integration node 'ACEv12_Broker' (integration server 'default') ...
BIP8286E: The administration request did not complete within the required time limit.
No response was been received within the required time limit for the administration request.
If the request is updating the deployed configuration it may complete successfully. Check the Event Log and/or Activity Log to determine if the administration request was successful. Increasing the timeout parameter will allow the command to wait for a longer period of time.
tdolby@IBM-7NGKB54:~$ date ; mqsireportproperties ACEv12_Broker -e default -o FlowThreadReporter -r
Wed Oct 29 16:26:36 CDT 2025

BIP8286E: The administration request did not complete within the required time limit.
No response was been received within the required time limit for the administration request.
If the request is updating the deployed configuration it may complete successfully. Check the Event Log and/or Activity Log to determine if the administration request was successful. Increasing the timeout parameter will allow the command to wait for a longer period of time.
tdolby@IBM-7NGKB54:~$ date ; mqsireportproperties ACEv12_Broker -e default -o FlowThreadReporter/IgnoreLocks -r
Wed Oct 29 16:31:53 CDT 2025

Avoiding deploy chain locks and writing report to stdout for the integration server at /var/mqsi/components/ACEv12_Broker/servers/default/stdout; examine the system event log to find a BIP2889 acknowledgement and see integration server stdout log for flow stack and history information.

BIP8071I: Successful command completion.
tdolby@IBM-7NGKB54:~$ cat /var/mqsi/components/ACEv12_Broker/servers/default/stdout
Redirecting stdout to /var/mqsi/components/ACEv12_Broker/servers/default/stdout
Redirecting stderr to /var/mqsi/components/ACEv12_Broker/servers/default/stderr
2025-10-29 16:20:49.812      1 Using jms interface URL: file:/opt/ace-12.0.12.17/server/../common/classes/jms2.0/jms.jar
FlowThreadReporter command received; dump follows:
applicationCount: 1
  ShutdownTestApp
    HangOnTimer
      tid: 100418
        currentFlowMessageId: (00018842-6902853C-00000001)
        stack:
          Timeout Notification[ComIbmTimeoutNotificationNode],FlowThreadPool,0,2025-10-29T21:21:00Z,(00000000-00000000-00000000)
          Trace[ComIbmTraceNode],Timeout Notification/out,0,2025-10-29T21:21:00Z,(00018842-6902853C-00000001)
          Java Compute[ComIbmJavaComputeNode],Trace/out,1,2025-10-29T21:21:00Z,(00018842-6902853C-00000001)
        history:
          Timeout Notification[ComIbmTimeoutNotificationNode],FlowThreadPool,0,2025-10-29T21:21:00Z,0,(00000000-00000000-00000000)
          Trace[ComIbmTraceNode],Timeout Notification/out,0,2025-10-29T21:21:00Z,0,(00018842-6902853C-00000001)
          Java Compute[ComIbmJavaComputeNode],Trace/out,1,2025-10-29T21:21:00Z,0,(00018842-6902853C-00000001)
```

Matching syslog (may need to scroll to the right):
```
2025-10-29T16:20:46.834374-05:00 IBM-7NGKB54 ACE[99941]: IBM App Connect Enterprise v1201217 (ACEv12_Broker) [Thread 99941] (Msg 1/1) BIP2001I: The IBM App Connect Enterprise service has started at version 1201217; process ID 99946.
2025-10-29T16:20:48.813387-05:00 IBM-7NGKB54 systemd-resolved[169]: Clock change detected. Flushing caches.
2025-10-29T16:20:49.267872-05:00 IBM-7NGKB54 ACE[99946]: IBM App Connect Enterprise v1201217 (ACEv12_Broker) [Thread 99946] (Msg 1/1) BIP2866I: IBM App Connect Enterprise administration security is inactive.
2025-10-29T16:20:49.290460-05:00 IBM-7NGKB54 ACE[99946]: IBM App Connect Enterprise v1201217 (ACEv12_Broker) [Thread 99969] (Msg 1/1) BIP3132I: The HTTP Listener has started listening on port '4414' for 'RestAdmin http' connections.
2025-10-29T16:20:49.404075-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 99982] (Msg 1/1) BIP2208I: Integration server (64) started: process '99982'; thread '99982'; additional information: integrationNodeName 'ACEv12_Broker' (operation mode 'advanced'); integrationServerUUID '00000000-0000-0000-0000-000000000000'; integrationServerLabel 'default'; queueManagerName 'ACEv12_QM'; trusted 'false'; userId 'tdolby'; migrationNeeded 'false'; integrationNodeUUID '86badf54-b4fc-11f0-a16d-7f0001010000'; filePath '/opt/ace-12.0.12.17/server'; workPath '/var/mqsi'; ICU Converter Path ''; ordinality '1'.
2025-10-29T16:20:49.432491-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 99982] (Msg 1/1) BIP9905I: Initializing resource managers.
2025-10-29T16:20:55.279063-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 99982] (Msg 1/1) BIP2152I: Configuration message received.
2025-10-29T16:20:55.279326-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 99982] (Msg 1/1) BIP2153I: About to 'Start' an integration server.
2025-10-29T16:20:55.283834-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 99982] (Msg 1/1) BIP9906I: Reading deployed resources.
2025-10-29T16:20:55.319585-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 99982] (Msg 1/1) BIP2154I: Integration server finished with Configuration message.
2025-10-29T16:21:00.397548-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100213] (Msg 1/1) BIP2152I: Configuration message received.
2025-10-29T16:21:00.404181-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100213] (Msg 1/1) BIP2155I: About to 'Initialize' the deployed resource 'ShutdownTestApp' of type 'Application'.
2025-10-29T16:21:00.547540-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100213] (Msg 1/1) BIP2155I: About to 'Start' the deployed resource 'ShutdownTestApp' of type 'Application'.
2025-10-29T16:21:00.547948-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100213] (Msg 1/1) BIP2269I: Deployed resource 'HangOnTimer' (uuid='HangOnTimer',type='MessageFlow') started successfully.
2025-10-29T16:21:00.548153-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100213] (Msg 1/1) BIP9332I: Application 'ShutdownTestApp' has been created successfully.
2025-10-29T16:21:00.548689-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100213] (Msg 1/1) BIP9326I: The source 'ShutdownTestApp.bar' has been successfully deployed.
2025-10-29T16:21:00.548805-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100418] (Msg 1/1) BIP3051E: Error message '  Sleeping for 1800 seconds   ' from trace node 'HangOnTimer.Trace'.
2025-10-29T16:21:00.548894-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100213] (Msg 1/1) BIP2154I: Integration server finished with Configuration message.
2025-10-29T16:21:10.072240-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100217] (Msg 1/1) BIP2152I: Configuration message received.
2025-10-29T16:21:10.075007-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100217] (Msg 1/1) BIP2155I: About to 'Stop' the deployed resource 'ShutdownTestApp' of type 'Application'.
2025-10-29T16:31:53.621799-05:00 IBM-7NGKB54 ACE[99982]: IBM App Connect Enterprise v1201217 (ACEv12_Broker.default) [Thread 100220] (Msg 1/1) BIP2889W: FlowThreadReporter information dump request by user; file location is '/var/mqsi/components/ACEv12_Broker/servers/default/stdout'.
```

## javacore limitations

While `kill -3 <DFE PID>` produces output regardless of the state of the admin REST API thread pool, 
the output (a javacore file in `/var/mqsi/common/errors`) may not show all the necessary information. 

In the case of the flow in this repo, the javacore stack correctly shows the location in the Java
code (HangOnTimer_JavaCompute.java:26) but does not show the message flow node (see below). This is
not a problem for a flow as simple as the demo flow, but for complex flows it could be an issue.

```
3XMTHREADINFO      "Thread-14" J9VMThread:0x0000000000BB3D00, omrthread_t:0x0000780514018668, java/lang/Thread:0x00000000FFE168B8, state:CW, rawStateValue:0x8, prio=5
3XMJAVALTHREAD            (java/lang/Thread getId:0x36, isDaemon:false)
3XMJAVALTHRCCL            com/ibm/broker/plugin/MbNonDelegatingJavaResourceClassLoader(0x00000000F0376C58)
3XMTHREADINFO1            (native thread ID:0x17AC3, native priority:0x5, native policy:UNKNOWN, vmstate:CW, vm thread flags:0x00000481)
3XMTHREADINFO2            (native stack address range from:0x000078073C0FF000, to:0x000078073C27F000, size:0x180000)
3XMCPUTIME               CPU usage total: 0.024484818 secs, current category="Application"
3XMHEAPALLOC             Heap bytes allocated since last GC cycle=0 (0x0)
3XMTHREADINFO3           Java callstack:
4XESTACKTRACE                at java/lang/Thread.sleepImpl(Native Method)
4XESTACKTRACE                at java/lang/Thread.sleep(Thread.java:973)
4XESTACKTRACE                at java/lang/Thread.sleep(Thread.java:956)
4XESTACKTRACE                at ace/demo/shutdown/HangOnTimer_JavaCompute.evaluate(HangOnTimer_JavaCompute.java:26)
4XESTACKTRACE                at com/ibm/broker/javacompute/MbRuntimeJavaComputeNode.evaluate(MbRuntimeJavaComputeNode.java:809)
4XESTACKTRACE                at com/ibm/broker/plugin/MbNode.evaluate(MbNode.java:2228)
3XMTHREADINFO3           Native callstack:
4XENATIVESTACK                (0x00007807EC291CD2 [libj9prt29.so+0x5dcd2])
4XENATIVESTACK                (0x00007807EC25AF97 [libj9prt29.so+0x26f97])
4XENATIVESTACK                (0x00007807EC2921AE [libj9prt29.so+0x5e1ae])
4XENATIVESTACK                (0x00007807EC25AF97 [libj9prt29.so+0x26f97])
4XENATIVESTACK                (0x00007807EC291B57 [libj9prt29.so+0x5db57])
4XENATIVESTACK                (0x00007807EC28E53C [libj9prt29.so+0x5a53c])
4XENATIVESTACK                (0x00007807F9045330 [libc.so.6+0x45330])
4XENATIVESTACK                (0x00007807F9098D71 [libc.so.6+0x98d71])
4XENATIVESTACK               pthread_cond_timedwait+0x23e (0x00007807F909BC8E [libc.so.6+0x9bc8e])
4XENATIVESTACK               omrthread_sleep_interruptable+0x111 (0x00007807F54124E1 [libj9thr29.so+0x94e1])
4XENATIVESTACK                (0x00007807EC3B130A [libj9vm29.so+0xec30a])
4XENATIVESTACK                (0x00007807EC2E3C36 [libj9vm29.so+0x1ec36])
4XENATIVESTACK                (0x00007807EC2D8392 [libj9vm29.so+0x13392])
4XENATIVESTACK                (0x00007807EC3BBB32 [libj9vm29.so+0xf6b32])
```
