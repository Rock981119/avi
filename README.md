# NSX ALB自动化后端池(Server Pool)水平扩缩
###### tags: `Avi`
![](https://i.imgur.com/wIVe9E3.png)

## 前言
推广VMware NSX ALB(原Avi)已经有段时间了. 经常在与客户的交流中提及ALB可以借助自身数控分离的结构, 虚拟化基建, 水平扩缩ALB实例, 实现性能的弹性. 但其实这是有几个前提的, 一是后端池自身性能足够, 二是有足够的License允许ALB水平扩展. 抛开这两个前提只说ALB自己能水平扩缩没有业务价值.
在公有云的使用场景中, 用户本身不需要关心云厂的LB如何实现性能弹性. 只需要根据所能获取的度量值对后端池做扩缩即可. 度量值也得来自云厂监控运维系统. 似乎这种模型客户普遍认同.
虽然这种模型得到普遍认同, 可是在私有云实践中往往难以实现自动化, 大多是LB和后端池都是以预估峰值构建的. 
在V言V, 本文讲讲如何使用NSX ALB的自身度量值实现ALB与后端池(vSphere)的自动化扩缩容. 无需借助额外的运维监控参数.


## 准备工作
* ALB与vSphere集成. ALB版本 > 18.1.5, vSphere版本 > 6.5
* 网络可使用vDS, NSX可选
* 将要发布的业务做成虚拟机模版
* DHCP地址分配
* 亿点点Python基础

## 演示拓扑
![](https://i.imgur.com/O9viDKq.png)
拓扑中采用了NSX-T为虚拟机构建网络, 非必选项, 使用vDS的Port Group也是可以的. 客户端使用Python Locust程序做Http压测. 预期效果: 当后端池平均连接数高于自定义值时, 后端池成员水平扩展. 当ALB实例到达自动平衡阀值时, 水平扩展, ALB池内有Buffer LB实例时直接使用.

## 大致步骤

1. 定义扩容 / 缩容脚本(Python)
2. 定义自动扩缩alerts
3. 后端池通过自动扩缩策略调用自定义扩缩alerts
4. 自定义扩缩alerts产生扩缩事件, 触发扩缩脚本

## 趟坑手册
本来ALB的文档质量相当高的, 有可能是收购后最近发版太快, 或者是这个特性实际用户用得少? 官方文档出现了一些错误, 已经提交后台处理了. 本文使用的配置为个人趟坑后的总结.

### 步骤1 定义扩容 / 缩容脚本

进入ALB界面, Templates--Scripts--ControlScripts, 新建Scaleout-ControlScripts, 输入以下内容:
```
#!/usr/bin/python3
import sys
from avi.autoscaling.vmware_example.vmware_scale import scale_out

vmware_settings = {
    'vcenter': '192.168.31.251',
    'user': 'administrator@rockhomelab.org',
    'password': '填入你的VC密码',
    'cluster_name': 'C2-WCP',
    'template_name': 'Photon4Web',
    'template_folder_name': 'DC-Home/VMware-Photon',
    'customization_spec_name': 'Tenant1-Web',
    'vm_folder_name': 'DC-Home',
    'resource_pool_name': None,
    'port_group': 'Tenant1-Web'}

scale_out(vmware_settings, *sys.argv)
```

再新建一个缩容的脚本, Scalein-ControlScripts, 输入以下内容:
```
#!/usr/bin/python3
import sys
from avi.autoscaling.vmware_example.vmware_scale import scale_in

vmware_settings = {
    'vcenter': '192.168.31.251',
    'user': 'administrator@rockhomelab.org',
    'password': '填入你的VC密码',
    'vm_folder_name': 'DC-Home'}

scale_in(vmware_settings, *sys.argv)
```
填入参数的含义官方文档有说明:
```
The vmware_settings parameters are as follows:

    vcenter – IP address or hostname of vCenter.
    user – Username for vCenter access.
    password – Password for vCenter access.
    cluster_name – vCenter cluster name.
    template_folder_name – Folder containing template VM, for instance, Datacenter1/Folder1/Subfolder1 or None to search all (likely more efficient to specify a folder if there are a large number of VMs).
    template_name – Name of template VM.
    vm_folder_name – Folder name to place new VM (or None for same as template).
    customization_spec_name – Name of a customization spec to use - DHCP should be used to allocate IP addresses to the newly-created VMs.
    resource_pool_name – Name of VMware Resource Pool or None for no specific resource pool (VM will be assigned to the default “hidden” resource pool).
    port_group – Name of port group containing pool member IP (useful if the VM has multiple vNICs) or None to use the IP from the first vNIC.

```
其中customization_spec要在vCenter中创建好, 不能为None. port_group虽然可以填None(即使用VM模版中的Port Group), 但经过测试port_group=None & customization_spec使用时, 会导致自动部署出来的虚拟机网卡无法自动连接. 所以我在后续的脚本中取消了执行时调用customization_spec. 建议按照我上述的字段创建即可.

### 步骤2 定义告警行为
进入ALB界面, Operations--Alerts--Alert Actions, 新建VM-Server-Scale-Event
![](https://i.imgur.com/isb3dqX.png)
除了名字其他的什么都不用填.

Operations--Alerts--Alert Config, 创建告警条件和触发器, 调用刚才创建的VM-Server-Scale-Event(Alert Actions)
创建Alert Config: VM-ScaleIn
![](https://i.imgur.com/8idipOd.png)
创建Alert Config: VM-ScaleOut
![](https://i.imgur.com/wAKAyGW.png)
我这里设置的条件是当后端池L7 RPS在5秒里持续大于100每秒产生ScaleOut事件, 反之产生ScaleIn事件, 抑制时间120s. AutoScale Alert必须勾选.

### 步骤3 定义Autoscaling Policy, 在后端池调用该策略
进入ALB界面, Templates--AutoScale--Server AutoScale, 创建VM-Server-AutoSacle-Policy
![](https://i.imgur.com/hMNnLES.png)
AutoSacle Policy中调用刚才创建的VM-ScaleIn / VM-ScaleOut(Alert Actions). Cooldown Period要大于Alert Actions中的抑制时间.

然后回到Application--Pool--<目标池>中调用VM-Server-AutoSacle-Policy:
![](https://i.imgur.com/2pQL1N1.png)

### 步骤4 创建事件触发, 执行扩缩脚本
回到Operations--Alerts--Alert Action, 创建VM-Server-AutoscaleIn, 调用缩容脚本:
![](https://i.imgur.com/4VFPbZm.png)
创建VM-Server-AutoscaleOut, 调用扩容脚本:
![](https://i.imgur.com/i5q5AK0.png)
回到Operations--Alerts--Alert Config, 创建VM-Server-ScaleIn-Action:
![](https://i.imgur.com/G7IJKdh.png)
创建VM-Server-ScaleOut-Action:
![](https://i.imgur.com/AAFTLae.png)
注意: 触发条件为Event, AutoScale Alert不需要勾选. 这里逻辑似乎不是很直接, 我自制一张示意图可供参考:
![](https://i.imgur.com/sGHzVQx.png)

### 步骤5, 调整ALB控制器Python Library
本来这步可以不做的, 正如我之前说的, 文档和该功能的脚本出了点小问题. 我在这里直接提供我执行成功的library脚本. 
SSH登录到ALB控制器上,在/opt/avi/python/lib/avi/autoscaling/vmware_example路径下可以看到vmutils.py和vmware_scale.py, 这两个脚本就是作为library被我们之前做的ControlScript调用的. 脚本里的函数出了点小问题, 我们先把这两个脚本备份一下. 然后把我修正的脚本替换进去. 
vmutils.py
```
from pyVmomi import vim     #pylint: disable=no-name-in-module
from pyVim.connect import SmartConnectNoSSL, Disconnect
import time


def _get_obj(content, vimtype, name, folder=None):
    """
    Get the vsphere object associated with a given text name
    """
    obj = None
    container = content.viewManager.CreateContainerView(
        folder or content.rootFolder, vimtype, True)
    for c in container.view:
        if c.name == name:
            obj = c
            break
    return obj


def _get_child_folder(parent_folder, folder_name):
    obj = None
    for folder in parent_folder.childEntity:
        if folder.name == folder_name:
            if isinstance(folder, vim.Datacenter):
                obj = folder.vmFolder
            elif isinstance(folder, vim.Folder):
                obj = folder
            else:
                obj = None
            break
    return obj


def get_folder(si, name):
    subfolders = name.split('/')
    parent_folder = si.RetrieveContent().rootFolder
    for subfolder in subfolders:
        parent_folder = _get_child_folder(parent_folder, subfolder)
        if not parent_folder:
            break
    return parent_folder


def get_vm_by_name(si, name, folder=None):
    """
    Find a virtual machine by its name and return it
    """
    return _get_obj(si.RetrieveContent(), [vim.VirtualMachine], name,
                    folder)


def get_resource_pool(si, name, folder=None):
    """
    Find a resource pool by its name and return it
    """
    return _get_obj(si.RetrieveContent(), [vim.ResourcePool], name,
                    folder)


def get_cluster(si, name, folder=None):
    """
    Find a cluster by it's name and return it
    """
    return _get_obj(si.RetrieveContent(), [vim.ComputeResource], name,
                    folder)


def wait_for_task(task, timeout=600):
    """
    Wait for a task to complete
    """
    timeout_time = time.time() + timeout
    timedout = True
    while time.time() < timeout_time:
        if task.info.state == 'success':
            return (True, task.info.result)
        if task.info.state == 'error':
            return (False, None)
        time.sleep(1)
    return (None, None)


def wait_for_vm_status(vm, condition, timeout=600):
    timeout_time = time.time() + timeout
    timedout = True
    while timedout and time.time() < timeout_time:
        if (condition(vm)):
            timedout = False
        else:
            time.sleep(60)

    return not timedout


def net_info_available(vm):
    return (vm.runtime.powerState == vim.VirtualMachinePowerState.poweredOn
            and
            vm.guest.toolsStatus == vim.vm.GuestInfo.ToolsStatus.toolsOk
            and
            vm.guest.net)


class vcenter_session:
    def __enter__(self):
        return self.session

    def __init__(self, host, user, pwd):
        session = SmartConnectNoSSL(host=host, user=user, pwd=pwd)
        self.session = session

    def __exit__(self, type, value, traceback):
        if self.session:
            Disconnect(self.session)
```

vmware_scale.py
```
from pyVmomi import vim     #pylint: disable=no-name-in-module
#This is no longer available as a part of SDK, needs to be fixed
# pylint: disable=no-name-in-module
from avi.autoscaling.vmware_example import vmutils
from avi.sdk.samples.autoscale.samplescaleout import scaleout_params
import uuid
from avi.sdk.avi_api import ApiSession
import json
import os


def getAviApiSession(tenant='admin', api_version=None):
    """
    Create local session to avi controller
    """
    token = os.environ.get('API_TOKEN')
    user = os.environ.get('USER')
    tenant = os.environ.get('TENANT')
    api = ApiSession.get_session("localhost", user, token=token,
                                 tenant=tenant, api_version=api_version)
    return api, tenant


def do_scale_in(vmware_settings, instance_names):
    """
    Perform scale in of pool.
    vmware_settings:
        vcenter: IP address or hostname of vCenter
        user: Username for vCenter access
        password: Password for vCenter access
        vm_folder_name: Folder containing VMs (or None for same as
            template)
    instance_names: Names of VMs to destroy
    """

    with vmutils.vcenter_session(host=vmware_settings['vcenter'],
                                 user=vmware_settings['user'],
                                 pwd=vmware_settings['password']) as session:

        vm_folder_name = vmware_settings.get('vm_folder_name', None)

        if vm_folder_name:
            vm_folder = vmutils.get_folder(session, vm_folder_name)
        else:
            vm_folder = None

        for instance_name in instance_names:
            vm = vmutils.get_vm_by_name(session, instance_name, vm_folder)
            if vm:
                print('Powering off VM %s...' % instance_name)
                power_off_task = vm.PowerOffVM_Task()
                (power_off_task_status,
                 power_off_task_result) = vmutils.wait_for_task(
                    power_off_task)
                if power_off_task_status:
                    print('Deleting VM %s...' % instance_name)
                    destroy_task = vm.Destroy_Task()
                    (destroy_task_status,
                     destroy_task_result) = vmutils.wait_for_task(
                        destroy_task)
                    if destroy_task_status:
                        print('VM %s deleted!' % instance_name)
                    else:
                        print('VM %s deletion failed!' % instance_name)
                else:
                    print('Unable to power off VM %s!' % instance_name)
            else:
                print('Unable to find VM %s!' % instance_name)


def do_scale_out(vmware_settings, pool_name, num_scaleout):
    """
    Perform scale out of pool.
    vmware_settings:
        vcenter: IP address or hostname of vCenter
        user: Username for vCenter access
        password: Password for vCenter access
        cluster_name: vCenter cluster name
        template_folder_name: Folder containing template VM, e.g.
            'Datacenter1/Folder1/Subfolder1' or None to search all
        template_name: Name of template VM
        vm_folder_name: Folder to place new VM (or None for same as
            template)
        customization_spec_name: Name of a customization spec to use
        resource_pool_name: Name of VMWare Resource Pool or None for default
        port_group: Name of port group containing pool member IP
    pool_name: Name of the pool
    num_scaleout: Number of new instances
    """

    new_instances = []

    with vmutils.vcenter_session(host=vmware_settings['vcenter'],
                                 user=vmware_settings['user'],
                                 pwd=vmware_settings['password']) as session:

        template_folder_name = vmware_settings.get('template_folder_name',
                                                   None)
        template_name = vmware_settings['template_name']
        if template_folder_name:
            template_folder = vmutils.get_folder(session,
                                                 template_folder_name)
            template_vm = vmutils.get_vm_by_name(
                session, template_name,
                template_folder)
        else:
            template_vm = vmutils.get_vm_by_name(
                session, template_name)

        vm_folder_name = vmware_settings.get('vm_folder_name', None)
        if vm_folder_name:
            vm_folder = vmutils.get_folder(session, vm_folder_name)
        else:
            vm_folder = template_vm.parent

        csm = session.RetrieveContent().customizationSpecManager
        customization_spec = csm.GetCustomizationSpec(
            name=vmware_settings['customization_spec_name'])

        cluster = vmutils.get_cluster(session,
                                      vmware_settings['cluster_name'])

        resource_pool_name = vmware_settings.get('resource_pool_name', None)

        if resource_pool_name:
            resource_pool = vmutils.get_resource_pool(session,
                                                      resource_pool_name)
        else:
            resource_pool = cluster.resourcePool
        relocate_spec = vim.vm.RelocateSpec(pool=resource_pool)

        clone_spec = vim.vm.CloneSpec(powerOn=True, template=False,
                                      location=relocate_spec)

        port_group = vmware_settings.get('port_group', None)

        clone_tasks = []

        for instance in range(num_scaleout):
            new_vm_name = '%s-%s' % (pool_name, str(uuid.uuid4()))

            print('Initiating clone of %s to %s' % (template_name,
                                                    new_vm_name))

            clone_task = template_vm.Clone(name=new_vm_name,
                                           folder=vm_folder,
                                           spec=clone_spec)
            print('Task %s created.' % clone_task.info.key)
            clone_tasks.append(clone_task)

        for clone_task in clone_tasks:
            print('Waiting for %s...' % clone_task.info.key)

            clone_task_status, clone_vm = vmutils.wait_for_task(clone_task,
                                                                timeout=600)

            ip_address = None

            if clone_vm:
                print('Waiting for VM %s to be ready...' % clone_vm.name)
                if vmutils.wait_for_vm_status(clone_vm,
                                              condition=vmutils.net_info_available,
                                              timeout=600):
                    print('Getting IP address from VM %s' % clone_vm.name)
                    for nic in clone_vm.guest.net:
                        if port_group is None or nic.network == port_group:
                            for ip in nic.ipAddress:
                                if '.' in ip:
                                    ip_address = ip
                                    break
                        if ip_address:
                            break
                else:
                    print('Timed out waiting for VM %s!' % clone_vm.name)

                if not ip_address:
                    print('Could not get IP for VM %s!' % clone_vm.name)
                    power_off_task = clone_vm.PowerOffVM_Task()
                    (power_off_task_status,
                     power_off_task_result) = vmutils.wait_for_task(
                        power_off_task)
                    if power_off_task_status:
                        destroy_task = clone_vm.Destroy_Task()
                else:
                    print('New VM %s with IP %s' % (clone_vm.name,
                                                    ip_address))
                    new_instances.append((clone_vm.name, ip_address))
            elif clone_task_status is None:
                print('Clone task %s timed out!' % clone_task.info.key)

    return new_instances


def scale_out(vmware_settings, *args):
    alert_info = json.loads(args[1])
    api, tenant = getAviApiSession()
    (pool_name, pool_uuid,
     pool_obj, num_scaleout) = scaleout_params('scaleout',
                                               alert_info,
                                               api=api,
                                               tenant=tenant)
    print('Scaling out pool %s by %d...' % (pool_name, num_scaleout))

    new_instances = do_scale_out(vmware_settings,
                                 pool_name, num_scaleout)

    # Get pool object again in case it has been modified
    pool_obj = api.get('pool/%s' % pool_uuid, tenant=tenant).json()

    new_servers = pool_obj.get('servers', [])

    for new_instance in new_instances:
        new_server = {
            'ip': {'addr': new_instance[1], 'type': 'V4'},
            'hostname': new_instance[0]}
        new_servers.append(new_server)
    pool_obj['servers'] = new_servers
    print('Updating pool with %s' % new_servers)
    resp = api.put('pool/%s' % pool_uuid, data=json.dumps(pool_obj))
    print('API status: %d' % resp.status_code)


def scale_in(vmware_settings, *args):
    alert_info = json.loads(args[1])
    api, tenant = getAviApiSession()
    (pool_name, pool_uuid,
     pool_obj, num_scaleout) = scaleout_params('scalein',
                                               alert_info,
                                               api=api,
                                               tenant=tenant)

    remove_instances = [instance['hostname'] for instance in
                        pool_obj['servers'][-num_scaleout:]]
    pool_obj['servers'] = pool_obj['servers'][:-num_scaleout]
    print('Scaling in pool %s by %d...' % (pool_name, num_scaleout))
    resp = api.put('pool/%s' % pool_uuid, data=json.dumps(pool_obj))
    print('API status: %d' % resp.status_code)

    do_scale_in(vmware_settings, remove_instances)
```
可以看到ALB控制也是使用的vSphere Python SDK来执行这个自动化需求的, 也就是说我们其实可以借用这个思路DIY成更贴近业务场景的脚本.

### 验证
在我个人电脑上使用Python程序Locust来产生L7流量, 软件本身是开源的, 支持本地多进程造流, 也支持Master / Worker模式分布式造流. HTTP(s)压测利器, 推荐一波.
![](https://i.imgur.com/UAs0OYr.png)
模拟1000个用户, 每秒孵化5个用户, 对着ALB虚拟服务地址192.168.30.101压测1.5小时.
在ALB的界面中我们可以很快的观测到流量压测的情况. 
![](https://i.imgur.com/cDbqSE6.png)
图上标注1号位置触发了扩容事件, 2号位置反馈扩容完成.
![](https://i.imgur.com/d1OinpB.png)
vCenter中观察到正在执行的克隆任务.
最终状态实现后端池从最初的一个后端, 按需自动扩展到了4个. 虚拟服务原本由一个ALB实例承担, 也自动扩展到了2个. 关于ALB实例的水平扩缩可以参考官方文档:https://avinetworks.com/docs/20.1/autoscale-service-engines/
![](https://i.imgur.com/hVh8cpn.png)
停止压测后等待3个扩容冷却时间后, 开始进入缩容的进程, 最总会回到一个ALB实例承担VS, 后端池内剩下一个后端. 回到最初的状态.

## 总结
等研发同事修复了library后, 其实这套解决方案的复用率还是很高的. 在这个前提下配置也并不复杂. 无需借助外部的运维监控系统, 使用ALB内置的度量值即可作为前后端水平扩缩的触发器. 比较简单的让客户可以在私有云中得到类公有云的服务发布体验.
ALB的所有度量值都可以对外暴露, 可以通过API索取或者配置Prometheus索取. 下回有空我们再Demo一下使用Prometheus索取ALB度量值, 触发K8s Deployment水平扩缩.
![](https://i.imgur.com/97EvCIo.jpg)
