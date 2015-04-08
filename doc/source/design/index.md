---
layout: page
title: Design
countheads: true
toc: true
comments: true
---

Remote Execution Technology
===========================

User Stories
------------

- As a user I want to run commands in parallel across large number of
hosts

- As a user I want to run commands on a host in a different network
  segment (the host doesn't see the Foreman server/the Foreman server
  doesn't see the host directly)

- As a user I want to manage a host without installing an agent on it
  (just plain old ssh)

- As a community user I want to already existing remote execution
  technologies in combination with the Foreman

Design
------

### Ssh Single Host Push

{% plantuml %}
actor User
participant "Foreman Server" as Foreman
participant "Foreman Proxy" as Proxy
participant "Host" as Host

autonumber
User -> Foreman : UserCommand
Foreman -> Proxy : ProxyCommand
Proxy -> Host : SshCommand
Activate Host
Host --> Proxy : ProgressReport[1, Running]
Host --> Proxy : ProgressReport[2, Running]
Proxy --> Foreman : AccumulatedProgressReport[1, Running]
Host --> Proxy : ProgressReport[3, Running]
Host --> Proxy : ProgressReport[4, Finished]
Deactivate Host
Proxy --> Foreman : AccumulatedProgressReport[2, Finished]
{% endplantuml %}

UserCommand:

  * hosts: [host.example.com]
  * template: install-packages-ssh
  * input: { packages: ['vim-X11'] }

ProxyCommand:

  * host: host.example.com
  * provider: ssh
  * input: "yum install -y vim-X11"

SshCommand:

  * host: host.example.com
  * input: "yum install -y vim-X11"

ProgressReport[1, Running]:

  * output: "Resolving depednencies"

ProgressReport[2, Running]:

  * output: "installing libXt"

AccumulatedProgressReport[1, Running]:

  * output: { stdout: "Resolving depednencies\ninstalling libXt" }

ProgressReport[3, Running]:

  * output: "installing vim-X11"

ProgressReport[4, Finished]:

  * output: "operation finished successfully"
  * exit_code: 0

AccumulatedProgressReport[2, Finished]:

  * output: { stdout: "installing vim-X11\noperation finished successfully", exit_code: 0 }
  * success: true

### Ssh Single Host Pull

{% plantuml %}
actor User
participant "Foreman Server" as Foreman
participant "Foreman Proxy" as Proxy
participant "Host" as Host

autonumber
User -> Foreman : UserCommand
group Optional
  Foreman -> Proxy : EnforceCheckIn
  Proxy -> Host : EnforceCheckIn
end
Host -> Proxy : CheckIn
Proxy -> Foreman : CheckIn
Foreman --> Proxy : ProxyCommand
Proxy --> Host : Script
Activate Host
Host -> Proxy : ProgressReport[1, Running]
Host -> Proxy : ProgressReport[2, Running]
Proxy -> Foreman : AccumulatedProgressReport[1, Running]
Host -> Proxy : ProgressReport[3, Running]
Host -> Proxy : ProgressReport[4, Finished]
Deactivate Host
Proxy -> Foreman : AccumulatedProgressReport[2, Finished]
{% endplantuml %}

UserCommand:

  * hosts: [host.example.com]
  * template: install-packages-ssh
  * input: { packages: ['vim-X11'] }

ProxyCommand:

  * host: host.example.com
  * provider: ssh
  * input: "yum install -y vim-X11"

Script:

  * input: "yum install -y vim-X11"

ProgressReport[1, Running]:

  * output: "Resolving depednencies"

ProgressReport[2, Running]:

  * output: "installing libXt"

AccumulatedProgressReport[1, Running]:

  * output: { stdout: "Resolving depednencies\ninstalling libXt" }

ProgressReport[3, Running]:

  * output: "installing vim-X11"

ProgressReport[4, Finished]:

  * output: "operation finished successfully"
  * exit_code: 0

AccumulatedProgressReport[2, Finished]:

  * output: { stdout: "Resolving depednencies\ninstalling libXt", exit_code: 0 }
  * success: true

### Ssh Multi Host

{% plantuml %}
actor User
participant "Foreman Server" as Foreman
participant "Foreman Proxy" as Proxy
participant "Host 1" as Host1
participant "Host 2" as Host2

autonumber
User -> Foreman : UserCommand
Foreman -> Proxy : ProxyCommand[host1]
Foreman -> Proxy : ProxyCommand[host2]
Proxy -> Host1 : SshCommand
Proxy -> Host2 : SshCommand
{% endplantuml %}

UserCommand:

  * hosts: *.example.com
  * template: install-packages-ssh
  * input: { packages: ['vim-X11'] }

ProxyCommand[host1]:

  * host: host-1.example.com
  * provider: ssh
  * input: "yum install -y vim-X11"

ProxyCommand[host2]:

  * host: host-2.example.com
  * provider: ssh
  * input: "yum install -y vim-X11"

{% info_block %}
we might want to optimize the communication between server and
the proxy (sending collection of ProxyCommands in bulk, as well as
the AccumulatedProgerssReports). That would could also be utilized
by the Ansible implementation, where there might be optimization
on the invoking the ansible commands at once (the same might apply
to mcollective). On the other hand, this is more an optimization,
not required to be implemented from the day one: but it's good to have
this in mind
{% endinfo_block %}

### MCollective Single Host

{% plantuml %}
actor User
participant "Foreman Server" as Foreman
participant "Foreman Proxy" as Proxy
participant "AMQP" as AMQP
participant "Host" as Host

autonumber
User -> Foreman : UserCommand
Foreman -> Proxy : ProxyCommand
Proxy -> AMQP : MCOCommand
AMQP -> Host : MCOCommand
Activate Host
Host --> AMQP : ProgressReport[Finished]
Deactivate Host
AMQP --> Proxy : ProgressReport[Finished]
Proxy --> Foreman : AccumulatedProgressReport[Finished]
{% endplantuml %}

UserCommand:

  * hosts: [host.example.com]
  * template: install-packages-mco
  * input: { packages: ['vim-X11'] }

ProxyCommand:

  * host: host.example.com
  * provider: mcollective
  * input: { agent: package, args: { package => 'vim-X11' } }

MCOCommand:

  * host: host.example.com
  * input: { agent: package, args: { package => 'vim-X11' } }

ProgressReport[Finished]:

  * output: [ {"name":"vim-X11","tries":1,"version":"7.4.160-1","status":0,"release":"1.el7"},
              {"name":"libXt","tries":1,"version":"1.1.4-6","status":0,"release":"1.el7"} ]

AccumulatedProgressReport[Finished]:

  * output: [ {"name":"vim-X11","tries":1,"version":"7.4.160-1","status":0,"release":"1.el7"},
              {"name":"libXt","tries":1,"version":"1.1.4-6","status":0,"release":"1.el7"} ]
  * success: true

### Ansible Single Host

{% plantuml %}
actor User
participant "Foreman Server" as Foreman
participant "Foreman Proxy" as Proxy
participant "Host" as Host

autonumber
User -> Foreman : UserCommand
Foreman -> Proxy : ProxyCommand
Proxy -> Host : AnsibleCommand
Activate Host
Host --> Proxy : ProgressReport[Finished]
Deactivate Host
Proxy --> Foreman : AccumulatedProgressReport[Finished]

{% endplantuml %}

UserCommand:

  * hosts: [host.example.com]
  * template: install-packages-ansible
  * input: { packages: ['vim-X11'] }

ProxyCommand:

  * host: host.example.com
  * provider: ansible
  * input: { module: yum, args: { name: 'vim-X11', state: installed } }

AnsibleCommand:

  * host: host.example.com
  * provider: ansible
  * input: { module: yum, args: { name: 'vim-X11', state: installed } }

ProgressReport[Finished]:

  * output: { changed: true,
              rc: 0,
              results: ["Resolving depednencies\ninstalling libXt\ninstalling vim-X11\noperation finished successfully"] }

AccumulatedProgressReport[Finished]:

  * output: { changed: true,
              rc: 0,
              results: ["Resolving depednencies\ninstalling libXt\ninstalling vim-X11\noperation finished successfully"] }
  * success: true


Command Preparation
===================

User Stories
------------

Design
------

Command Invocation
===================

User Stories
------------

Design
------

Command Execution
=================

User Stories
------------

- ?As a user I want to be able to specify default number of tries per command. # Command preparation?

- ?As a user I want to be able to specify default retry interval per command. # Command preparation?

- ?As a user I want to be able to specify default splay time per command. # Command preparation?

- As a user I want to be able to override default values like (number of tries, retry interval, splay time, ...) when I plan an execution of command.

- As a user I want to be able to specify infinite number of tries (until success/cancel) when I plan an execution of command.



- As a user I want to be able to cancel command which hasn't been started yet.

- As a user I want to be able to cancel command which is in progress. # some providers might not support this? therefore next user stories

- As a developer I want to specify whether cancellation of running commands is possible.

- As a user I want to see if I'm able to cancel the command.



- As a user I want to set timeout when I plan an execution of a command.

- ?As a user I want to setup default timeout per command. # Command preparation?

- As a user I want to override default timeout when I plan an execution of command.

- ?As a developer I want dynflow to support timeouts # unless it already supports it

Design
------

Class diagram of Foreman classes

{% plantuml %}

class Command {
  InstallPackage, Exec, RestartService

  // default values for command
  retry: integer
  retry_interval: integer
  timeout: integer
  splay: integer

  plan(target, input) - creates CommandExecution
}

class Host {
  get_proxy_with_feature(type)
}

class CommandExecution {
  command_id: integer
  target: n:m to host_groups/bookmarks/hosts
  input: $input_abstraction values clone

  state: $CommandState
  started_at: datetime
  canceled_at datetime
  provider: SSH | MCollective

  retry: integer
  retry_interval: integer
  timeout: integer
  splay: integer

  cancel()
}

abstract class ProxyCommand {
  command_execution_id: integer
  host_id: integer
  type: string
  state: $CommandState
  started_at: datetime
  canceled_at datetime
  timeout_at datetime
  tried_count: integer
  cancel()
  {abstract} support_cancel?()
  {abstract} proxy_endpoint()
  plan()
}

class SSHProxyCommand {
  {static} support_cancel?()
  proxy_endpoint():string
}

class MCollectiveProxyCommand {
  {static} support_cancel?()
  proxy_endpoint():string
}

enum CommandState {
PLANNED
STARTED
FINISHED
}

Command <- CommandExecution
CommandExecution <-- ProxyCommand
Host <-- ProxyCommand

ProxyCommand <|-- SSHProxyCommand
ProxyCommand <|-- MCollectiveProxyCommand

{% endplantuml %}

CommandExecution will be probably later replaced by Scheduler that
will schedule ProxyCommands (could be responsible for batch jobs,
retrying on failure, timeouts)

Open questions
--------------

Do we need anything extra to fulfill using system facts story? MCollective e.g. can use facter on every run or use
values stored on server (equals to facts we have in Foreman). I think we should take facts from Foreman rather
than runtime (different result than expected when planning, performance)

Reporting
=========

User Stories
------------

Design
------

Scheduling
==========

User Stories
------------

Design
------

Katello Workflow Integration
============================

User Stories
------------

Design
------

Security
========

User Stories
------------

Design
------

Orchestration
=============

User Stories
------------

Design
------