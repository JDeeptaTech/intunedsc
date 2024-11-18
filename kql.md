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
``` yaml
- name: Map network drive and copy file on Windows Server
  hosts: windows
  gather_facts: no
  vars:
    share_path: "\\\\network\\share\\path"  # Windows network share path
    drive_letter: "Z:"                      # Drive letter to assign for the mapped drive
    username: "your_username"               # Username for network share authentication
    password: "your_password"               # Password for network share authentication
    file_to_copy: "file_from_share.txt"     # File name to copy from share
    destination_path: "C:\\destination"     # Destination path on Windows server

  tasks:
    - name: Map network drive
      ansible.windows.win_command: |
        net use {{ drive_letter }} {{ share_path }} /user:{{ username }} {{ password }}
      register: drive_map_result
      ignore_errors: yes

    - name: Verify drive mapping
      ansible.builtin.fail:
        msg: "Failed to map network drive"
      when: drive_map_result.rc != 0

    - name: Copy file from network share
      ansible.windows.win_copy:
        src: "{{ drive_letter }}\\{{ file_to_copy }}"
        dest: "{{ destination_path }}\\{{ file_to_copy }}"
    
    - name: Unmap network drive
      ansible.windows.win_command: "net use {{ drive_letter }} /delete"
      ignore_errors: yes

```
