```kql
AzureDiagnostics
| where Category in ("AzureFirewallNetworkRule", "AzureFirewallApplicationRule")
| extend Action = extract("Action: ([^,]+)", 1, msg_s),
         RuleCollection = extract("Rule Collection: ([^,]+)", 1, msg_s),
         SourceIP = extract("from ([^:]+)", 1, msg_s),
         Target = extract("to ([^:]+):", 1, msg_s),
         Protocol = extract("Protocol: ([^,]+)", 1, msg_s),
         TargetPort = extract("to .*:([0-9]+)", 1, msg_s)
| where ipv4_is_in_range(SourceIP, " ") 
| summarize PacketFlows = count() by SourceIP, Action, Target, Protocol, TargetPort
| order by PacketFlows desc

```
