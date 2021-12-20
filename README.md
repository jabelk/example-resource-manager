
# Learn by Doing: Dynamic Resource Allocation Using Resource Manager

NSO is often used to provision network services. It is common for these services to require dynamic network-level information, that is not a part of the initial service inputs. Such data can be obtained from, and eventually released back to an external system.

A common example of this are VLAN IDs and IP-addresses used for VLAN interfaces (SVIs). There are software systems to manage these types of dynamically allocated assets. For example, IP Address Management (IPAM) systems are usually used for planning, tracking, and managing an IP address space on a network.

NSO can work nicely with IPAM systems in terms of API integration. However, there are situations where accessing an IPAM is not possible or not needed. In that case, NSO and its Resource Manager package is a useful complementary feature as it can provide basic resource allocation and lifecycle management for the assets required for services managed by NSO. 

In this example, you will learn how to use a Resource Manager and dynamically allocate a VLAN ID in your service.

## Installation

[Reserve the NSO Reservable Sandbox](https://devnetsandbox.cisco.com/RM/Diagram/Index/43964e62-a13c-4929-bde7-a2f68ad6b27c?diagramType=Topology)

If you need to revisit some of the NSO Basics, you can [start here](https://developer.cisco.com/learning/lab/learn-nso-the-easy-way/step/1). 

A demo version of the `resource-manager` package with modules `id_allocator` and `ipaddress_allocator` is already deployed and available on the configured NSO system install server (10.10.20.49):

```
[developer@nso src]$ ncs_cli -C
developer@ncs# show packages package package-version
                      PACKAGE
NAME                  VERSION
-------------------------------
cisco-asa-cli-6.12    6.12.4
cisco-ios-cli-6.67    6.67.9
cisco-iosxr-cli-7.32  7.32.1
cisco-nx-cli-5.20     5.20.6
resource-manager      3.5.3
selftest              1.3
svi_verify_example    1.0
```

## Explanation of Components

The NSO Resource Manager interface, the resource allocator, provides a generic resource allocation mechanism that works well with services. It comes with two ready-to-use allocators, but additional custom packages can be added as well when required.

You can navigate to the `resource-manager` in the `packages` directory of the running NSO instance and examine the contents of the model and its accompanying two modules in the `src/yang` sub directory.

```bash
[developer@nso ~]$ cd /var/opt/ncs/packages/resource-manager/src/yang/
[developer@nso yang]$ ls -l
total 36
-rw-r--r-- 1 developer developer  609 Nov 18  2020 id-allocator-alarms.yang
-rw-r--r-- 1 developer developer 1861 Nov 18  2020 id-allocator-oper.yang
-rw-r--r-- 1 developer developer 4384 Nov 18  2020 id-allocator.yang
-rw-r--r-- 1 developer developer  695 Nov 18  2020 ipaddress-allocator-alarms.yang
-rw-r--r-- 1 developer developer 2260 Nov 18  2020 ipaddress-allocator-oper.yang
-rw-r--r-- 1 developer developer 6396 Nov 18  2020 ipaddress-allocator.yang
-rw-r--r-- 1 developer developer 3062 Nov 18  2020 resource-allocator.yang
```

The YANG model of the resource allocator (`resource-allocator.yang`) can be augmented with different resource pools, as is the case for the two applications `id-allocator` and `ipaddress-allocator`. Each pool contains an `allocation` list where services are supposed to create instances to indicate that they request an allocation. Request parameters are stored in the `request` container and the allocation response is written in the `response` container. The following text will explain how it works.

## NSO ID allocator configuration

The NSO ID Allocator is an extension of the generic resource allocation mechanism named NSO Resource Manager. It can allocate integers which can serve, for instance, as VLAN identifiers.

The ID Allocator can host any number of ID pools. Each pool contains a certain number of IDs that can be allocated. They are specified by a range, and potentially broken into several ranges by a list of excluded ranges.

### Create an ID pool

Configuration for an ID pool requires setting a pool name, the range (start and end numbers), and possible exclusions. 

```
developer@ncs(config)# resource-pools id-pool vlan-id range start 400 end 450
developer@ncs(config-id-pool-vlan-id)# exclude range 410 415
developer@ncs(config-id-pool-vlan-id)# commit dry-run 
cli {
    local-node {
        data  resource-pools {
            +    id-pool vlan-id {
            +        range {
            +            start 400;
            +            end 450;
            +        }
            +        exclude 410 415;
            +    }
              }
    }
}
developer@ncs(config-id-pool-vlan-id)# commit
Commit complete.
```

You can examine the status of the current ID pool with the `show resource-pools id-pool <name>` command. The `show` command shows operational data – the numeric IDs allocated, along with the allocation IDs. Since the allocation has not yet been created, the table will show no allocated IDs.

```
developer@ncs# show resource-pools id-pool vlan-id 
  NAME     ID  ERROR  ID  
  ------------------------
  vlan-id  
!
```

If you want to display the configured resource pools, examine the configuration data with the `show running-config resource-pools` command.

```
developer@ncs# show running-config resource-pools
  resource-pools id-pool vlan-id
  range start 400
  range end 450
  exclude 410 415
!
```

### Create an allocation request

In this section, you will see how you can manually create an allocation request. 

When a pool is created, it is possible to create manual allocation requests on the values handled by the pool. The CLI interaction below shows how to allocate a value in the pool defined above.

```
developer@ncs# config
Entering configuration mode terminal
developer@ncs(config)#
developer@ncs(config)# resource-pools id-pool vlan-id allocation LAB username lab_user
developer@ncs(config-allocation-01)# commit
```

Now that the allocation has been made, you can check the status of the ID pool again.

```
developer@ncs# show resource-pools id-pool vlan-id
NAME     ID   ERROR  ID
--------------------------
vlan-id  LAB  -      400
```

The configuration must be present in the CDB before requesting a new ID from the pool, otherwise the allocation will fail.

## Modifying an existing service for resource manager implementation

In this example, the focus is not about designing or creating the service but to demonstrate how your network services can use external resources. You will create a basic python service with a template so that you get a reference of what needs to be done in your existing service to use the `resource-manager` package for ID allocation inside it. As mentioned, this package simulates an external ID allocation service that is commonly used to allocate parameters such as IP addresses, customer data and dynamic network data.

SSH to the NSO host and go to the packages of the NSO running directory where you will make a python package:

```bash
[developer@nso ~]$ cd /var/opt/ncs/packages
[developer@nso packages]$ ncs-make-package --service-skeleton python-and-template create-vlan
```

### Update the package-meta-data.xml

In this service, you will use the resource-manager package for allocating a VLAN ID number. The first step is to update the package metadata. Update the version of `create-vlan` package (`<package-version>`) and insert `resource-manager` inside the `required-package` tag. Any package listed in this tag will be included in the PYTHONPATH of the started Python VM, so its accompanying Python code will be accessible. It is also used to make sure that nobody can use the service package without also including the `resource-manager` package. The change of the metadata is described below:

```xml
<ncs-package xmlns="http://tail-f.com/ns/ncs-packages">
  <name>create-vlan</name>
  <package-version>1.1</package-version>
  <description>Generated Python package</description>
  <ncs-min-version>5.4</ncs-min-version>
  
  <required-package>
    <name>resource-manager</name>
  </required-package>

  <component>
    <name>main</name>
    <application>
      <python-class-name>alloc_example.main.Main</python-class-name>
    </application>
  </component>
</ncs-package>
```

### Update the service model

Open the `packages/create-vlan/src/yang/create-vlan.yang` file and add the `augment` statement so that your service will be present in the `services` path in the NSO. Make the `device` leaf-list; a list with a leaf `name`. Delete the `dummy` leaf as you will not need it in this case.

```yang
module create-vlan {
  ...
  augment /ncs:services {
    list create-vlan {
      description "This is an RFS skeleton service";

      key name;
      leaf name {
        tailf:info "VLAN name";
        tailf:cli-allow-range;
        type string;
      }

      uses ncs:service-data;
      ncs:servicepoint create-vlan-servicepoint;

      list device {
        tailf:info "L3 switch";
        key "name";

        leaf name {
          tailf:info "Device name";
          type leafref {
            path "/ncs:devices/ncs:device/ncs:name";
          }
        }
      }
    }
  }
}
```

You will need to rebuild the package. In the `packages/create-vlan/src/` execute the `make` command.

```bash
[developer@nso src]$ make
/opt/ncs/current/bin/ncsc  `ls create-vlan-ann.yang  > /dev/null 2>&1 && echo "-a create-vlan-ann.yang"` \
              -c -o ../load-dir/create-vlan.fxs yang/create-vlan.yang
```

Modify the template accordingly.

### Modify the python code

To use the Resource Manager in your own code, load the packages in Cisco NSO and reference the Python code from your packages.

Inside the FASTMAP create callback, you will use this Python module to request a new allocation ID number.

Open the `id_allocator.py` file and study it. The `id_allocator` module implements abstraction provided by the YANG data model and offers two functions that you will use in the service code:

1. `id_request(service, svc_xpath, username, pool_name, allocation_name)` ― Call this method to do custom initialization. The function parameters are as follows:
- `service`: The requesting service node
- `svc_xpath`: The XPath to the requesting service instance
- `username`: A username to use when redeploying the requesting service
- `pool_name`: The name of a pool to request from
- `allocation_name`: A unique allocation name
- `sync`: Sync allocations with this name across pools (optional)
- `requested_id`: A specific ID to be requested (optional)
- `redeploy_type`: A service redeploy action; default, touch, re-deploy, reactive-re-deploy

2. `id_read(username, root, pool_name, allocation_name)`: Returns the allocated ID or None. The function parameters are as follows:
- `username`: The requesting service's transaction's user
- `root`: A maagic root for the current transaction
- `pool_name`: The name of pool to request from
- `allocation_name`: A unique allocation name

The `redeploy_type` argument in the `id_request()` sets the redeploy type used by the Resource Manager to redeploy the service. The allowed values are as follows:
- touch
- re-deploy
- reactive-re-deploy
- default: chooses one of the previous options based on the NSO version.

In this way, you can choose the right redeploy strategy for your scenario. A service redeploy is an important part of how the Resource Manager works, following the so-called Reactive FASTMAP approach. It is required in order to ensure the allocated resource value stays the same across service changes and redeploys.

To see what this means for your `cb_create()` code, consider the following:

- In the first run, the code will request an ID by writing an allocation request to the esource manager tree. It will then check if the response is ready, which it will not be, and return.
- The resource manager subscribes to changes to the resource manager tree, looking for allocation request to be created and deleted. In this case, a new allocation request is created. The resource manager allocates the resources and writes the response in a CDB operational leaf. Finally, the resource manager triggers a service reactive-redeploy action.
- The `cb_create()` is run for a second time. The code will create the allocation request, just as it did the first time, and then it will check if the response is ready. This time it will be ready, and the code can proceed to read the allocated ID and use it in its configuration.

The reactive-re-deploy is used with the subscriber applications. Any service will expose both a re-deploy and a reactive-re-deploy action.

You will see this in action in the following chapter by observing the logs.

In the `packages/python/alloc_example/` open the `main.py` file. The id-allocator Python module is imported into the service Python module with the following import statement.

```python
from resource_manager import id_allocator
```

Modify the python mapping code to use an `id_allocator`. Inside `cb_create()` make a request for the ID with the help of a `id_allocator.id_request()` function. When the ID is available, read it by using the `id_allocator.id_read()` function inside the service configuration.

```python
class ServiceCallbacks(Service):

    # The create() callback is invoked inside NCS FASTMAP and
    # must always exist.
    @Service.create
    def cb_create(self, tctx, root, service, proplist):
        self.log.info(f'Service create(service={service._path}')

        service_path = f'/services/create-vlan[name={service.name}]'
        id_allocator.id_request(service, service_path, tctx.username, 'vlan-id', service.name, False)
        vlan_id = id_allocator.id_read(tctx.username, root, 'vlan-id', service.name)
        self.log.info(f'VLAN ID: {vlan_id}')

        if vlan_id is None:
            self.log.info(f'Allocation is not ready yet. Redeploying ....')
            return proplist                

        for device in service.device:
            self.log.info(f'SVI device name = {device.name}')

            self.log.info(f'VLAN-ID = {vlan_id}')
            self.log.info(f'VLAN name = {service.name}')

            vars = ncs.template.Variables()
            vars.add('DEVICE', device.name)
            vars.add('VLAN-ID', vlan_id)
            vars.add('VLAN-NAME', service.name)
            template = ncs.template.Template(service)
            template.apply('create-vlan-template', vars)
```

Save the file and reload the packages.

### Service deployment

Before deploying the service, verify what you have in the allocation pool:

```
developer@ncs# show resource-pools id-pool vlan-id
NAME     ID   ERROR  ID
--------------------------
vlan-id  LAB  -      400
```

To check the logs, you may want to open another terminal window and log in to the NSO. To catch the real-time logs, enter the `tail -f` command along with the log file name, in this case `ncs-python-vm-create-vlan.log`.

```bash
[developer@nso ~]$  tail -f /var/log/ncs/ncs-python-vm-create-vlan.log
```

Configure the service and commit:

```
developer@ncs# config
Entering configuration mode terminal
developer@ncs(config)# services create-vlan LAB1 device dist-sw01
developer@ncs(config-device-dist-sw01)# commit
Commit complete.
developer@ncs(config-device-dist-sw01)# exit
developer@ncs(config-create-vlan-LAB1)# exit
developer@ncs(config)# services create-vlan LAB2 device dist-sw01
developer@ncs(config-device-dist-sw01)# commit
Commit complete.
```

### Verification

Check the allocation:

```
developer@ncs# show resource-pools id-pool vlan-id
NAME     ID    ERROR  ID
---------------------------
vlan-id  LAB   -      400
         LAB1  -      401
         LAB2  -      402

developer@ncs#
```

Check the terminal window with the log file. You should see some output there.

```bash
<INFO> 07-Dec-2021::04:27:30.975 create-vlan ncs-dp-5898-create-vlan:main-1-th-1493: - Service create(service=/ncs:services/create-vlan:create-vlan{LAB1}
<INFO> 07-Dec-2021::04:27:30.985 create-vlan ncs-dp-5898-create-vlan:main-1-th-1493: - VLAN ID: None
<INFO> 07-Dec-2021::04:27:30.985 create-vlan ncs-dp-5898-create-vlan:main-1-th-1493: - Allocation is not ready yet. Redeploying ....
<INFO> 07-Dec-2021::04:27:31.69 create-vlan ncs-dp-5898-create-vlan:main-1-th-1507: - Service create(service=/ncs:services/create-vlan:create-vlan{LAB1}
<INFO> 07-Dec-2021::04:27:31.78 create-vlan ncs-dp-5898-create-vlan:main-1-th-1507: - VLAN ID: 401
<INFO> 07-Dec-2021::04:27:31.79 create-vlan ncs-dp-5898-create-vlan:main-1-th-1507: - SVI device name = dist-sw01
<INFO> 07-Dec-2021::04:27:31.79 create-vlan ncs-dp-5898-create-vlan:main-1-th-1507: - VLAN-ID = 401
<INFO> 07-Dec-2021::04:27:31.79 create-vlan ncs-dp-5898-create-vlan:main-1-th-1507: - VLAN name = LAB1
<INFO> 07-Dec-2021::04:42:47.924 create-vlan ncs-dp-5898-create-vlan:main-2-th-1532: - Service create(service=/ncs:services/create-vlan:create-vlan{LAB2}
<INFO> 07-Dec-2021::04:42:47.933 create-vlan ncs-dp-5898-create-vlan:main-2-th-1532: - VLAN ID: None
<INFO> 07-Dec-2021::04:42:47.933 create-vlan ncs-dp-5898-create-vlan:main-2-th-1532: - Allocation is not ready yet. Redeploying ....
<INFO> 07-Dec-2021::04:42:48.14 create-vlan ncs-dp-5898-create-vlan:main-2-th-1546: - Service create(service=/ncs:services/create-vlan:create-vlan{LAB2}
<INFO> 07-Dec-2021::04:42:48.24 create-vlan ncs-dp-5898-create-vlan:main-2-th-1546: - VLAN ID: 402
<INFO> 07-Dec-2021::04:42:48.24 create-vlan ncs-dp-5898-create-vlan:main-2-th-1546: - SVI device name = dist-sw01
<INFO> 07-Dec-2021::04:42:48.25 create-vlan ncs-dp-5898-create-vlan:main-2-th-1546: - VLAN-ID = 402
<INFO> 07-Dec-2021::04:42:48.25 create-vlan ncs-dp-5898-create-vlan:main-2-th-1546: - VLAN name = LAB2
```

As explained earlier, the allocation did not take place at the first run of `cb_create()` therefore the VLAN ID has not yet been allocated. After the second take, you can see that the `id_read()` method is able to get the allocated VLAN ID.


You can also check the `dist-sw01` device configuration and ensure that the VLAN IDs and names have been allocated. Note that some of the VLANs were already created:

```
developer@ncs# show running-config devices device dist-sw01 config vlan
devices device dist-sw01
config
  vlan 1
  !
  vlan 101
  name prod
  !
  vlan 102
  name dev
  !
  vlan 103
  name test
  !
  vlan 104
  name security
  !
  vlan 105
  name iot
  !
  vlan 401
  name LAB1
  !
  vlan 402
  name LAB2
  !
!
```

### Credit

Special Thanks to [Robert Silar](https://www.linkedin.com/in/robert-%C5%A1ilar-a5544487/) for helping write this post! 