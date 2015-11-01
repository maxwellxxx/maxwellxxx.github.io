---
layout: post
title: 进程管理核心函数分析
description: do_fork copy_process等
category: manual
---

##一些说明

写这篇的主要原因是，纸质笔记对代码分析不方便，对于一些准备知识，本篇并不做诠释。如果有幸哪位读者能看到这篇文章，那建议请先去阅读相关x86CPU工作管理，包括EU，PU，SU各个单元的工作的大致原理。以及linux对PID的管理体系，包括了命名管理等先导知识。另外，在写这篇文章的时候，我自己有些还没有弄懂，有些知识，包括进程信号处理等知识。所以这里叙述的一些东西是不连贯且不成体系的，仅供我自己做笔记使用和给大家做一点参考。

这里的话主要是对两个函数进行一些分析，持续更新，有些我没有弄懂的东西先不做注释，等以后用到了，或者我自己弄懂了再回来补上，希望不要烂尾。


##do_fork()

废话不说先把代码贴上，今天是来不及做分析了。

	/*
	 *  Ok, this is the main fork-routine.
	 *
	 * It copies the process, and if successful kick-starts
	 * it and waits for it to finish using the VM if required.
	 */
	long do_fork(unsigned long clone_flags,
		      unsigned long stack_start,
		      unsigned long stack_size,
		      int __user *parent_tidptr,
		      int __user *child_tidptr)
	{
		struct task_struct *p;
		int trace = 0;
		long nr;

		/*
		 * Determine whether and which event to report to ptracer.  When
		 * called from kernel_thread or CLONE_UNTRACED is explicitly
		 * requested, no event is reported; otherwise, report if the event
		 * for the type of forking is enabled.
		 */
		if (!(clone_flags & CLONE_UNTRACED)) {
			if (clone_flags & CLONE_VFORK)
				trace = PTRACE_EVENT_VFORK;
			else if ((clone_flags & CSIGNAL) != SIGCHLD)
				trace = PTRACE_EVENT_CLONE;
			else
				trace = PTRACE_EVENT_FORK;

			if (likely(!ptrace_event_enabled(current, trace)))
				trace = 0;
		}

		p = copy_process(clone_flags, stack_start, stack_size,
				 child_tidptr, NULL, trace);
		/*
		 * Do this prior waking up the new thread - the thread pointer
		 * might get invalid after that point, if the thread exits quickly.
		 */
		if (!IS_ERR(p)) {
			struct completion vfork;
			struct pid *pid;

			trace_sched_process_fork(current, p);

			pid = get_task_pid(p, PIDTYPE_PID);
			nr = pid_vnr(pid);

			if (clone_flags & CLONE_PARENT_SETTID)
				put_user(nr, parent_tidptr);

			if (clone_flags & CLONE_VFORK) {
				p->vfork_done = &vfork;
				init_completion(&vfork);
				get_task_struct(p);
			}

			wake_up_new_task(p);

			/* forking complete and child started to run, tell ptracer */
			if (unlikely(trace))
				ptrace_event_pid(trace, pid);

			if (clone_flags & CLONE_VFORK) {
				if (!wait_for_vfork_done(p, &vfork))
					ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
			}

			put_pid(pid);
		} else {
			nr = PTR_ERR(p);
		}
		return nr;
	}


##copy_process

	/*
	 * This creates a new process as a copy of the old one,
	 * but does not actually start it yet.
	 *
	 * It copies the registers, and all the appropriate
	 * parts of the process environment (as per the clone
	 * flags). The actual kick-off is left to the caller.
	 */
	static struct task_struct *copy_process(unsigned long clone_flags,
						unsigned long stack_start,
						unsigned long stack_size,
						int __user *child_tidptr,
						struct pid *pid,
						int trace)
	{
		int retval;
		struct task_struct *p;

		if ((clone_flags & (CLONE_NEWNS|CLONE_FS)) == (CLONE_NEWNS|CLONE_FS))
			return ERR_PTR(-EINVAL);

		if ((clone_flags & (CLONE_NEWUSER|CLONE_FS)) == (CLONE_NEWUSER|CLONE_FS))
			return ERR_PTR(-EINVAL);

		/*
		 * Thread groups must share signals as well, and detached threads
		 * can only be started up within the thread group.
		 */
		if ((clone_flags & CLONE_THREAD) && !(clone_flags & CLONE_SIGHAND))
			return ERR_PTR(-EINVAL);

		/*
		 * Shared signal handlers imply shared VM. By way of the above,
		 * thread groups also imply shared VM. Blocking this case allows
		 * for various simplifications in other code.
		 */
		if ((clone_flags & CLONE_SIGHAND) && !(clone_flags & CLONE_VM))
			return ERR_PTR(-EINVAL);

		/*
		 * Siblings of global init remain as zombies on exit since they are
		 * not reaped by their parent (swapper). To solve this and to avoid
		 * multi-rooted process trees, prevent global and container-inits
		 * from creating siblings.
		 */
		if ((clone_flags & CLONE_PARENT) &&
					current->signal->flags & SIGNAL_UNKILLABLE)
			return ERR_PTR(-EINVAL);

		/*
		 * If the new process will be in a different pid or user namespace
		 * do not allow it to share a thread group or signal handlers or
		 * parent with the forking task.
		 */
		if (clone_flags & CLONE_SIGHAND) {
			if ((clone_flags & (CLONE_NEWUSER | CLONE_NEWPID)) ||
			    (task_active_pid_ns(current) !=
					current->nsproxy->pid_ns_for_children))
				return ERR_PTR(-EINVAL);
		}

		retval = security_task_create(clone_flags);
		if (retval)
			goto fork_out;

		retval = -ENOMEM;
		p = dup_task_struct(current);
		if (!p)
			goto fork_out;

		ftrace_graph_init_task(p);

		rt_mutex_init_task(p);

	#ifdef CONFIG_PROVE_LOCKING
		DEBUG_LOCKS_WARN_ON(!p->hardirqs_enabled);
		DEBUG_LOCKS_WARN_ON(!p->softirqs_enabled);
	#endif
		retval = -EAGAIN;
		if (atomic_read(&p->real_cred->user->processes) >=
				task_rlimit(p, RLIMIT_NPROC)) {
			if (p->real_cred->user != INIT_USER &&
			    !capable(CAP_SYS_RESOURCE) && !capable(CAP_SYS_ADMIN))
				goto bad_fork_free;
		}
		current->flags &= ~PF_NPROC_EXCEEDED;

		retval = copy_creds(p, clone_flags);
		if (retval < 0)
			goto bad_fork_free;

		/*
		 * If multiple threads are within copy_process(), then this check
		 * triggers too late. This doesn't hurt, the check is only there
		 * to stop root fork bombs.
		 */
		retval = -EAGAIN;
		if (nr_threads >= max_threads)
			goto bad_fork_cleanup_count;

		if (!try_module_get(task_thread_info(p)->exec_domain->module))
			goto bad_fork_cleanup_count;

		delayacct_tsk_init(p);	/* Must remain after dup_task_struct() */
		p->flags &= ~(PF_SUPERPRIV | PF_WQ_WORKER);
		p->flags |= PF_FORKNOEXEC;
		INIT_LIST_HEAD(&p->children);
		INIT_LIST_HEAD(&p->sibling);
		rcu_copy_process(p);
		p->vfork_done = NULL;
		spin_lock_init(&p->alloc_lock);

		init_sigpending(&p->pending);

		p->utime = p->stime = p->gtime = 0;
		p->utimescaled = p->stimescaled = 0;
	#ifndef CONFIG_VIRT_CPU_ACCOUNTING_NATIVE
		p->prev_cputime.utime = p->prev_cputime.stime = 0;
	#endif
	#ifdef CONFIG_VIRT_CPU_ACCOUNTING_GEN
		seqlock_init(&p->vtime_seqlock);
		p->vtime_snap = 0;
		p->vtime_snap_whence = VTIME_SLEEPING;
	#endif

	#if defined(SPLIT_RSS_COUNTING)
		memset(&p->rss_stat, 0, sizeof(p->rss_stat));
	#endif

		p->default_timer_slack_ns = current->timer_slack_ns;

		task_io_accounting_init(&p->ioac);
		acct_clear_integrals(p);

		posix_cpu_timers_init(p);

		p->start_time = ktime_get_ns();
		p->real_start_time = ktime_get_boot_ns();
		p->io_context = NULL;
		p->audit_context = NULL;
		if (clone_flags & CLONE_THREAD)
			threadgroup_change_begin(current);
		cgroup_fork(p);
	#ifdef CONFIG_NUMA
		p->mempolicy = mpol_dup(p->mempolicy);
		if (IS_ERR(p->mempolicy)) {
			retval = PTR_ERR(p->mempolicy);
			p->mempolicy = NULL;
			goto bad_fork_cleanup_threadgroup_lock;
		}
	#endif
	#ifdef CONFIG_CPUSETS
		p->cpuset_mem_spread_rotor = NUMA_NO_NODE;
		p->cpuset_slab_spread_rotor = NUMA_NO_NODE;
		seqcount_init(&p->mems_allowed_seq);
	#endif
	#ifdef CONFIG_TRACE_IRQFLAGS
		p->irq_events = 0;
		p->hardirqs_enabled = 0;
		p->hardirq_enable_ip = 0;
		p->hardirq_enable_event = 0;
		p->hardirq_disable_ip = _THIS_IP_;
		p->hardirq_disable_event = 0;
		p->softirqs_enabled = 1;
		p->softirq_enable_ip = _THIS_IP_;
		p->softirq_enable_event = 0;
		p->softirq_disable_ip = 0;
		p->softirq_disable_event = 0;
		p->hardirq_context = 0;
		p->softirq_context = 0;
	#endif
	#ifdef CONFIG_LOCKDEP
		p->lockdep_depth = 0; /* no locks held yet */
		p->curr_chain_key = 0;
		p->lockdep_recursion = 0;
	#endif

	#ifdef CONFIG_DEBUG_MUTEXES
		p->blocked_on = NULL; /* not blocked yet */
	#endif
	#ifdef CONFIG_BCACHE
		p->sequential_io	= 0;
		p->sequential_io_avg	= 0;
	#endif

		/* Perform scheduler related setup. Assign this task to a CPU. */
		retval = sched_fork(clone_flags, p);
		if (retval)
			goto bad_fork_cleanup_policy;

		retval = perf_event_init_task(p);
		if (retval)
			goto bad_fork_cleanup_policy;
		retval = audit_alloc(p);
		if (retval)
			goto bad_fork_cleanup_perf;
		/* copy all the process information */
		shm_init_task(p);
		retval = copy_semundo(clone_flags, p);
		if (retval)
			goto bad_fork_cleanup_audit;
		retval = copy_files(clone_flags, p);
		if (retval)
			goto bad_fork_cleanup_semundo;
		retval = copy_fs(clone_flags, p);
		if (retval)
			goto bad_fork_cleanup_files;
		retval = copy_sighand(clone_flags, p);
		if (retval)
			goto bad_fork_cleanup_fs;
		retval = copy_signal(clone_flags, p);
		if (retval)
			goto bad_fork_cleanup_sighand;
		retval = copy_mm(clone_flags, p);
		if (retval)
			goto bad_fork_cleanup_signal;
		retval = copy_namespaces(clone_flags, p);
		if (retval)
			goto bad_fork_cleanup_mm;
		retval = copy_io(clone_flags, p);
		if (retval)
			goto bad_fork_cleanup_namespaces;
		retval = copy_thread(clone_flags, stack_start, stack_size, p);
		if (retval)
			goto bad_fork_cleanup_io;

		if (pid != &init_struct_pid) {
			retval = -ENOMEM;
			pid = alloc_pid(p->nsproxy->pid_ns_for_children);
			if (!pid)
				goto bad_fork_cleanup_io;
		}

		p->set_child_tid = (clone_flags & CLONE_CHILD_SETTID) ? child_tidptr : NULL;
		/*
		 * Clear TID on mm_release()?
		 */
		p->clear_child_tid = (clone_flags & CLONE_CHILD_CLEARTID) ? child_tidptr : NULL;
	#ifdef CONFIG_BLOCK
		p->plug = NULL;
	#endif
	#ifdef CONFIG_FUTEX
		p->robust_list = NULL;
	#ifdef CONFIG_COMPAT
		p->compat_robust_list = NULL;
	#endif
		INIT_LIST_HEAD(&p->pi_state_list);
		p->pi_state_cache = NULL;
	#endif
		/*
		 * sigaltstack should be cleared when sharing the same VM
		 */
		if ((clone_flags & (CLONE_VM|CLONE_VFORK)) == CLONE_VM)
			p->sas_ss_sp = p->sas_ss_size = 0;

		/*
		 * Syscall tracing and stepping should be turned off in the
		 * child regardless of CLONE_PTRACE.
		 */
		user_disable_single_step(p);
		clear_tsk_thread_flag(p, TIF_SYSCALL_TRACE);
	#ifdef TIF_SYSCALL_EMU
		clear_tsk_thread_flag(p, TIF_SYSCALL_EMU);
	#endif
		clear_all_latency_tracing(p);

		/* ok, now we should be set up.. */
		p->pid = pid_nr(pid);
		if (clone_flags & CLONE_THREAD) {
			p->exit_signal = -1;
			p->group_leader = current->group_leader;
			p->tgid = current->tgid;
		} else {
			if (clone_flags & CLONE_PARENT)
				p->exit_signal = current->group_leader->exit_signal;
			else
				p->exit_signal = (clone_flags & CSIGNAL);
			p->group_leader = p;
			p->tgid = p->pid;
		}

		p->nr_dirtied = 0;
		p->nr_dirtied_pause = 128 >> (PAGE_SHIFT - 10);
		p->dirty_paused_when = 0;

		p->pdeath_signal = 0;
		INIT_LIST_HEAD(&p->thread_group);
		p->task_works = NULL;

		/*
		 * Make it visible to the rest of the system, but dont wake it up yet.
		 * Need tasklist lock for parent etc handling!
		 */
		write_lock_irq(&tasklist_lock);

		/* CLONE_PARENT re-uses the old parent */
		if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
			p->real_parent = current->real_parent;
			p->parent_exec_id = current->parent_exec_id;
		} else {
			p->real_parent = current;
			p->parent_exec_id = current->self_exec_id;
		}

		spin_lock(&current->sighand->siglock);

		/*
		 * Copy seccomp details explicitly here, in case they were changed
		 * before holding sighand lock.
		 */
		copy_seccomp(p);

		/*
		 * Process group and session signals need to be delivered to just the
		 * parent before the fork or both the parent and the child after the
		 * fork. Restart if a signal comes in before we add the new process to
		 * it's process group.
		 * A fatal signal pending means that current will exit, so the new
		 * thread can't slip out of an OOM kill (or normal SIGKILL).
		*/
		recalc_sigpending();
		if (signal_pending(current)) {
			spin_unlock(&current->sighand->siglock);
			write_unlock_irq(&tasklist_lock);
			retval = -ERESTARTNOINTR;
			goto bad_fork_free_pid;
		}

		if (likely(p->pid)) {
			ptrace_init_task(p, (clone_flags & CLONE_PTRACE) || trace);

			init_task_pid(p, PIDTYPE_PID, pid);
			if (thread_group_leader(p)) {
				init_task_pid(p, PIDTYPE_PGID, task_pgrp(current));
				init_task_pid(p, PIDTYPE_SID, task_session(current));

				if (is_child_reaper(pid)) {
					ns_of_pid(pid)->child_reaper = p;
					p->signal->flags |= SIGNAL_UNKILLABLE;
				}

				p->signal->leader_pid = pid;
				p->signal->tty = tty_kref_get(current->signal->tty);
				list_add_tail(&p->sibling, &p->real_parent->children);
				list_add_tail_rcu(&p->tasks, &init_task.tasks);
				attach_pid(p, PIDTYPE_PGID);
				attach_pid(p, PIDTYPE_SID);
				__this_cpu_inc(process_counts);
			} else {
				current->signal->nr_threads++;
				atomic_inc(&current->signal->live);
				atomic_inc(&current->signal->sigcnt);
				list_add_tail_rcu(&p->thread_group,
						  &p->group_leader->thread_group);
				list_add_tail_rcu(&p->thread_node,
						  &p->signal->thread_head);
			}
			attach_pid(p, PIDTYPE_PID);
			nr_threads++;
		}

		total_forks++;
		spin_unlock(&current->sighand->siglock);
		syscall_tracepoint_update(p);
		write_unlock_irq(&tasklist_lock);

		proc_fork_connector(p);
		cgroup_post_fork(p);
		if (clone_flags & CLONE_THREAD)
			threadgroup_change_end(current);
		perf_event_fork(p);

		trace_task_newtask(p, clone_flags);
		uprobe_copy_process(p, clone_flags);

		return p;

	bad_fork_free_pid:
		if (pid != &init_struct_pid)
			free_pid(pid);
	bad_fork_cleanup_io:
		if (p->io_context)
			exit_io_context(p);
	bad_fork_cleanup_namespaces:
		exit_task_namespaces(p);
	bad_fork_cleanup_mm:
		if (p->mm)
			mmput(p->mm);
	bad_fork_cleanup_signal:
		if (!(clone_flags & CLONE_THREAD))
			free_signal_struct(p->signal);
	bad_fork_cleanup_sighand:
		__cleanup_sighand(p->sighand);
	bad_fork_cleanup_fs:
		exit_fs(p); /* blocking */
	bad_fork_cleanup_files:
		exit_files(p); /* blocking */
	bad_fork_cleanup_semundo:
		exit_sem(p);
	bad_fork_cleanup_audit:
		audit_free(p);
	bad_fork_cleanup_perf:
		perf_event_free_task(p);
	bad_fork_cleanup_policy:
	#ifdef CONFIG_NUMA
		mpol_put(p->mempolicy);
	bad_fork_cleanup_threadgroup_lock:
	#endif
		if (clone_flags & CLONE_THREAD)
			threadgroup_change_end(current);
		delayacct_tsk_free(p);
		module_put(task_thread_info(p)->exec_domain->module);
	bad_fork_cleanup_count:
		atomic_dec(&p->cred->user->processes);
		exit_creds(p);
	bad_fork_free:
		free_task(p);
	fork_out:
		return ERR_PTR(retval);
	}

