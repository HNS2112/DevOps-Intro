# Lab 5 — Virtualization: QuickNotes in a Vagrant VM

**Author:** HNS ([@HNS2112](https://github.com/HNS2112))
**Date:** 7 August 2026
**Host:** Linux Mint 22 (Ubuntu 24.04 base), kernel 7.0.0-28, 12 cores, 15 GB RAM
**Tooling:** Vagrant 2.4.9, VirtualBox 7.1.18

Raw command output is committed under `evidence/lab5/`.

---

## Host setup — and why VirtualBox 7.0 could not be used

Worth recording before the lab proper, because it is a real constraint rather
than a configuration mistake.

The distribution package `virtualbox-dkms 7.0.16` fails to build against kernel
7.0. The DKMS log is explicit:

```
ERROR: modpost: module vboxdrv uses symbol kvm_enable_virtualization from
       namespace module:kvm-amd,kvm-intel, but does not import it.
ERROR: modpost: module vboxdrv uses symbol cr4_update_irqsoff from
       namespace module:kvm,kvm-amd,kvm-intel, but does not import it.
```

Recent kernels moved these virtualisation-control symbols into a KVM-private
namespace. VirtualBox 7.0 calls them without declaring the import, so the module
cannot link. This machine also runs KVM (`kvm_intel` loaded, libvirt installed
for other work), which is exactly the coexistence case that broke.

VirtualBox **7.1.18** from Oracle's own repository builds and loads cleanly on
the same kernel — 7.1 is the release where VirtualBox learned to run as a KVM
guest hypervisor rather than fighting it for the CPU:

```console
$ VBoxManage --version
7.1.18r173720

$ lsmod | grep vbox
vboxnetadp             28672  0
vboxnetflt             40960  0
vboxdrv               716800  2 vboxnetadp,vboxnetflt
```

Two smaller distribution issues along the way: `lsb_release -cs` returns `zena`
on Linux Mint, which is not a codename HashiCorp publishes, so the apt line has
to say `noble` explicitly; and Oracle's old signing-key URL now returns HTTP 410,
with the key having moved under the repository path.

---

## Task 1 — Vagrant Up + QuickNotes Inside

### 1.1 The Vagrantfile

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"
  config.vm.hostname = "quicknotes-vm"

  config.vm.network "forwarded_port",
    guest: 8080,
    host: 18080,
    host_ip: "127.0.0.1"

  config.vm.synced_folder "./app", "/opt/quicknotes"

  config.vm.provider "virtualbox" do |vb|
    vb.cpus = 2
    vb.memory = 1024
  end

  config.vm.provision "shell", inline: <<-SHELL
    set -eux
    GO_VERSION=1.24.6
    curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz" -o /tmp/go.tgz
    rm -rf /usr/local/go
    tar -C /usr/local -xzf /tmp/go.tgz
    echo 'export PATH=$PATH:/usr/local/go/bin' > /etc/profile.d/go.sh
    chmod +x /etc/profile.d/go.sh
    /usr/local/go/bin/go version
  SHELL
end
```

`.vagrant/` is in `.gitignore` — it is per-machine state, not source.

### 1.2 First `vagrant up`

```console
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Box 'bento/ubuntu-24.04' could not be found. Attempting to find and install...
    default: Box Provider: virtualbox
    default: Box Version: >= 0
==> default: Loading metadata for box 'bento/ubuntu-24.04'
    default: URL: https://vagrantcloud.com/api/v2/vagrant/bento/ubuntu-24.04
==> default: Adding box 'bento/ubuntu-24.04' (v202510.26.0) for provider: virtualbox (amd64)
    default: Downloading: https://vagrantcloud.com/bento/boxes/ubuntu-24.04/versions/202510.26.0/providers/virtualbox/amd64/vagrant.box
==> default: Successfully added box 'bento/ubuntu-24.04' (v202510.26.0) for 'virtualbox (amd64)'!
==> default: Importing base box 'bento/ubuntu-24.04'...
==> default: Matching MAC address for NAT networking...
```

Provisioning tail:

```console
==> default: Mounting shared folders...
    default: /home/hns/dev/DevOps-Intro => /vagrant
    default: /home/hns/dev/DevOps-Intro/app => /opt/quicknotes
==> default: Running provisioner: shell...
    default: ++ GO_VERSION=1.24.6
    default: ++ curl -fsSL https://go.dev/dl/go1.24.6.linux-amd64.tar.gz -o /tmp/go.tgz
    default: ++ tar -C /usr/local -xzf /tmp/go.tgz
    default: ++ /usr/local/go/bin/go version
    default: go version go1.24.6 linux/amd64
```

### 1.3 Verification

```console
$ vagrant ssh -c 'go version'
go version go1.24.6 linux/amd64

$ vagrant ssh -c 'cd /opt/quicknotes && go build -o /tmp/qn . && ls -la /tmp/qn'
-rwxrwxr-x 1 vagrant vagrant 8654192 Aug  7 09:01 /tmp/qn

$ vagrant ssh -c 'cd /opt/quicknotes && ADDR=0.0.0.0:8080 DATA_PATH=/tmp/notes.json \
    setsid nohup /tmp/qn > /tmp/qn.log 2>&1 < /dev/null & sleep 3; \
    cat /tmp/qn.log; curl -s http://localhost:8080/health'
2026/08/07 09:02:37 quicknotes listening on 0.0.0.0:8080 (notes loaded: 4)
{"notes":4,"status":"ok"}

$ curl -s http://localhost:18080/health          # from the host
{"notes":4,"status":"ok"}
```

**Two things had to be set explicitly, and both are worth naming.**

`ADDR=0.0.0.0:8080`. QuickNotes defaults to `:8080`, which Go binds to all
interfaces — but the first attempts produced an empty response on the host. The
forwarded port only reaches a listener bound to an address the NAT interface can
deliver to, so being explicit removed the ambiguity. This is the same
bind-address question as Lab 4 §1.2, seen from the opposite side: there the
concern was that binding to `*` exposed the service too widely; here the concern
is that binding too narrowly makes it unreachable.

`DATA_PATH=/tmp/notes.json`. The default is `data/notes.json`, relative to the
working directory — which is `/opt/quicknotes`, a VirtualBox shared folder owned
by the host. Writing there through vboxsf is unreliable, so the data file was
moved to the guest's own filesystem. The same relative-path behaviour caused the
silent empty-store problem in Lab 1 and the `permission denied` crash loop in
Lab 6; three labs, three different symptoms, one root cause.

### 1.4 Design questions

#### a) Which synced-folder type, and what is the trade-off?

The default **VirtualBox shared folders** (vboxsf), chosen because it needs no
extra host packages and — critically for this lab — syncs **bidirectionally and
continuously**. Editing `app/main.go` on the host is visible in the guest
immediately, which is what makes the edit-build-test loop usable.

The trade-off is performance and semantics. vboxsf is a kernel-level shim over
the host filesystem and is markedly slower than native I/O for many small files;
Go builds over it are noticeably slower than in-guest builds. It also does not
reproduce POSIX ownership faithfully, which is why `DATA_PATH` had to be moved
off it.

`rsync` would have been faster inside the guest, because files land on the
guest's own disk — but it is a **one-way, on-demand copy**: changes made on the
host require `vagrant rsync` to appear, and changes made inside the guest are
lost on the next sync. `nfs` gives near-native speed with two-way sync but needs
an NFS server on the host and sudo for exports. `smb` is the Windows-host
equivalent. For a lab where the host edits and the guest runs, vboxsf's
convenience outweighs its speed cost; for a large repository with a slow build,
rsync would be the better call.

#### b) Which network mode, and why is `127.0.0.1` port forwarding safer than bridged?

**NAT**, the Vagrant default. The guest sits behind a virtual NAT device, gets a
private address, can reach the internet outbound, and is unreachable from outside
except through explicitly forwarded ports.

`host_ip: "127.0.0.1"` narrows that further: the forwarded port binds only to the
host's loopback interface, so `18080` is reachable from this machine and from
nothing else. Without it, VirtualBox binds `0.0.0.0` and the VM's service becomes
available to everyone on the LAN.

A **bridged** interface would put the VM directly on the physical network with its
own address, as if it were a separate machine on the LAN. For a course exercise
that is a bad default for three reasons: QuickNotes has no authentication, so
anyone on the network could read and write notes; the VM is unpatched beyond the
box image; and on a shared or campus network the exposure is not to a handful of
trusted machines. NAT plus loopback-bound forwarding is deny-by-default with one
deliberate exception — the same principle as `cap_drop: ALL` in Lab 6.

#### c) Which provisioner, and why?

**`shell`**, for three reasons. The task is a handful of ordered commands —
download a tarball, extract it, set PATH — with no branching or state to manage,
which is the exact shape shell handles well. It requires nothing installed on
either side: `ansible` needs Ansible on the host, `ansible_local` needs it
installed into the guest first, and puppet or chef need a whole agent for what is
four lines. And it keeps the `Vagrantfile` self-contained, so a clean clone plus
`vagrant up` reproduces the environment with no external files.

The limit is idempotency. This script is idempotent by construction —
`rm -rf /usr/local/go` before extracting means re-running produces the same
result — but that is a property I had to build in deliberately, not one the tool
guarantees. A longer provisioning script would drift toward reimplementing what
Ansible gives for free, which is precisely why Lab 7 replaces this with an
Ansible playbook against the same VM.

#### d) Why pin `1.24.6` rather than `1.24`?

Because `1.24` is not a version, it is a moving pointer. Anyone running
`vagrant up` next month against `1.24` gets whatever the newest patch is by then;
`1.24.6` gets the same bytes today, next month, and after the box is rebuilt.
Requirement 7 asks for exactly this — that another student from a clean clone
produces the same working state.

The concrete failure the pin prevents: a Go patch release changes behaviour, the
VM stops matching what CI runs, and the difference is invisible because the
`Vagrantfile` did not change. That is the "works on my machine" bug the whole
lab exists to eliminate.

The cost is the same one Lab 3 documented for SHA-pinned actions and Lab 9 found
in the Trivy scan: a pin that is never revisited accumulates unpatched CVEs.
Pinning buys control over *when* the update happens, not exemption from it.

---

## Task 2 — Snapshots: Save, Break, Restore

### 2.1 The cycle

```console
$ vagrant snapshot save clean-go-installed
==> default: Snapshotting the machine as 'clean-go-installed'...
==> default: Snapshot saved!

$ vagrant ssh -c 'sudo rm -rf /usr/local/go && echo "Go removed"'
Go removed

$ vagrant ssh -c 'go version || echo "BROKEN: go not found"'
bash: line 1: go: command not found
BROKEN: go not found

$ time vagrant snapshot restore clean-go-installed
==> default: Forcing shutdown of VM...
==> default: Restoring the snapshot 'clean-go-installed'...
==> default: Booting VM...
==> default: Machine booted and ready!

real	0m23,639s
user	0m2,052s
sys	0m1,636s

$ vagrant ssh -c 'go version'
go version go1.24.6 linux/amd64
```

Restore took **23.6 seconds** from a deliberately destroyed toolchain back to a
working one — inside the 30 seconds the lab brief promises.

The lecture's framing is the point: nobody diagnosed *why* Go was missing or
tried to repair it. The VM was discarded and replaced from a known-good state.
That is only possible because nothing irreplaceable lived inside it.

### 2.2 Design questions

#### e) Why snapshots are not backups

A snapshot lives on the same disk, in the same host, inside the same VirtualBox
installation as the thing it protects. Every failure that takes out the host takes
out the snapshot with it: a dead drive, a stolen laptop, a filesystem corruption,
`vagrant destroy`, ransomware encrypting the VMs directory.

It is also useless against slow corruption. A snapshot from before a bug was
introduced is a rollback; a snapshot taken *after* it faithfully preserves the
bug. If the problem is discovered weeks later, the snapshot chain may contain no
clean state at all.

The distinction is that a backup is a copy that is **independent** of the original
— different medium, ideally different location, with its own retention. A snapshot
is a cheap point-in-time marker within one system. Snapshots protect against
"I broke it in the last hour"; backups protect against "the machine is gone".

#### f) Copy-on-write and ten snapshots

Under VirtualBox a snapshot does not copy the disk. It freezes the current disk
image read-only and creates a differencing image that receives all subsequent
writes. Taking a snapshot is therefore nearly instant and initially almost free —
which is what the near-immediate `snapshot save` above demonstrates.

Ten snapshots do not cost ten times one, because each holds only the blocks
changed since the previous one. Total disk is the base image plus the sum of the
deltas. Whether that is small or enormous depends entirely on write volume: ten
snapshots around a few config edits cost megabytes, ten around a package upgrade
or a rebuilt Go toolchain cost gigabytes.

The non-obvious cost is read performance rather than space. With ten snapshots
the hypervisor may walk a ten-deep chain to find the current version of a block,
and the VM gets slower as the chain grows.

#### g) When snapshotting is an antipattern

**Long chains** are the headline case, and the mechanism is the one above: every
layer adds indirection to reads, so a VM with dozens of snapshots degrades
measurably. Deleting a snapshot mid-chain requires merging differencing images,
which is slow, disk-hungry, and the operation most likely to lose data.

The deeper antipattern is using snapshots as a substitute for reproducibility.
If the only way to rebuild an environment is "restore the snapshot from March",
that environment has become a pet — nobody knows how it was built, and it cannot
be recreated on new hardware. The `Vagrantfile` in this lab is the alternative:
`vagrant destroy && vagrant up` gets to a known state from source in a couple of
minutes, and that state is reviewable in a PR. The snapshot is a convenience for
a fast inner loop, not the record of how the machine came to be.

Snapshots are also a poor fit for anything holding state that matters. Rolling
back a VM rolls back its data too, which is fine for a toolchain and destructive
for a database.

---

## Bonus Task — VM vs Container Resource Baseline

Both measured on the same machine, in the same session, running the same
QuickNotes application. The container uses the Lab 6 image
(multi-stage, distroless, `CGO_ENABLED=0`).

### B.1 Measurements

```console
=== VM: cold boot ===              === Container: cold start ===
real	0m24,846s                  real	0m0,125s

=== VM: idle RAM ===               === Container: idle RAM ===
Mem: 961Mi total, 319Mi used       MEM USAGE: 3.207MiB / 15.42GiB

=== VM: process count ===          === Container: process count ===
161                                1

=== VM: disk size ===              === Container: image size ===
2,7G                               9.22MB
```

### B.2 Comparison

| Dimension | Vagrant VM | Docker container | Ratio |
|---|---|---|---|
| Cold start | 24.8 s | 0.125 s | **198×** |
| Idle RAM | 319 MB | 3.2 MB | **100×** |
| On-disk size | 2.7 GB | 9.22 MB | **293×** |
| Process count (guest) | 161 | 1 | **161×** |

### B.3 Analysis

The number that surprised me is the process count: **161 versus 1**. The VM was
idle and doing nothing but running one Go binary, yet it carried a full init
system, journald, cron, snapd, systemd-resolved, an SSH daemon and a hundred
other things — none of which QuickNotes needs. Everything else follows from that
one fact. The 319 MB of idle RAM is those processes; the 24.8 seconds of boot is
starting them in order; the 2.7 GB on disk is the filesystem they live in. The
container's single process is the application and nothing else.

The 198× start-up gap explains the industry shift better than any of the others.
A container start is `fork`, `exec` and a set of namespaces — no kernel boot, no
firmware, no service dependency graph. That is the difference between an
autoscaler that reacts in a second and one that reacts in half a minute, and
between rolling a deployment in seconds and in minutes.

**Containers are the right tool for stateless workloads that share a kernel with
their neighbours**: HTTP services, workers, CI jobs, anything that scales
horizontally and can be replaced rather than repaired. Density is the argument —
15 GB of RAM holds roughly 48 of these VMs at their idle footprint and several
thousand of these containers.

**VMs are the right tool where the kernel boundary matters.** Running a different
kernel or OS than the host, hard multi-tenant isolation between untrusted
workloads, kernel-level work such as drivers or eBPF development, and anything
where a container escape is an unacceptable risk. The 2.7 GB and the 161
processes are the price of a real machine boundary, and sometimes that boundary
is exactly what you are buying.

That is also why the comparison is not "containers won". The 2014–2020 era of
stateless microservices was the workload where the VM's isolation was overkill
and its cost was pure overhead — and this table shows the overhead is two orders
of magnitude in every dimension. For workloads where the isolation is the point,
the same numbers read as a bill worth paying, which is why Firecracker and
gVisor exist to sell the boundary back at a lower price.

---

## Summary

| Task | Status |
|------|--------|
| Task 1 — Vagrantfile, VM, Go 1.24.6, port forward verified | Complete |
| Task 2 — Snapshot save → break → restore in 23.6 s | Complete |
| Bonus — VM vs container baseline with analysis | Complete |
