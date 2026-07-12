# WHY — read this on the hard days

> You will hit a week (probably in Phase 3, somewhere in KubeVirt or Longhorn source) where nothing makes sense and you feel dumb. That feeling **is the moat being built.** It's the exact wall that turns 90% of people back — which is exactly why the people past it are rare. Reread this, then go one more layer down.

---

## The one sentence

**Almost no one is willing to go all the way down — so the few who do are nearly irreplaceable. And you're already further down than you think.**

---

## You can be *that* engineer

The one who, when the room is stuck and the senior people are guessing, quietly opens the source, reads for twenty minutes, and says *"found it."* The person whose name comes up when something is **truly** broken. The "how does he just *know* that?" person.

That person isn't smarter. They just refused to stop at the abstraction and went one layer deeper, every time, until there were no layers left.

---

## You already started building this (facts, not flattery)

- You **built a real multi-cluster system** — a provisioner deploying Harbor across Harvester via cross-cluster kubeconfig. Most engineers never touch multi-cluster. That's not a tutorial project.
- You already understand the **networking deeply** — VLANs, the `harvester-br0` bridge, Calico's IPIP tunnel, the full 5-hop external→pod path. That's the layer people find *hardest*.
- You're already **in the Go codebase**, chapter by chapter — learning to read the exact language Harvester, Rancher, KubeVirt, and Longhorn are written in.
- You have a **real cluster, real access, real problems** to learn on — the thing most learners never get.

You're not starting at zero and hoping. You're standing on a foundation most people never lay, asking to go one floor higher. The 3-month plan isn't a moonshot — it's the next floor on a building you've already poured.

---

## Why code-level is the multiplier (not a nice-to-have)

**The operator** hits a broken cluster, greps the docs, tries three Stack Overflow answers, and when none work — opens a vendor ticket and waits. Their ceiling is *"what's documented."*

**You, code-level**, hit the same cluster at 3am with nothing in the docs. So you go: *which resource → which controller's `OnChange` → read the function → "it requeues forever because this field is nil."* You fix in an hour what the operator escalates for three days. Your ceiling is *"what the code actually does"* — which is everything.

That one difference is the line between Senior and Staff/Principal. Promotion past senior is almost never about working harder — it's about solving the class of problem nobody else can. **Reading source is that ability, generalized.**

---

## What it opens

- **The level ladder:** this is the literal path to **Staff / Principal SRE / Platform Architect.** Depth others can't replicate is the only durable moat at that level.
- **Global + remote:** deep Kubernetes/SRE skill is one of the most geography-independent skills in tech — measured by what you can do, demonstrable in public, hired remotely worldwide. You compete on depth, not location. Depth pays top-of-market everywhere.
- **Ecosystem reputation:** a few merged PRs into `harvester/harvester`, `longhorn`, or a CNCF project and your GitHub becomes a credential no résumé bullet can match. Maintainership is a career in itself.
- **Inside WSO2:** you become *the* platform person — the one who designs the infra everyone depends on and can't ship without. Not job security. Leverage.

---

## Why it's worth 3 months: it doesn't rot

Most things you could learn in 3 months are obsolete in a few years — this framework, that library. **Systems fundamentals don't decay.** TCP/IP, Linux, how a control loop reconciles state, how storage replicates, how a packet crosses a bridge — identical in 2016, 2026, 2036. Kubernetes won; it's infrastructure now, like Linux. Every hour in the *deep* layer **compounds for your whole career** instead of resetting.

You're not learning a tool. You're learning **how this entire class of systems works — permanently.**

---

## The deal

It's hard, and it's slow, and the wall is real. The retention system and the "behavior first, then read its source the same day" sequencing exist precisely to get you over that wall without quitting.

Go one layer deeper. Every time. That's the whole secret.

---

## Does anyone actually do all 8 phases?

**Almost no one does the whole vertical — that's the entire point.**

The world is full of specialists who each own *one* phase: kernel contributors (Phase 8 L1) who've never provisioned a Rancher cluster; Rancher engineers (Phases 5–6) who've never read `kvm_cpu_exec()`; KubeVirt maintainers who know QEMU's interface but not Longhorn's engine. Every *piece* is done by someone. What's rare — rare enough that the people who have it are distinguished engineers, cross-project maintainers, the ones giving keynotes and writing the books — is **one person who spans the whole stack**, from the BMC through the kernel's C up through the apiserver, Rancher, Harvester, KubeVirt, Longhorn, and the netfilter rules.

Not 100% of every line — nobody has that. But the **architecture of all of it + the critical-path code + the idea-level C**, in one head, navigable on demand. It's rare not because it's impossible, but because **almost everyone stops at their one layer.** You're deciding not to.

## What you'll literally be able to do at the end of 8 phases

- **A VM won't boot** → you trace `VirtualMachine` CRD → virt-controller → virt-launcher → libvirt `qemuBuildCommandLine()` → real QEMU argv → `kvm_cpu_exec()`. Found.
- **A pod can't reach a Service** → pod → kube-proxy → `nft list ruleset` → the DNAT rule → `conntrack -L` → the kernel hook path. The exact broken hop.
- **A Longhorn volume degrades** → you read the replica/engine controller *and* the iSCSI/device-mapper stack it rides on.
- **A Rancher cluster hangs provisioning** → provisioningv2 controller → Cluster API → the `remotedialer` tunnel to the agent. You see where it stops.
- **etcd is slow** → you reason about raft + MVCC + bbolt compaction and know if it's disk, quorum, or a watch storm.
- **A reconcile loops forever** → you open the `OnChange` handler, spot the requeue-on-nil bug, fix it in an hour.

That's not "a person who knows Kubernetes." That's **the person the org routes its undebuggable problems to — who then fixes the upstream project instead of filing a ticket.**

## Who you are at the end

- **Level:** the profile of a Staff / Principal / Distinguished engineer — depth nobody around you can replicate.
- **Open source:** able to contribute to *any* of these projects; a few merged PRs and your GitHub is a globally portable, remote-friendly credential.
- **At WSO2:** the platform's center of gravity — it can't ship without you.
- **Personally:** you stop being afraid of *any* system. No magic layer left, no box you can't open. That confidence is permanent and generalizes to every new technology for the rest of your career.

The end of 8 phases isn't an end — it's a launchpad. You won't have read every line (nobody has), but you'll have crossed the line 99% never cross, and everything after compounds on top of it.

**Does anyone do this? Almost no one. Which is exactly why being the one who did is worth the next eight phases.**

---

→ Back to the plan: [`00-curriculum.md`](./00-curriculum.md)
