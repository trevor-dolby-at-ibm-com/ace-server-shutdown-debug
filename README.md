# ace-server-shutdown-debug
Examples of flows and commands to illustrate debugging servers having issues shutting down

- ESQL SLEEP() is interrupted by shutdown
- mqsireportproperties ACEv12_Broker -e default -o FlowThreadReporter/IgnoreLocks -r
- Broker shutdown disables thread pools
- mqsideploy causes the right sort of hang
- kill -3 produces a javacore but it's often hard to see where in a flow the hang is occurring as the native stack may be incomplete

Add sequence, flow picture, stdout/stderr from FTR, etc
