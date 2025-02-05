# pwr module

This is a python module which allows an application to modify power attributes of a CPU. Manipulation can be done to the frequency of cores,
frequency of the uncore, frequency profiles can be set to achieve desired performance, as well as that capabilities of a CPU can be obtained and its frequency and power consumption stats monitored.
The library will provide a list of core, CPU objects, and a system object, some of whose attributes can be modified and committed to make changes on the CPU.

## Prerequisites

The module requires that hardware P-States are enabled in the BIOS and the intel_pstate performance scaling driver is used in the kernel.
Specific features of the module can only be utilized as long as the underlying hardware/software has that capability. Minimum requirements for these features are as follows:

* Intel(R) Speed Select Technology - Base Frequency (SST-BF): 
    * Requires Linux kernel v5.1 or later.
    * SST-BF compatible BIOS (For Intel(R) boards, please check their respective BIOS downloads, for example [the S2600WF BIOS with SST-BF support](https://downloadcenter.intel.com/download/29105/Intel-Server-Board-S2600WF-Family-BIOS-and-Firmware-Update-Package-for-UEFI?v=t)

    * A supported CPU (such as Intel(R) Xeon 6230N)

## Installation

The module can be installed with `pip` using the following command:

```sh
pip install "git+https://github.com/intel/CommsPowerManagement.git#egg=pwr&subdirectory=pwr"
```

If not running as a privileged user, the above should be prefixed with
`sudo -H`. If the `pwr` module is to be used in Python3 environment, then the
`pip` command should be replaced with `pip3`.

## Initialization

Creation of the system, cpu and core objects is done using the `get_system()/get_cpus()/get_cores()` functions, which will return a list of the respective objects in the case of cores or CPUS and will return a single system object when requesting system object.

```python
import pwr  # Import the module

cores = pwr.get_cores()  # Return core object list
cpus = pwr.get_cpus()  # Return CPU object list
system = pwr.get_system()  # Return system object
```
There is also a `get_objects()` API which will return all three class objects.
```python
import pwr

system, cpus, cores = get_objects()  # Return all objects
```

## Adjusting Power Configuration

### Modifying

Each system, cpu and core object have attributes which replicate the capabilities and stats of the physical cores and cpus.

### SYSTEM

* `cpu_list`                # list of CPU objects on the system
* `sst_bf_enabled`          # base frequency enabled in BIOS
* `sst_bf_configured`       # all system cores min&max set to sst_bf base frequency
* `epp_enabled`             # epp enabled flag

### CPU

* `cpu_id`                  # CPU id number
* `physical_id`             # physical CPU number
* `core_list`               # list of core objects on this CPU
* `sys`                     # system object
* `sst_bf_configured`       # core configured to use base frequencies
* `turbo_enabled`           # turbo enabled flag
* `hwp_enabled`             # HWP enabled flag
* `base_freq`               # base frequency
* `all_core_turbo_freq`     # all core turbo frequency
* `highest_freq`            # highest available frequency
* `lowest_freq`             # lowest available frequency
* `uncore_hw_max`           # max available uncore frequency
* `uncore_hw_min`           # min available uncore frequency
* `power_consumption`       # power consumption since last update
* `tdp`                     # max possible power consumption
* `uncore_freq`             # current uncore frequency
* `uncore_max_freq`         # max desired uncore frequency
* `uncore_min_freq`         # min desired uncore frequency

### Core

* `core_id`                 # logical core id number
* `online`                  # core availability flag 
* `cpu`                     # this cores cpu object
* `thread_siblings`         # list of other logical cores residing on same physical core
* `high_priority`           # boolean value indicating whether the core will be set up to be a high priority core when SST-BF is configured.
* `base_freq`               # base frequency [2300Mhz]
* `sst_bf_base_freq`        # priority based frequency [2100Mhz/2700Mhz]
* `all_core_turbo_freq`     # all core turbo frequency [2800Mhz]
* `highest_freq`            # highest frequency available [3900Mhz]
* `lowest_freq`             # lowest active frequency [800Mhz]
* `curr_freq`               # current core frequency
* `min_freq`                # desired low frequency
* `max_freq`                # desired high frequency
* `epp`                     # energy performance preference
* `cstates`                 # dict of C-states

> NOTE: Specific frequencies will depend on system and configuration.

Most of the object attributes are constant and cannot be changed. The only **Core** attributes that can be written to by the user, are `min_freq`, `max_freq` and `epp`.
The only **CPU** object attributes which can be written to by the user, are `uncore_max_freq` and `uncore_min_freq`. All **System** attributes are read-only.

```python
core.min_freq = core.lowest_freq  # Set the desired minimum frequency to be lowest available
core.max_freq = core.highest_freq  # Set the desired maximum frequency to the highest available

cpu.uncore_max_freq = cpu.uncore_hw_max  # Set the desired maximum uncore frequency to the highest available
cpu.uncore_min_freq = cpu.uncore_hw_min  # Set the desired minimum uncore frequency to the lowest available
```

### Committing

Modification of the power settings of a system is done by altering the core or CPU characteristics, as shown above,
then finalizing with the `commit()` function call. Commits can be done at a core, cpu or system level.

```python
cores = pwr.get_cores()  

for core in cores:  # Loop through core objects in list
    core.min_freq = core.base_freq  # Set desired minimum frequency to be the base frequency
    core.max_freq = core.highest_freq  # Set the desired maximum frequency to the highest available
    core.commit()  # Commits this cores changes to be made on system
```
or 
```python
system, cpus, cores = pwr.get_objects()

for core in cores:
    core.min_freq = core.base_freq  # Set the desired minimum frequency to be the base frequency
    core.max_freq = core.highest_freq  # Set the desired maximum frequency to the highest available
system.commit()
```

### Pre-set Profiles

When an application is modifying the desired min and max core frequencies, pre-set configurations can also be applied, these will overwrite current configurations and commit the pre-sets.

* `minimum`:            Set all cores minimum to 800Mhz and maximum to 800Mhz.
* `maximum`:            Set all cores minimum to 3900Mhz and maximum to 3900Mhz.
* `base`:               Set all cores minimum to 2300Mhz and maximum to 2300Mhz.
* `default`:            Set all cores minimum to 800Mhz and maximum to 3900Mhz.
* `no_turbo`:           Set all cores minimum to 800Mhz and maximum to 2300Mhz.
* `sst_bf`:             Set high priority cores minimum and maximum to 2700Mhz, normal priority cores minimum and maximum to 2100Mhz (Only available with SST-BF).

> NOTE: Specific frequencies will depend on system and configuration.

```python
For c in cores:
    c.commit("sst_bf")  # Configure cores with the SST-BF configuration
```
or 
```python
system.commit("sst_bf")  # Configure system with the SST-BF configuration
```

## Concept Overview

### CPU Objects

A CPU may have multiple physical cores, which are represented by the core objects. These cores can optionally have multiple threads, which means there would be multiple logical cores on a single physical core, these are called thread siblings. Each logical core will also have its own core object. Uncore frequency is the frequency at which everything on the CPU, except the cores, runs at. This can be scaled within the limits of min and max.

### Core Objects

Each core's frequency can be scaled up or down to particular frequencies (P-states). P-states may include P0 (Turbo Frequency), P1 (Base Freqeuncy) with as many frequency steps as the CPU provides, with a larger the P-state number indicating lower core operating frequency. A core can also be put into idle/sleep states, C-states, by the OS when it is not busy. C-states follow the same pattern as P-states where larger numbered C-states consume less power. Available P-states and C-states will depend on system configuration and CPU model. In the case of multiple logical cores on a physical core, the P-states and C-states are shared and must be configured the same for desired changes to take effect. Core frequencies can be set up to utilize the SST-BF configuration, if available. Energy performance profiles can be set up on a per core basis using a specific EPP policy, if the system configuration allows.

## Refreshing CPU stats

CPU stats can become out of date, such as `curr_freq` or `sst_bf_configured`. These can be refreshed in with `refresh_stats()`.
It is *advised* that CPU and core objects be updated with `refresh_stats()` before requesting object data.

core.refresh_stats() will update:

* `curr_freq`
* `min_freq`
* `max_freq`
* `epp`
* `online`
* `cstates`

cpu.refresh_stats() will update:

* `uncore_freq`
* `uncore_max_freq`
* `uncore_min_freq`
* `sst_bf_configured`
* `power_consumption`

system.refresh_stats() will update:
* `sst_bf_configured`

system.refresh_all() will update all of the above object attributes:
* `curr_freq`
* `min_freq`
* `max_freq`
* `epp`
* `online`
* `cstates`
* `uncore_freq`
* `uncore_max_freq`
* `uncore_min_freq`
* `sst_bf_configured`
* `power_consumption`

```python
for c in cores:
    c.refresh_stats()  # Refresh core stats, which refreshes the sst_bf_configured value
    if not c.sst_bf_configured:  # Check are cores configured for SST-BF
        c.commit("sst_bf_base")  # Set cores to SST-BF configuration
```

## Object Referencing

Once you have any one of the three library objects you can access the other two.

```python
system.cpu_list[0].core_list[0].core_id  # Accessing core attributes through system object
cores[0].cpu.sys.epp_enabled  # Accessing system attributes through core object
```

## EPP

The EPP value (`epp` attribute in the `Core` object) uses SST-CP technology to prioritize core power consumption. Available EPP values are:

* "performance" (maximum power consumption priority)
* "balance_performance" (default power consumption priority)
* "balance_power" (lower power consumption priority)
* "power" (lowest power consumption priority)

For more information about EPP, see relevant product manuals' section describing the SST-CP technology.

## Power Consumption

The power consumption of the CPU can be read from the `power_consumption` attribute, this value can be compared against the `tdp` value to check is the current power draw close to the limit, indicated by the tdp value. The power consumption is reported as average since last time it was calculated. The first time this value is read, it can be 0. The next time it is read, it will show average power consumption (in Watts) since last read. Time period between reads must not exceed 60 seconds, otherwise the value will be reset to 0.

## C-States Configuration

Any C-states permitted by the BIOS can be enabled/disabled by the pwr library. A cores' current c-state configuration can be checked and is stored in `core.cstates` dict.
```python
cstates = { 'C6': True,
            'C1': True,
            'C1E': True,
            'POLL': True}
```
 C-State configuration can be modified by writing True/False to corresponding dict value, to enable/disable respectively, then committing. 
```python
for core in cores:
    core.cstates["C1"] = True  # Enabling C1
    core.cstates["C6"] = False  # Disabling C6
    core.commit()
```

## Request Configuration

An application can request to check if a certain core frequency configuration is stable. This should be used as an indicator as to whether the configuration is within the TDP threshold. A positive return from the API is not a guarantee on the configuration, but a gauge on whether the setup is likely to stay within the available power budget. Once the request is validated the application can then proceed to commit it to the system.
The request is checked through `system.request_configuration()`. If the request is called with no arguments the check will be done on the current configuration of all core objects across the system. If the request is called with a CPU object or a list of CPU objects, a check will be done on just the given CPU(s).  This feature can be utilized to allow the user to to frame core frequencies within stable boundaries while also getting desired results. 


```python
# Full system configuration test
for core in cores:
    core.min_freq = core.lowest_freq
    core.max_freq = core.all_core_turbo_freq

if system.request_config():
    system.commit()
```
```python
# Single CPU configuration test
cpu = system.cpu_list[0]

for core in cpu.core_list:
    core.min_freq = core.lowest_freq
    core.max_freq = core.all_core_turbo_freq
if request_config(cpus=cpu):
    for core in cpu.core_list:
        core.commit()
```
