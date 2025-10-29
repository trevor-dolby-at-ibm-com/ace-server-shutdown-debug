# ace-server-shutdown-debug

This repo contains an example flow that will sleep upon startup in order to cause flow shutdown 
to hang. The goal is to allow experiments with various debugging strategies and commands that can
help show where issues are occurring.

An important command is
```
mqsireportproperties ACEv12_Broker -e default -o FlowThreadReporter/IgnoreLocks -r
```
as it will bypass the usual REST API locking and cause the server to dump the state of the various
flows (including the one that is failing to shut down). 

Using `kill -3` on a DataFlowEngine is also helpful if the flow is stuck in a Java call, but will
not always show which message flow node is calling Java.


- ESQL SLEEP() is interrupted by shutdown
- mqsireportproperties ACEv12_Broker -e default -o FlowThreadReporter/IgnoreLocks -r
- Broker shutdown disables thread pools
- mqsideploy causes the right sort of hang
- kill -3 produces a javacore but it's often hard to see where in a flow the hang is occurring as the native stack may be incomplete

Add sequence, flow picture, stdout/stderr from FTR, etc

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
