::: {align="center"}
# 🛠️ VirtualBox & Kubernetes Lab Recovery Guide

### A practical runbook for VM operations, resource checks, and cluster recovery

![macOS](https://img.shields.io/badge/Host-macOS-black?style=for-the-badge)
![VirtualBox](https://img.shields.io/badge/VirtualBox-VM%20Management-183A61?style=for-the-badge)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.35.9-326CE5?style=for-the-badge)
![containerd](https://img.shields.io/badge/Runtime-containerd-575757?style=for-the-badge)

**Purpose:** A reusable, step-by-step guide to inspect, troubleshoot,
and safely recover the local Kubernetes lab.
:::

------------------------------------------------------------------------

## 📚 Contents

1.  [Lab Overview](#1--lab-overview)
2.  [VirtualBox Commands](#2--virtualbox-commands)
3.  [Change RAM and CPU](#3--change-ram-and-cpu)
4.  [Start and Stop VMs](#4--start-and-stop-vms)
5.  [Connect to the VMs](#5--connect-to-the-vms)
6.  [Check VM Health](#6--check-vm-health)
7.  [Check Disk Space](#7--check-disk-space)
8.  [Check Kubernetes Health](#8--check-kubernetes-health)
9.  [Troubleshooting Toolkit](#9--troubleshooting-toolkit)
10. [Safe Recovery Procedure](#10--safe-recovery-procedure)
11. [New Relic Incident Notes](#11--new-relic-incident-notes)
12. [Quick Reference](#12--quick-reference)

------------------------------------------------------------------------

## 1. 🧭 Lab Overview

### Host machine

  Setting           Value
  ----------------- -----------------------
  Host              MacBook Pro
  Architecture      Apple Silicon (ARM64)
  Physical memory   16 GB
  Hypervisor        Oracle VirtualBox

### Virtual machines

  VM        Role                       IP address          RAM       CPU
  --------- -------------------------- ---------------- ------ ---------
  **VM1**   Kubernetes control plane   `192.168.56.5`     5 GB   3 vCPUs
  **VM2**   Kubernetes worker          `192.168.56.6`     3 GB   3 vCPUs

### Software stack

  Component           Version / detail
  ------------------- --------------------
  Guest OS            Ubuntu 26.04.1 LTS
  Kubernetes          v1.35.9
  Container runtime   containerd

> \[!TIP\] Current VM allocation is 8 GB in total. Keep enough memory
> available for macOS and VirtualBox. Check host memory pressure before
> increasing either VM's allocation.

------------------------------------------------------------------------

## 2. 🖥️ VirtualBox Commands

**Run these commands on the Mac host** in macOS Terminal.

### List all registered VMs

``` bash
VBoxManage list vms
```

### List currently running VMs

``` bash
VBoxManage list runningvms
```

### View VM configuration

``` bash
VBoxManage showvminfo "VM1"
VBoxManage showvminfo "VM2"
```

### View RAM and CPU settings only

``` bash
VBoxManage showvminfo "VM1" | grep -E "Memory size|Number of CPUs"
```

------------------------------------------------------------------------

## 3. ⚙️ Change RAM and CPU

> \[!WARNING\] Fully power off the VM before changing its RAM or CPU
> settings. Do not modify resources while it is running or saved.

### VM1 --- 5 GB RAM, 3 CPUs

``` bash
VBoxManage modifyvm "VM1" --memory 5120 --cpus 3
```

### VM2 --- 3 GB RAM, 3 CPUs

``` bash
VBoxManage modifyvm "VM2" --memory 3072 --cpus 3
```

### Verify the settings

``` bash
VBoxManage showvminfo "VM1" | grep -E "Memory size|Number of CPUs"
VBoxManage showvminfo "VM2" | grep -E "Memory size|Number of CPUs"
```

**Memory conversion:** VirtualBox uses MB. `5120` = 5 GB; `3072` = 3 GB.

------------------------------------------------------------------------

## 4. ▶️ Start and Stop VMs

### Start VM1

``` bash
VBoxManage startvm "VM1" --type gui
```

### Start VM2

``` bash
VBoxManage startvm "VM2" --type gui
```

### Gracefully shut down from Ubuntu

``` bash
sudo shutdown -h now
```

### Request graceful shutdown from macOS

``` bash
VBoxManage controlvm "VM1" acpipowerbutton
VBoxManage controlvm "VM2" acpipowerbutton
```

### Check VM state

``` bash
VBoxManage showvminfo "VM1" | grep -i "State"
VBoxManage showvminfo "VM2" | grep -i "State"
```

> \[!CAUTION\] Prefer graceful shutdown. Force-killing a VM can lose
> unsaved in-memory state and interrupt disk writes. It normally does
> not delete the virtual disk, but force termination should be a last
> resort. Never terminate unrelated VirtualBox processes.

------------------------------------------------------------------------

## 5. 🔐 Connect to the VMs

Run SSH commands from macOS Terminal.

  VM                      SSH command
  ----------------------- --------------------------
  VM1 --- Control plane   `ssh admin@192.168.56.5`
  VM2 --- Worker          `ssh admin@192.168.56.6`

------------------------------------------------------------------------

## 6. 📊 Check VM Health

Run these commands **inside the relevant Ubuntu VM**.

  Check            Command            Purpose
  ---------------- ------------------ -----------------------------------
  Memory           `free -h`          RAM usage and available memory
  CPU load         `uptime`           Uptime and load averages
  Live processes   `top`              Per-process CPU and memory
  CPU count        `nproc`            Available logical CPUs
  OS version       `lsb_release -a`   Ubuntu release
  Kernel           `uname -r`         Running kernel version
  System details   `hostnamectl`      Host, OS, kernel and architecture

### Memory

``` bash
free -h
```

Focus on the **available** column. Linux uses free memory for cache, so
low `free` memory by itself does not necessarily mean the VM is out of
memory.

### CPU and load

``` bash
uptime
top
```

Press `q` to exit `top`.

------------------------------------------------------------------------

## 7. 💽 Check Disk Space

### Filesystem usage

``` bash
df -h
```

### Disk and partition layout

``` bash
lsblk
```

### Find large directories under `/var`

``` bash
sudo du -xh /var --max-depth=1 2>/dev/null | sort -h
```

### Check containerd storage size

``` bash
sudo du -sh /var/lib/containerd
```

### If Docker is installed

``` bash
docker system df
```

> \[!WARNING\] Do not manually delete files from `/var/lib/containerd`,
> `/var/lib/kubelet`, or Kubernetes data directories. Identify what is
> consuming disk space and use the appropriate cleanup procedure.

------------------------------------------------------------------------

## 8. ☸️ Check Kubernetes Health

Run cluster-wide administrator commands from **VM1**.

### Node status

``` bash
kubectl get nodes -o wide
```

Expected nodes:

-   `vm1` --- control plane
-   `vm2` --- worker

Both should eventually show `Ready`.

### Pods across all namespaces

``` bash
kubectl get pods -A
```

### Include pod placement

``` bash
kubectl get pods -A -o wide
```

### Node resource usage

``` bash
kubectl top nodes
```

### Pod resource usage

``` bash
kubectl top pods -A
```

> \[!NOTE\] `kubectl top` requires Metrics Server. If metrics are
> unavailable, check Metrics Server before assuming the node itself is
> unhealthy.

### Check containerd and kubelet

``` bash
sudo systemctl status containerd --no-pager
sudo systemctl status kubelet --no-pager
```

------------------------------------------------------------------------

## 9. 🔎 Troubleshooting Toolkit

### Kubelet logs --- latest 100 lines

``` bash
sudo journalctl -u kubelet -n 100 --no-pager
```

### Follow kubelet logs live

``` bash
sudo journalctl -u kubelet -f
```

Press `Ctrl+C` to stop.

### Containerd logs

``` bash
sudo journalctl -u containerd -n 100 --no-pager
```

### List containers through CRI

``` bash
sudo crictl ps -a
```

### List container images

``` bash
sudo crictl images
```

### Inspect container logs

``` bash
sudo crictl logs <container-id>
```

### Describe a Kubernetes pod

``` bash
kubectl describe pod <pod-name> -n <namespace>
```

### Review recent Kubernetes events

``` bash
kubectl get events -A --sort-by=.lastTimestamp
```

### Restart services only when justified by troubleshooting

``` bash
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Then validate:

``` bash
kubectl get nodes
kubectl get pods -A
```

> \[!IMPORTANT\] `crictl` inspects containers through the node's
> container runtime. It is useful for node-level troubleshooting, but it
> does not replace `kubectl` for managing Kubernetes resources.

### Worker kubeconfig reminder

VM2's `/etc/kubernetes/kubelet.conf` identifies the kubelet as a node.
It is **not** a cluster-admin kubeconfig. Use VM1 or a separately
authorized administrator kubeconfig for cluster-wide `kubectl` commands.

------------------------------------------------------------------------

## 10. 🧯 Safe Recovery Procedure

Use this sequence if VirtualBox freezes, VMs stop unexpectedly, or the
Kubernetes cluster becomes unhealthy.

### Recovery flow

``` text
                 macOS Terminal
                       |
                       v
              List VirtualBox VMs
                       |
                       v
                 Start VM1 only
                       |
                       v
             SSH into the control plane
                       |
                       v
           Check memory, CPU and disk
                       |
                       v
          Check containerd and kubelet
                       |
                       v
           Check Kubernetes nodes/pods
                       |
                       v
                 Start VM2
                       |
                       v
             Validate worker readiness
                       |
                       v
              Validate the full cluster
```

### Step 1 --- Inspect VM states on macOS

``` bash
VBoxManage list vms
VBoxManage list runningvms
```

### Step 2 --- Start VM1 only

``` bash
VBoxManage startvm "VM1" --type gui
```

Wait for Ubuntu to boot.

### Step 3 --- SSH into VM1

``` bash
ssh admin@192.168.56.5
```

### Step 4 --- Check VM1 resources

``` bash
free -h
uptime
df -h
```

### Step 5 --- Check node services

``` bash
sudo systemctl status containerd kubelet --no-pager
```

### Step 6 --- Validate the control plane

``` bash
kubectl get nodes -o wide
kubectl get pods -A
```

**Do not start VM2 until VM1 has been checked and the control plane is
responding.**

### Step 7 --- Start VM2 from macOS

Open another macOS Terminal window:

``` bash
VBoxManage startvm "VM2" --type gui
```

Wait for VM2 to boot.

### Step 8 --- Validate the worker from VM1

``` bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

If VM2 remains `NotReady`, SSH into VM2 and inspect:

``` bash
sudo systemctl status containerd kubelet --no-pager
sudo journalctl -u kubelet -n 100 --no-pager
sudo journalctl -u containerd -n 100 --no-pager
```

Allow time for the worker to reconnect. If it remains unhealthy,
investigate the logs and Kubernetes events before restarting services
repeatedly.

------------------------------------------------------------------------

## 11. 📝 New Relic Incident Notes

During an earlier lab recovery, we experienced VirtualBox and Kubernetes
instability. New Relic components were present in the cluster during the
investigation.

We removed the `newrelic` namespace as part of troubleshooting. After
that change, the environment recovered and appeared stable in our
subsequent checks.

**Observed:** The lab recovered after the namespace was removed.

**Not confirmed:** This observation does not prove New Relic was the
root cause. Resource pressure, VM restarts, and other cluster conditions
may also have contributed.

**Impact:** Removing the New Relic deployment stops its associated
telemetry collection. New Relic is not a Kubernetes control-plane
requirement.

------------------------------------------------------------------------

## 12. ⚡ Quick Reference

```{=html}
<details>
```
```{=html}
<summary>
```
`<strong>`{=html}VirtualBox --- macOS Terminal`</strong>`{=html}
```{=html}
</summary>
```
``` bash
VBoxManage list vms
VBoxManage list runningvms
VBoxManage showvminfo "VM1"
VBoxManage startvm "VM1" --type gui
VBoxManage controlvm "VM1" acpipowerbutton
VBoxManage modifyvm "VM1" --memory 5120 --cpus 3
```

```{=html}
</details>
```
```{=html}
<details>
```
```{=html}
<summary>
```
`<strong>`{=html}Ubuntu --- VM health and disk`</strong>`{=html}
```{=html}
</summary>
```
``` bash
free -h
uptime
top
nproc
df -h
lsblk
hostnamectl
```

```{=html}
</details>
```
```{=html}
<details>
```
```{=html}
<summary>
```
`<strong>`{=html}Kubernetes --- run from VM1`</strong>`{=html}
```{=html}
</summary>
```
``` bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pods -A -o wide
kubectl top nodes
kubectl top pods -A
kubectl get events -A --sort-by=.lastTimestamp
```

```{=html}
</details>
```
```{=html}
<details>
```
```{=html}
<summary>
```
`<strong>`{=html}Services and containers --- run on the relevant
VM`</strong>`{=html}
```{=html}
</summary>
```
``` bash
sudo systemctl status containerd kubelet --no-pager
sudo journalctl -u kubelet -n 100 --no-pager
sudo journalctl -u containerd -n 100 --no-pager
sudo crictl ps -a
sudo crictl images
```

```{=html}
</details>
```

------------------------------------------------------------------------

## ✅ Operational Checklist

-   [ ] Check VM state before starting or modifying it.
-   [ ] Start VM1 first; validate the control plane before starting VM2.
-   [ ] Monitor host and guest memory usage.
-   [ ] Check disk capacity before deploying more workloads.
-   [ ] Review logs before restarting services.
-   [ ] Change one thing at a time and validate after each change.
-   [ ] Prefer graceful VM shutdown over force termination.
-   [ ] Record incident symptoms, commands, changes, and results.
-   [ ] Never use worker kubelet credentials for cluster administration.

------------------------------------------------------------------------

::: {align="center"}
### Diagnose → Fix → Validate → Document

**A repeatable recovery process keeps the lab maintainable.**

*Runbook for the local VirtualBox Kubernetes lab.*
:::
