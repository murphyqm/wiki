# Install Slurm on WSL: test environment

Step-by-step install and setup of Slurm on WSL2 for testing purposes. Mainly followed this [guide: Slurm for Dummies](https://github.com/SergioMEV/slurm-for-dummies) with some modifications for a single-node WSL2 mini set-up.

Obviously this isn't going to let you test all config changes, but will allow messing with user accounts/QoS permissions etc.

I've just copied the commands I used below in order; WSL2 was Ubuntu 24.05.5 LTS. Obviously, again, not matching the OS of the cluster.

```bash
sudo apt update
sudo apt upgrade
sudo apt install munge libmunge2 libmunge-dev

# to test if it worked
munge -n | unmunge | grep STATUS

# fix permissions
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
sudo chmod 0755 /run/munge/
sudo chmod 0700 /etc/munge/munge.key
sudo chown -R munge: /etc/munge/munge.key

# so that it runs on startup
systemctl enable munge
systemctl restart munge

# checking if it works
systemctl status munge

# Install slurm
sudo apt install slurm-wlm

# To find config file
cd /usr/share/doc/slurmctld/
explorer.exe .
# launch the html file configure template

# to figure out name for ControlMachine, etc.:
hostname

# for cores
slurmd -C

# for memory, - approx 512 MB for safety
free -m
```

Output from `slurmd -C`:

```bash
NodeName=<hostname-output> CPUs=20 Boards=1 SocketsPerBoard=1 CoresPerSocket=10 ThreadsPerCore=2 RealMemory=15707
UpTime=0-08:37:17
```

Config settings:

- ClusterName: airetest
- SlurmctldHost: <hostname-output>
- SlurmUser: slurm
- NodeName: <hostname-output>
- RealMemory: 14500

See other output from `slurmd -C` above for CPUs etc.

- ProctrackType: proctrack/linuxproc
- TaskPlugin: task/none
- SelectType: select/cons_tres
- SelectTypeParameters: CR_Core_Memory
- JobAcctGatherType: jobacct_gather/linux (mimicking current set-up, also avoids fragile WSL cgroup shenanigans)

Other settings left default.

Click submit, and it generates a config file for you, copy and paste this into the following file:

```bash
sudo nano /etc/slurm/slurm.conf

systemctl enable slurmctld
systemctl restart slurmctld

systemctl status slurmctld
sinfo
```

Can then submit a little test job to see what happens:

```bash
#!/bin/bash
sleep 30
env
```

```bash
# 1) See node state/reason
sinfo -N -l
scontrol show node <hostname-output> | egrep "State=|Reason=|CfgTRES|RealMemory|CPUTot"

# 2) Check daemons/auth
systemctl is-active munge slurmctld slurmd

# 3) If any are not active:
sudo systemctl restart munge slurmctld slurmd

# 4) Clear DOWN/DRAIN
sudo scontrol update NodeName=<hostname-output> State=RESUME

# 5) Recheck
sinfo -N -l
squeue

```

Ended up having to also run:

```bash
sudo systemctl enable --now munge slurmctld slurmd
sudo systemctl status slurmd --no-pager -l
sudo journalctl -u slurmd -n 80 --no-pager

sudo scontrol update NodeName=<hostname-output> State=RESUME
```

Jobs can be submitted, queue and run. I'll have to set up `sacctmgr` to properly test accounts/QoS/priority behaviour etc., but that's for another day.