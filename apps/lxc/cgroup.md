# Control Groups
## Problem Statement
To develop a mechanism to provide find-grained resource control on a process/set of processes running on a machine

## Solution

- A Cgroup refers to a resource controller for a certain type of CPU resource. e.g., Memory Cgroup, Network Cgroup etc.
- A process group attaches itself to a single node present on each the hierarchies mounted on a system.

![Cgroups Implementation, Hierarchies and Kernel Data Structures](images/cgroup_illustration)

### Implementation
There are data structures introduced to incorporate Cgroups.

1. Container_subsys_state
  
  1.1 Represents the base type from which subsystem state objects are derived

2. Css_group
  
  2.1 It holds one containter_subsys_state pointer for each registered subsystem