---
title: Windows.Remediation.Quarantine
hidden: true
tags: [Client Artifact]
---

**Apply quarantine via Windows local IPSec policy**

- By default the current client configuration is applied as an exclusion
using resolved IP address at time of application.
- A configurable lookup table is also used to generate additional entries
using the same syntax as netsh ipsec configuration.
  - DNS and DHCP are
entires here allowed by default.
- An optional MessageBox may also be configured to alert all logged in users.
  - The message will be truncated to 256 characters.
- After policy application, connection back to the Velociraptor
frontend is tested and the policy removed if connection unavailable.
- To remove policy, select the RemovePolicy checkbox.
- To update policy, simply rerun the artifact.

NOTE:

- Remember DNS resolution may change. It is highly recommended to plan
policy accordingly and not rely on DNS lookups.
- Local IPSec policy can not be applied when Domain IPSec policy
is already enforced. Please configure at GPO level in this case.


```yaml
name: Windows.Remediation.Quarantine
description: |
      **Apply quarantine via Windows local IPSec policy**

      - By default the current client configuration is applied as an exclusion
      using resolved IP address at time of application.
      - A configurable lookup table is also used to generate additional entries
      using the same syntax as netsh ipsec configuration.
        - DNS and DHCP are
      entires here allowed by default.
      - An optional MessageBox may also be configured to alert all logged in users.
        - The message will be truncated to 256 characters.
      - After policy application, connection back to the Velociraptor
      frontend is tested and the policy removed if connection unavailable.
      - To remove policy, select the RemovePolicy checkbox.
      - To update policy, simply rerun the artifact.

      NOTE:

      - Remember DNS resolution may change. It is highly recommended to plan
      policy accordingly and not rely on DNS lookups.
      - Local IPSec policy can not be applied when Domain IPSec policy
      is already enforced. Please configure at GPO level in this case.

author: Matt Green - @mgreen27

reference:
  - https://mgreen27.github.io/posts/2020/07/23/IPSEC.html

required_permissions:
  - EXECVE

precondition: SELECT OS From info() where OS = 'windows'

parameters:
  - name: PolicyName
    default: "VelociraptorQuarantine"
  - name: RuleLookupTable
    type: csv
    default: |
        Action,SrcAddr,SrcMask,SrcPort,DstAddr,DstMask,DstPort,Protocol,Mirrored,Description
        Permit,me,,0,any,,53,udp,yes,DNS
        Permit,me,,0,any,,53,tcp,yes,DNS TCP
        Permit,me,,68,any,,67,udp,yes,DHCP
        Block,any,,,any,,,,yes,All other traffic
  - name: MessageBox
    description: |
        Optional message box notification to send to logged in users. 256
        character limit.
  - name: RemovePolicy
    type: bool
    description: Tickbox to remove policy.

sources:
    - query: |

        // If a MessageBox configured truncate to 256 character limit
        LET MessageBox <= parse_string_with_regex(
                  regex='^(?P<Message>.{0,255}).*',
                  string=MessageBox).Message

        // Normalise Action
        LET normalise_action(Action)=if(condition= lowcase(string=Action)= 'permit',
              then= 'Permit',
              else= if(condition= lowcase(string=Action)= 'block',
                  then= 'Block'))

        // extract configurable policy from lookuptable
        LET configurable_policy <= SELECT
                  normalise_action(Action=Action) AS Action,
                  SrcAddr,SrcMask,SrcPort,
                  DstAddr,DstMask,DstPort,
                  Protocol,Mirrored,Description
              FROM RuleLookupTable

        // Parse a URL to get domain name.
        LET get_domain(URL) = parse_string_with_regex(
             string=URL, regex='^https?://(?P<Domain>[^:/]+)').Domain

        // Parse a URL to get the port
        LET get_port(URL) = if(condition= URL=~"https://[^:]+/", then="443",
              else=if(condition= URL=~"http://[^:]+/", then="80",
              else=parse_string_with_regex(string=URL,
                  regex='^https?://[^:/]+(:(?P<Port>[0-9]*))?/').Port))

        // extract Velociraptor config for policy
        LET extracted_config <= SELECT * FROM foreach(
                  row=config.server_urls,
                  query={
                      SELECT
                          'Permit' AS Action,
                          'me' AS SrcAddr,
                          '' As SrcMask,
                          '0' AS SrcPort,
                          get_domain(URL=_value) AS DstAddr,
                          '' As DstMask,
                          get_port(URL=_value) AS DstPort,
                          'tcp' AS Protocol,
                          'yes' AS Mirrored,
                          'VelociraptorFrontEnd' AS Description,
                          _value AS URL
                      FROM scope()
                  })

        // build policy with extracted config and lookuptable
        LET policy <= SELECT *
              FROM chain(
                  a=extracted_config,
                  b=configurable_policy
              )
              WHERE Action =~ '^(Permit|Block)$'

        // Removes empty options from the command line
        LET clean_cmdline(CMD) = filter(list=CMD, regex='^(\\w+|\\w+=.+)$')

        LET delete_cmdline = clean_cmdline(
             CMD=('netsh','ipsec','static','delete','policy', 'name=' + PolicyName))

        LET create_cmdline = clean_cmdline(
             CMD=('netsh','ipsec','static','add','policy', 'name=' + PolicyName))

        LET action_cmdline(Action) = clean_cmdline(
             CMD=('netsh','ipsec','static','add','filteraction',
                  'name=' + PolicyName + ' ' + Action + 'Action',
                  'action=' + Action))

        LET rule_cmdline(Action) = clean_cmdline(
             CMD=('netsh','ipsec','static','add','rule',
                  'name=' + PolicyName + ' ' + Action + 'Rule',
                  'policy=' + PolicyName,
                  'filterlist=' + PolicyName + ' ' + Action + 'FilterList',
                  'filteraction=' + PolicyName + ' ' + Action + 'Action'))

        LET enable_cmdline = clean_cmdline(
             CMD=('netsh','ipsec','static','set','policy',
                   'name=' + PolicyName, 'assign=y'))

        // Emit the message if no output is emitted, otherwise emit the output.
        LET combine_results(Stdout, Stderr, ReturnCode, Message) = if(
              condition=Stdout =~ "[^\\s]", then=Stdout,
              else= if(condition=Stderr =~ "[^\\s]", then=Stderr,
              else= if(condition= ReturnCode=0,
                    then=Message )))

        // delete old or unwanted policy
        LET delete_policy = SELECT
              timestamp(epoch=now()) as Time,
              PolicyName + ' IPSec policy removed.' AS Result
          FROM execve(argv=delete_cmdline, length=10000)

        // first step is creating IPSec policy
        LET create_policy = SELECT
              timestamp(epoch=now()) as Time,
              combine_results(Stdout=Stdout, Stderr=Stderr,
                  ReturnCode=ReturnCode,
                  Message=PolicyName + ' IPSec policy created.') AS Result
          FROM execve(argv=create_cmdline, length=10000)

        LET entry_cmdline(Action, SrcAddr, SrcPort, SrcMask,
                   DstAddr, DstPort, DstMask, Protocol,
                   Mirrored, Description) = clean_cmdline(
              CMD=('netsh','ipsec','static','add','filter',
                   format(format='filterlist=%s %sFilterList', args=[PolicyName, Action]),
                   format(format='srcaddr=%v', args=SrcAddr),
                   format(format='srcmask=%v', args=SrcMask),
                   format(format='srcport=%v', args=SrcPort),
                   format(format='dstaddr=%v', args=DstAddr),
                   format(format='dstmask=%v', args=DstMask),
                   format(format='dstport=%v', args=DstPort),
                   format(format='protocol=%v', args=Protocol),
                   format(format='mirrored=%v', args=Mirrored),
                   format(format='description=%v', args=Description)))

        // second step is to create policy filters
        LET create_filters = SELECT * FROM foreach(row=policy,
                  query={
                      SELECT
                          timestamp(epoch=now()) as Time,
                          combine_results(Stdout=Stdout, Stderr=Stderr,
                               ReturnCode=ReturnCode,
                               Message='Entry added: ' +
                                 join(array=entry_cmdline(Action=Action,
                                   SrcAddr=SrcAddr, SrcPort=SrcPort, SrcMask=SrcMask,
                                   DstAddr=DstAddr, DstPort=DstPort, DstMask=DstMask,
                                   Protocol=Protocol, Mirrored=Mirrored,
                                   Description=Description), sep=" ")) AS Result
                      FROM execve(argv=entry_cmdline(Action=Action,
                                   SrcAddr=SrcAddr, SrcPort=SrcPort, SrcMask=SrcMask,
                                   DstAddr=DstAddr, DstPort=DstPort, DstMask=DstMask,
                                   Protocol=Protocol, Mirrored=Mirrored,
                                   Description=Description), length=10000)
                  })

        // third step is to create policy filter actions
        LET create_actions = SELECT * FROM foreach(
                  row= {
                      SELECT Action
                      FROM policy
                      GROUP BY Action
                  },
                  query={
                      SELECT
                          timestamp(epoch=now()) as Time,
                          combine_results(Stdout=Stdout, Stderr=Stderr,
                               ReturnCode=ReturnCode,
                               Message='FilterAction added: ' +
                                 join(array=action_cmdline(Action=Action), sep=" ")) AS Result
                      FROM execve(argv=action_cmdline(Action=Action), length=10000)
                  })

        // fourth step combines action lists and actions in a Rule
        LET create_rules = SELECT * FROM foreach(
                  row= {
                      SELECT Action
                      FROM policy
                      GROUP BY Action
                  },
                  query={
                      SELECT
                          timestamp(epoch=now()) as Time,
                          combine_results(Stdout=Stdout, Stderr=Stderr,
                               ReturnCode=ReturnCode,
                               Message='Rule added: ' +
                                 join(array=rule_cmdline(Action=Action), sep=" ")) AS Result
                      FROM execve(argv=rule_cmdline(Action=Action), length=10000)
                  })

        // fith step is to enable our IPSec policy
        LET enable_policy = SELECT
              timestamp(epoch=now()) as Time,
              combine_results(Stdout=Stdout, Stderr=Stderr,
                  ReturnCode=ReturnCode,
                  Message=PolicyName + ' IPSec policy applied.') AS Result
              FROM execve(argv=enable_cmdline, length=10000)

        // test connection to a frontend server
        LET test_connection = SELECT * FROM foreach(
                  row={
                      SELECT * FROM policy
                      WHERE Description = 'VelociraptorFrontEnd'
                  },
                  query={
                      SELECT *
                          Url,
                          response
                      FROM
                          http_client(url='https://' + DstAddr + ':' + DstPort + '/server.pem',
                              disable_ssl_security='TRUE')
                      WHERE Response = 200
                      LIMIT 1
                  })

        // final check to keep or remove policy
        LET final_check = SELECT * FROM if(condition= test_connection,
                  then={
                      SELECT
                          timestamp(epoch=now()) as Time,
                          if(condition=MessageBox,
                              then= PolicyName + ' connection test successful. MessageBox sent.',
                              else= PolicyName + ' connection test successful.'
                              ) AS Result
                      FROM if(condition=MessageBox,
                          then= {
                              SELECT * FROM execve(argv=['msg','*',MessageBox])
                          },
                          else={
                              SELECT * FROM scope()
                          })
                  },
                  else={
                      SELECT
                          timestamp(epoch=now()) as Time,
                          PolicyName + ' failed connection test. Removing IPSec policy.' AS Result
                      FROM delete_policy
                  })

        // Execute content
        SELECT * FROM if(condition=RemovePolicy,
                  then=delete_policy,
                  else={
                      SELECT * FROM chain(
                          a=delete_policy,
                          b=create_policy,
                          c=create_filters,
                          d=create_actions,
                          e=create_rules,
                          g=enable_policy,
                          h=final_check)
                  })

```
