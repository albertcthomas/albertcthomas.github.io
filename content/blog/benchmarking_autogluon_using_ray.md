+++
type = "post"
title = "Benchmarking AutoGluon using Ray"
date = 2025-10-07
+++

The story behind this one: my paternity leave was imminent and I wanted to have a benchmark of [AutoGluon](https://auto.gluon.ai/dev/index.html) running on a bunch of tabular Kaggle challenges while I was off. At least this was the initial objective. The goal was to compare AutoGluon with our own AutoML solution.

My plan was to use [Ray](https://docs.ray.io/en/latest/index.html) to schedule tasks across several VMs automatically and benefit from all its features: the dashboard to monitor tasks and the load on each machine, stop tasks taking too much time, retry failed ones, and so on. To be honest, I also just wanted to try Ray for this. I knew it should be possible, and this was a good excuse to dive in. I had previously used `ray.tune` on a single machine but had never set up a cluster across multiple VMs. Although everything eventually worked the way I wanted, it was more difficult than I anticipated and required quite a bit of debugging and babysitting at first. I also learned a lot about Ray (and AutoGluon) in the process, so I wanted to share some of those lessons here.

#### Enforcing resource constraints for AutoML comparisons

One of the key aspects of running this benchmark was making sure that we could enforce computational resource constraints for each run. This is important if we want to make fair comparisons between different AutoML solutions under controlled conditions. In practice, this means fixing the number of CPUs (and GPUs) and the amount of memory available to each run.

As written in [AMLB, An AutoML Benchmark](https://arxiv.org/abs/2207.12560), running each task on a specific EC2 instance type ensures consistent resources. I followed a similar principle here. Instead of relying on large shared machines, I used a pool of identical VMs and aimed to run one task per VM to guarantee fixed resources per run.

AutoGluon exposes parameters to control resources like CPUs, GPUs, and memory [^1]. However, the memory parameter is only a soft limit: in most cases it is respected, but it can occasionally be exceeded. In retrospect, I could have experimented with this, but given the VMs I had, it was simpler to enforce resource limits at the VM level.

To orchestrate this setup, I turned to Ray with the goal of launching one run per machine under fixed resource constraints. In practice, achieving this was not straightforward.

#### Setting up the Ray cluster

Setting up the Ray cluster was mostly straightforward, although I ended up doing it manually. I followed the [Ray manual setup guide](https://docs.ray.io/en/master/cluster/vms/user-guides/launching-clusters/on-premises.html#manually-set-up-a-ray-cluster), running `ray start` on each VM [^2]. I did try to use [Ray YAML cluster configuration](https://docs.ray.io/en/latest/cluster/vms/user-guides/launching-clusters/on-premises.html#start-ray-with-the-ray-cluster-launcher) to automate everything, but I ran into issues activating the right conda environment remotely. At the time, I wasn't very familiar with non-interactive conda environment activation. I suspect I could revisit this now given what I learned when [trying Ansible](../trying_ansible/#the-challenge-of-setting-up-conda). All the VMs shared an NFS mount, which simplified data access: I didn't have to copy datasets around, and results could be written to a common location accessible from every machine.

#### Ray resources are logical

I learned through this experience that Ray uses logical resources for scheduling, which means it doesn’t enforce CPU affinity or hard resource limits at the OS level. As a result, Ray doesn’t guarantee that one and only one task will be pinned to a single VM by default. However, because I requested the same number of CPUs per task as the number available on each VM, Ray’s scheduler behaved as I was hoping it would: one and only one task was placed on a single VM. This happens because Ray always places a task on a single node (VM) that has at least the number of requested CPUs available (in terms of logical resources). Since a VM is fully occupied by a task, Ray has to select another VM for the next task. You can read about this in [Ray documentation](https://docs.ray.io/en/latest/ray-core/scheduling/index.html#resources). There might be ways to assign tasks to specific nodes of the cluster with [Ray scheduling strategies](https://docs.ray.io/en/latest/ray-core/scheduling/index.html#scheduling-strategies) but I haven't played with this. If you want to learn more on Ray logical versus physical resources I give more details about it at the [end of the post](#appendix-more-on-ray-resources).

#### Isolating AutoGluon from the external Ray cluster

AutoGluon uses Ray under the hood, and this can lead to conflicts when you also manage your own Ray cluster. AutoGluon actually [discourages using Ray on top of it](https://auto.gluon.ai/dev/tutorials/tabular/tabular-faq.html#i-know-autogluon-uses-ray-underneath-what-s-the-best-practice-for-me).

In my case, AutoGluon’s internal Ray instance connected to the entire cluster and scheduled its own tasks across all nodes. My hypothesis is that AutoGluon internally launches smaller tasks (requesting fewer CPUs than the total available on a node), which allows Ray’s scheduler to distribute these tasks across multiple nodes depending on resource availability. This would require further testing to confirm[^3].

To isolate things, I first had to wrap the AutoGluon call in a dedicated Python script that was launched by the Ray task as a subprocess using `subprocess.run`:
```python
import subprocess
import ray

@ray.remote(num_cpus=8)  # 8 logical CPUs per task
def run_autogluon_task(challenge_name):
    subprocess.run(
        ["python", "autogluon_script.py", challenge_name],
    )

```

Then, inside the Python script that invokes AutoGluon (`autogluon_script.py`), I initialized a local Ray instance with `ray.init(address="local")` just before running AutoGluon, so that AutoGluon connects to the VM's local Ray cluster instead of connecting to the external one. After that, AutoGluon stayed confined to the resources of the VM it was launched on, which was the behavior I wanted.

This approach worked well overall, but I had another issue. Sometimes, Ray would kill a task because it exceeded its memory threshold [^4], but the subprocess running AutoGluon would keep running on the VM. Ray then assumed the machine was free and tried to schedule new tasks on it, which led to out-of-memory errors. Increasing Ray's memory thresholds helped in some cases, but the key fix was enabling [`RAY_kill_child_processes_on_worker_exit_with_raylet_subreaper=true`](https://docs.ray.io/en/latest/ray-core/user-spawn-processes.html), which ensured orphaned subprocesses were properly killed when Ray terminated a task.

#### Lessons learned

This kind of setup works best when the codebase is stable. When the code changes frequently, debugging errors through Ray can be cumbersome. It's usually easier to debug tasks outside of Ray first and only distribute them once the code is stable. Overall, the setup worked and allowed me to run a large number of AutoGluon tasks in parallel with fixed resource constraints. The key lessons were:

* Ray makes distribution easy but not foolproof. The logical vs physical resource model means you still need to think carefully about task placement and resource isolation if you want strict control. Other orchestration approaches could have achieved the same goal with different trade-offs in complexity, isolation, and monitoring.
* For a small number of machines, the manual Ray cluster setup was fine, though with better environment management I could probably automate this fully with YAML configs.
* One thing I didn't fully solve was Ray's behavior when tasks exceeded memory. Ray killed tasks for going over its memory threshold, even though the same AutoGluon runs succeeded without Ray. One solution would be to disable Ray task killing because of out of memory but this seemed a bit dangerous.

#### Appendix: More on Ray resources
Ray allows you to specify resources for each task using parameters like `num_cpus=2` and `memory=500 * 1024 * 1024` telling that you want to use 2 CPUs and 500MiB of RAM for each task. However, this does not prevent your task from using more than 2 physical CPUs and 500 MiB of RAM. It is actually your job to make sure the code run by your task does not use more physical resources than what you want to use. This is because Ray resources are logical and this can be very different from the physical resources of your machine. This is explained in the [Ray Resources documentation page](https://docs.ray.io/en/latest/ray-core/scheduling/resources.html): "Physical resources are resources that a machine physically has such as physical CPUs and GPUs and logical resources are virtual resources defined by a system. [...] Resource requirements of tasks or actors do NOT impose limits on actual physical resource usage." The definition of a logical resource depends on the system that defines it, so it differs between Ray and other software. For example, the number of logical CPUs (as defined by the operating system when counting hyperthreads as separate CPUs) is often twice the number of physical CPUs. This is different from Ray's own definition of logical resources. Also, there is no CPU affinity by default in Ray, so your task will not be pinned to specific cores.

Let's use an example to make this clear. When you use Ray, you start a Ray cluster (`ray.init`), which initializes the total number of resources that the cluster can use. By default this will use the number of CPUs your machine has as the total number of logical CPUs for the cluster, say 8 CPUs. Ray only monitors what happens within this cluster and schedules jobs according to the number of available CPUs and the number of CPUs required by each task (`num_cpus`). So let's say you have 10 tasks to run, each requiring 2 logical CPUs. Ray can start 4 tasks immediately, taking the 8 CPUs of its cluster. This does not mean that each task will only use 2 physical CPUs, nor that the physical CPUs used by the task are not used by another program external to Ray (or from another Ray cluster). If other programs were running on your machine, taking the 8 CPUs, Ray does not monitor this and does not know about these programs. Your Ray cluster does not know about other Ray clusters either. It just monitors what happens in its cluster, the total logical resources you allocated to it, and how many logical resources each task is using. When a task ends, your Ray cluster sees that there are 2 free logical CPUs, meaning that the total number of logical CPUs used by your tasks is 6 out of 8. Again, this doesn't mean there are actually 2 physical CPUs free on your machine: Ray only monitors the logical resources used by the tasks running within its cluster.

If setting `num_cpus` for a Ray task does not prevent your task from using more than `num_cpus` physical CPUs, why is it named like that? This is because `num_cpus` does have some connection to physical CPUs. First, by default, the total number of CPUs Ray assumes for its cluster is the number of logical CPUs on your machine (including hyperthreading). Then, when you explicitly pass `num_cpus`, Ray sets the `OMP_NUM_THREADS` environment variable to the same value, which ensures that libraries that read this variable will not use more than `num_cpus` physical CPUs. For similar reasons, Ray defines `num_gpus` and `memory` logical resources.

[^1]: AutoGluon has `num_cpus`, `num_gpus` and `memory_limit` parameters that you can check in the [documentation](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.fit.html#autogluon.tabular.TabularPredictor.fit).
[^2]: You need to make sure you have the exact same setup on all the VMs.
[^3]: From Ray documentation, ["the nodes are sorted to first favor those that already have tasks or actors scheduled (for locality), then to favor those that have low resource utilization (for load balancing)"](https://docs.ray.io/en/latest/ray-core/scheduling/index.html#default). This suggests that AutoGluon’s smaller tasks should stay on the same node but this locality rule might apply at the cluster level, not with respect to the parent task's node.
[^4]: To be completely exact, Ray kills a task when a memory threshold is exceeded at the **node (VM)** level, not the task level. Ray monitors memory usage on each node to prevent out-of-memory errors, but it does not monitor or enforce memory limits for individual tasks. This behavior is explained [here](https://docs.ray.io/en/latest/ray-core/scheduling/ray-oom-prevention.html).
