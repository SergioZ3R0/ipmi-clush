# IPMI-CLUSH

`ipmi-clush` - Parallel IPMI Utility with ClusterShell Aggregation

**Author:** @Sergio Cabello Simón

**Target Subsystem:** Out-of-Band Management (BMC / IPMI Infrastructure)

- 1 [Overview](#Overview)
- 2 [Key Features](#Key-Features)
- 3 [Installation](#Installation)
- 4 [Syntax & Usage](#Syntax-%26-Usage)- 3.1 [Arguments](#Arguments)
- 5 [Use Cases & Production Examples](#Use-Cases-%26-Production-Examples)- 4.1 [Use Case 1: Cluster-Wide Out-of-Band User Auditing](#Use-Case-1%3A-Cluster-Wide-Out-of-Band-User-Auditing)
- 5.2 [Use Case 2: Mass Power Management & Telemetry Verification](#Use-Case-2%3A-Mass-Power-Management-%26-Telemetry-Verification)
- 6 [Error Handling & Debugging](#Error-Handling-%26-Debugging)


## Overview

`ipmi-clush` is a high-performance administration tool engineered to execute `ipmitool` commands across large compute cluster node ranges simultaneously.

While native tools like `clush` (ClusterShell) operate smoothly over SSH within the operating system layer, managing hardware at the Baseboard Management Controller (BMC) layer typically requires repetitive loop iterations. `ipmi-clush` bridges this gap by expanding cluster node ranges natively and **aggregating identical BMC outputs into a single consolidated view** using an MD5 hashing algorithm.

## Key Features

- **Native Node Range Expansion:** Parses standard cluster bracket notation (e.g., `node[01-50]`) and sequentially maps them to target `-bmc` interfaces.
- **Intelligent Output Deduplication:** Hashes responses in real-time to group identical outputs, vastly reducing stdout noise.
- **Dynamic Authentication Tracking:** Configures individual username and password injection on the fly, avoiding static or hardcoded script credentials.
- **Natural Sorting Representation: **Displays grouped host ranges utilizing semantic, compressed notation for rapid infrastructure health assessments.
- **True Parallel Execution: **Utilizes background subshells to query hundreds of BMCs simultaneously, cutting execution time to seconds.
- **ClusterShell Integration:** Natively resolves `@groups` defined in `/etc/clustershell/groups.conf` via the `nodeset` command.

## Installation

Since this is a private repository, ensure your environment has the proper Git credentials configured, then run the following one-liner to clone, deploy globally, and set execution permissions:

```
git clone -q [https://github.com/doitnowgroup/ipmi-clush.git](https://github.com/doitnowgroup/ipmi-clush.git) /tmp/ipmi-clush-tmp && sudo mv /tmp/ipmi-clush-tmp/ipmi-clush /usr/local/bin/ipmi-clush && sudo chmod +x /usr/local/bin/ipmi-clush && rm -rf /tmp/ipmi-clush-tmp
```

## Syntax & Usage

`ipmi-clush <clustershell_groups>,<node_ranges> <username> <password> <ipmitool_command>`### Arguments

1. `<node_ranges_or_@groups>` (String): Comma-separated list of nodes, bracketed node matrices (e.g., `cpu0[01-50],gpu0[01-10]`), or native ClusterShell groups (e.g., `@cpu,@vdi`).
1. `<username>` (String): The active BMC user account (e.g., `admin`).
1. `<password>` (String): Password associated with the specified BMC user profile.
1. `<ipmitool_command>` (String): The raw command payload intended for `ipmitool` (do not include authentication parameters or hostname flags).

## Use Cases & Production Examples

### Use Case 1: Cluster-Wide Out-of-Band User Auditing

**Scenario:** Verify user directory privileges and detect inconsistencies across various compute, hardware accelerator, and VDI profiles.

`ipmi-clush cpu0[01-50],gpu0[01-44],vdi0[01-05],fat0[01-08] admin Sup3rP4ssword user list 1`Production Output:



```
---------------
vdi0[01-02] (2)
---------------
ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
1                    true    false      false      NO ACCESS
2   adminU            true    true       true      ADMINISTRATOR
3   tux              true    true       true       ADMINISTRATOR
4   fwupd            true    true       true       ADMINISTRATOR
5   ibmsupport       true    false      false      USER
6                    true    false      false      NO ACCESS
7                    true    false      false      NO ACCESS
8                    true    false      false      NO ACCESS
9                    true    false      false      NO ACCESS
10                   true    false      false      NO ACCESS
11                   true    false      false      NO ACCESS
12                   true    false      false      NO ACCESS
13                   true    false      false      NO ACCESS

---------------
cpu0[01-13,15-26,28-41,43-50],fat0[01-08],gpu0[01-44] (99)
---------------
ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
1                    true    false      false      NO ACCESS
2   adminU           true    true       true       ADMINISTRATOR
3   tux              true    false      false      ADMINISTRATOR
4   dcim             true    false      false      USER
5   ibmsupport       true    false      false      USER
6                    true    false      false      NO ACCESS
7                    true    false      false      NO ACCESS
8                    true    false      false      NO ACCESS
9                    true    false      false      NO ACCESS
10                   true    false      false      NO ACCESS
11                   true    false      false      NO ACCESS
12                   true    false      false      NO ACCESS
13                   true    false      false      NO ACCESS

---------------
cpu0[27,42] (2)
---------------
ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
1                    true    false      false      NO ACCESS
2                    true    false      false      NO ACCESS
3   adminU           true    true       true       ADMINISTRATOR
4                    true    false      false      NO ACCESS
5   ibmsupport       true    false      false      USER
6                    true    false      false      NO ACCESS
7                    true    false      false      NO ACCESS
8                    true    false      false      NO ACCESS
9                    true    false      false      NO ACCESS
10                   true    false      false      NO ACCESS
11                   true    false      false      NO ACCESS
12                   true    false      false      NO ACCESS
13                   true    false      false      NO ACCESS

---------------
cpu0[14] (1)
---------------
ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
1                    true    false      false      NO ACCESS
2   adminU            true    true       true      ADMINISTRATOR
3   tux              true    false      false      ADMINISTRATOR
4   dcim             true    false      false      ADMINISTRATOR
5   ibmsupport       true    false      false      USER
6                    true    false      false      NO ACCESS
7                    true    false      false      NO ACCESS
8                    true    false      false      NO ACCESS
9                    true    false      false      NO ACCESS
10                   true    false      false      NO ACCESS
11                   true    false      false      NO ACCESS
12                   true    false      false      NO ACCESS
13                   true    false      false      NO ACCESS
```

**Critical Diagnostic Value:** Notice how `ipmi_clush` instantly exposes that `demu4xcpu0[27,42]` have an anomalous user map where the `admin` profile resides on **ID 3** instead of the standard **ID 2**. Executing un-aggregated sequential scripting would easily miss this deviation, leading to catastrophic locked-out conditions during cluster-wide password updates.

### Use Case 2: Mass Power Management & Telemetry Verification

**Scenario:** Check chassis power delivery matrices to guarantee nodes are matching orchestration state expectations.

`ipmi-clush cpu0[01-20] admin Sup3rP4ssword power status`Expected Output:



```
---------------
cpu0[01-20] (20)
---------------
Chassis Power is on
```

## Error Handling & Debugging

If an entire range collapses or returns unexpected parameters, check the output key blocks. When a node's BMC layer hangs or suffers from wrong credentials, `ipmi_clush` isolates it safely under a separate summary block header detailing the `ipmitool` timeout or driver exception signature directly:



```
---------------
cpu0[05] (1)
---------------
Error: Unable to establish IPMI v2 / RMCP+ session
```
