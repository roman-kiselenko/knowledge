---
title: Dealing with process termination in Linux (with Rust examples)
source: https://iximiuz.com/en/posts/dealing-with-processes-termination-in-Linux/#awaiting-a-grandchild-process-termination
clipped: 2024-05-12
published: 2019-10-20
category: development
tags:
  - os-process
  - os-signal
  - os-subreaper
  - rust
  - c
read: false
---

A looooong one this time, so straight to the point! First, we'll discuss the Linux processes trivia and then review the following scenarios, as usually, with some code examples:

-   awaiting a child process termination;
-   awaiting a grandchild process termination;
-   catching the parent process termination.

Level up your server-side game — join 6,500 engineers getting insightful learning materials straight to their inbox.

## Linux processes basics

First and foremost, every process in Linux has an ID, so-called *PID*. Whenever [we create a new process](https://iximiuz.com/en/posts/how-to-on-processes/) via [`fork()`](https://linux.die.net/man/2/fork) \[or [clone()](https://linux.die.net/man/2/clone)\] system call, a next spare *PID* is assigned to it by the kernel. The process that makes the `fork()` call becomes *a parent* of the newly created process and its PID becomes a *parent process id, i.e. PPID* of the *child* process. This procedure is respected for all the processes in the system, starting from the very first one (i.e. *init* process with *PID 1*). Thus, processes in Linux form a tree-like structure where at any given time, a process always has a single parent process (ancestor) and can have from *0* to *N* child processes (descendants):

![[Raw/Media/Resources/22286a781423e9bcb5135699e9d7a9b1_MD5.png]]

On the screenshot above there is an output of `ps --ppid 2 -p 2 --deselect -fo pid,ppid,command` showing the current state of the process tree on a Centos 7 machine. The *init* process, as it usually happens in modern Linux distributions, is implemented by `systemd`. Also, I intentionally used `--ppid 2 -p 2 --deselect` to prevent spoiling the results with the numerous kernel threads spawned by the process with *PID 2*. Notice, the branches started by *sshd* daemon with *PID 3164*. The daemon itself has been started by the *init* process with *PID 1*. In turn, it spawned two subprocesses *6746* and *6957* for the two ongoing ssh sessions to this server. And these sessions in their turns spawned *bash* shells with PIDs *6750* and *6961*.

## Awaiting a child process termination

First of all, why do we need to wait? Well, sometimes the parent process is just interested in the child termination event. The famous example is a worker pool pattern utilized by *Nginx*, *PHP-FPM*, *uWSGI*, and others. The main process has to maintain a fixed-size pool of sub-processes, while the processes in the pool will be doing the actual payload. Occasionally, workers die, and the main process has to spawn more processes in response to keep the pool size unaltered. For this and similar situations, the Linux kernel provides a family of [`wait()`](https://linux.die.net/man/2/wait) system calls to await the child process termination. However, due to the way, the reporting of the child exit status back to the parent is implemented, even careless parents have to explicitly await their children terminations. Otherwise, a [*zombie*](https://en.wikipedia.org/wiki/Zombie_process) apocalypse begins.

When a process forks itself, the execution of the created child process happens independently (we will leave [process groups](https://en.wikipedia.org/wiki/Process_group) and [job control](https://en.wikipedia.org/wiki/Job_control_(Unix)) out of the scope of this article) and concurrently with the parent's execution. Even though the new process is still parented to the original process, by default its execution is not synchronized with it. Since the Linux kernel has to always provide a way to report the child exit status (and some usage stats, like spent CPU cycles, number of page faults, etc) back to its parent, the kernel needs to keep a tiny bit of information about every process in system memory even after its exit. This information gets cleaned up only when it has been explicitly read by the corresponding parent process. From the process termination and until its status has been read by the parent via `wait()` call, the process stays in *defunct* or *zombie* state. Zombie processes keep system resources, such as memory or process ids, consumed and that eventually leads to the denial of service.

Hopefully, I convinced you that the process' awaiting is of paramount importance. Now, let’s get our hands dirty with some code!

*Disclaimer on the programming language choice.*

The single most important thing while learning a platform (Linux kernel in this case) is to ensure that the observed behavior is not spoiled by any intermediary components. A lot of modern programming languages (the most prominent example would be *Go*) possess extensive *runtimes* with significant auxiliary logic written on top of the OS core functionality [to provide a unified programming experience for different platforms](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/) (Linux, BSD, macOS, Windows, etc). And for our Linux processes journey, the language is such an intermediary component. Thus, the most reasonable choice would be obviously *C*. There is little to no extra logic (i.e. *language runtime*) in between *C* standard library and the Linux system calls. Thus, the language doesn't spoil the original system's behavior. However, [*Rust* is claimed to be a *new C*](https://hub.packtpub.com/rust-is-the-future-of-systems-programming-c-is-the-new-assembly-intel-principal-engineer-josh-triplett/). So, let's check it out! But we have to be careful because *Rust* does have [higher-level abstractions to deal with processes](https://doc.rust-lang.org/std/process/) as well. Instead, we'll be using raw bindings to [*libc*](https://github.com/rust-lang/libc) \[luckily provided by the core team, but unfortunately *unsafe*\], and a \[luckily\] shallow \[but unfortunately\] third-party wrapper around Unix-specific part of *libc* called [*nix*](https://crates.io/crates/nix), that makes the calls great safe again. The full version of the source code in the article can be found [on my GitHub](https://github.com/iximiuz/reapme).

```rust
use std::process::exit;
use std::thread::sleep;
use std::time::Duration;

use nix::sys::wait::waitpid;
use nix::unistd::{fork, getpid, getppid, ForkResult};

fn main() {
    println!("[main] Hi there! My PID is {}.", getpid());

    let child_pid = match fork() {
        Ok(ForkResult::Child) => {
            
            
            
            println!(
                "[child] I'm alive! My PID is {} and PPID is {}.",
                getpid(),
                getppid()
            );

            println!("[child] I'm gonna sleep for a while and then just exit...");
            sleep(Duration::from_secs(2));
            exit(0);
        }

        Ok(ForkResult::Parent { child, .. }) => {
            println!("[main] I forked a child with PID {}.", child);
            child
        }

        Err(err) => {
            panic!("[main] fork() failed: {}", err);
        }
    };

    println!("[main] I'll be waiting for the child termination...");
    match waitpid(child_pid, None) {
        Ok(status) => println!("[main] Child exited with status {:?}", status),
        Err(err) => panic!("[main] waitpid() failed: {}", err),
    }
    println!("[main] Bye Bye!");
}
```

The idea behind the snipped above is hell simple. The execution starts from the *main process* which reports its *PID* and immediately creates *a child process* by forking itself. The *child process*, in turn, reports its *PID* and *PPID*, but then falls asleep before the termination. However, the execution of the parent process continues to the next line after the `fork()` call regardless of the child's behavior. Thus, the *main process* uses `waitpid()` call to pause itself until the child's exit status will be available:

```
[main] Hi there! My PID is 7231.
[main] I forked a child with PID 7264.
[child] I'm alive! My PID is 7264 and PPID is 7231.
[main] I'll be waiting for the child termination...
[child] I'm gonna sleep for a while and then just exit...
[main] Child exited with status Exited(Pid(7264), 0)
[main] Bye Bye!
```

By default, `waitpid()` call blocks the caller, but a non-blocking behavior is achievable via specifying `WNOHANG` flag in the second argument.

The `wait()` system call is much less flexible. It doesn't allow to specify the PID of the awaited process and always suspends the execution of the caller process. The first terminated child (assuming there are some) will resume the execution of the parent process. There are obsolete alternatives like [`wait3()` and `wait4()`](https://linux.die.net/man/2/wait4) trying to enrich `wait()` behavior, but nowadays `waitpid()` is the preferred method.

Last but not least, what if we want to know when the child process terminates but at the same time we don't want to block the parent process? The parent process may have its own things to be done in parallel with awaiting the child. One solution could be to use a *busy loop*, something like:

```rust
fn main() {
    println!("[main] Hi there! My PID is {}.", getpid());

    

    println!("[main] I'll be doing my own stuff while waiting for the child termination...");
    loop {
        match waitpid(child_pid, Some(WaitPidFlag::WNOHANG)) {
            Ok(WaitStatus::StillAlive) => {
                println!("[main] Child is still alive, do my own stuff while waiting.");
                
                sleep(Duration::from_millis(500));
            }

            Ok(status) => {
                println!("[main] Child exited with status {:?}.", status);
                break;
            }

            Err(err) => panic!("[main] waitpid() failed: {}", err),
        }
    }
    println!("[main] Bye Bye!");
}
```

While legit, this is not the most efficient way to solve the problem. A much more flexible solution is based on [*Linux signals*](https://linux.die.net/man/7/signal) - an asynchronous IPC mechanism. When a child process gets terminated the kernel sends a *SIGCHLD* signal to the parent process. By default, there is no action assigned for this kind of signals, so parents just ignore them, but we can set up a custom *SIGCHLD* handler:

***UPDATED**: thanks to redditor [u/jschievink](https://www.reddit.com/user/jschievink/) for pointing out that Rust standard library functions aren't [async-signal-safe](https://www.man7.org/linux/man-pages/man7/signal-safety.7.html). This is a nice pitfall people should be aware of when choosing not C for their system programming projects. Unix signals are ancients asynchronous creatures interrupting your program at any arbitrarily bad time. Working properly with signals [is very tricky](https://vorner.github.io/2018/06/28/signal-hook.html) even in pure C since even primitive synchronization and memory allocation is not safe in signal handler code. Luckily, our examples are rather simplistic and libc `write()` and `waitpid()` are in the async-signal-safe list.*

*Original and incorrect version of the code.*

```rust
use std::process::exit;
use std::thread::sleep;
use std::time::Duration;

use nix::sys::signal::{sigaction, SaFlags, SigAction, SigHandler, SigSet, SIGCHLD};
use nix::sys::wait::waitpid;
use nix::unistd::{fork, getpid, getppid, ForkResult, Pid};

extern "C" fn handle_sigchld(_: libc::c_int) {
    println!("[main] What a surprise! Got SIGCHLD!");
    match waitpid(Pid::from_raw(-1), None) {
        Ok(status) => println!("[main] Child exited with status {:?}", status),
        Err(err) => panic!("[main] waitpid() failed: {}", err),
    }
    println!("[main] Bye Bye!");
    exit(0);
}

fn main() {
    println!("[main] Hi there! My PID is {}.", getpid());

    match fork() {
        Ok(ForkResult::Child) => {
            
            
            
            println!(
                "[child] I'm alive! My PID is {} and PPID is {}.",
                getpid(),
                getppid()
            );

            println!("[child] I'm gonna sleep for a while and then just exit...");
            sleep(Duration::from_secs(2));
            exit(0);
        }

        Ok(ForkResult::Parent { child, .. }) => {
            println!("[main] I forked a child with PID {}.", child);
        }

        Err(err) => {
            panic!("[main] fork() failed: {}", err);
        }
    };

    let sig_action = SigAction::new(
        SigHandler::Handler(handle_sigchld),
        SaFlags::empty(),
        SigSet::empty(),
    );

    if let Err(err) = unsafe { sigaction(SIGCHLD, &sig_action) } {
        panic!("[main] sigaction() failed: {}", err);
    };

    println!("[main] I'll be doing my own stuff...");
    loop {
        println!("[main] Do my own stuff.");
        
        sleep(Duration::from_millis(500));
    }
}
```

```rust
use libc::{_exit, STDOUT_FILENO, write};


extern "C" fn handle_sigchld(_: libc::c_int) {
    print_signal_safe("[main] What a surprise! Got SIGCHLD!\n");
    match waitpid(Pid::from_raw(-1), None) {
        Ok(_) => {
            print_signal_safe("[main] Child exited.\n");
            print_signal_safe("[main] Bye Bye!\n");
            exit_signal_safe(0);
        }
        Err(_) => {
            print_signal_safe("[main] waitpid() failed.\n");
            exit_signal_safe(1);
        }
    }
}

fn main() {
    println!("[main] Hi there! My PID is {}.", getpid());

    

    let sig_action = SigAction::new(
        SigHandler::Handler(handle_sigchld),
        SaFlags::empty(),
        SigSet::empty(),
    );
    if let Err(err) = unsafe { sigaction(SIGCHLD, &sig_action) } {
        panic!("[main] sigaction() failed: {}", err);
    };

    println!("[main] I'll be doing my own stuff...");
    loop {
        println!("[main] Do my own stuff.");
        
        sleep(Duration::from_millis(500));
    }
}

fn print_signal_safe(s: &str) {
    unsafe {
        write(STDOUT_FILENO, s.as_ptr() as (* const c_void), s.len());
    }
}

fn exit_signal_safe(status: i32) {
    unsafe {
        _exit(status);
    }
}
```

The program from above produces the following output:

```
[main] Hi there! My PID is 7134.
[main] I forked a child with PID 7136.
[child] I'm alive! My PID is 7136 and PPID is 7134.
[child] I'm gonna sleep for a while and then just exit...
[main] I'll be doing my own stuff...
[main] Do my own stuff.
[main] Do my own stuff.
[main] Do my own stuff.
[main] Do my own stuff.
[main] What a surprise! Got SIGCHLD!
[main] Child exited.
[main] Bye Bye!
```

## Awaiting a grandchild process termination

Ok, here is a tricky one. We already know that every process has to wait for its children's termination to prevent the zombie apocalypse. But what happens if we forked a child process, which in turn forked another process and then suddenly died. Huh, seems like we have a grandchild now. But it can easily be, that we haven't even heard about our grandchild. So, what is the grandchild's parent id after its actual parent death? And who is going to await this apparently abandoned process?

![[Raw/Media/Resources/0431bdd4739514219b5abaaa5b07f19e_MD5.png]]

Right, the *PID 1*! Linux doesn't allow orphan processes. Historically, in situations similar to the described above, the grandchild processes would be reparented to the *init* process, the ancestor of all the processes. Although it’s not necessarily the case starting from the Linux 3.4 kernel, we will get back to it in a minute.

Now, let's make the problem even more interesting. What if our main process wants to catch the termination event of the grandchild? One real-world example of such a situation would be a supervisor-alike program trying to start and keep track of another program that daemonizes itself [using the double fork trick](https://stackoverflow.com/questions/881388/what-is-the-reason-for-performing-a-double-fork-when-creating-a-daemon).

The double fork technique is an ancient way of daemonization based on the reparenting to *PID 1* behavior. With a rise of [supervisory programs](https://en.wikipedia.org/wiki/Supervisory_program) like [*supervisor*](https://github.com/Supervisor/supervisor) or [*systemd*](https://en.wikipedia.org/wiki/Systemd), it's getting less and less popular, however, the supervisors need to deal with the legacy software as well. Luckily, starting from the kernel version 3.4 there is a way to solve the problem!

To prevent the reparenting of the orphan process to the *init* process, we need to mark one of its more close ancestors (for instance, our main process) as a [***subreaper***](https://linux.die.net/man/2/prctl#PR_SET_CHILD_SUBREAPER)!

![[Raw/Media/Resources/aaabbb0bbcb4267627602c4c746a9ace_MD5.png]]

The implementation of this trick is as simple as a single [`prctl()`](https://linux.die.net/man/2/prctl) invocation. This system call is an almighty tool when it comes to process behavior tuning. Let's try to make a workable example:

```rust
use std::process::{exit};
use std::thread::{sleep};
use std::time::Duration;

use libc::{prctl, PR_SET_CHILD_SUBREAPER};
use nix::sys::wait::waitpid;
use nix::unistd::{fork, ForkResult, getpid, getppid, Pid};

fn main() {
    println!("[main] Hi there! My PID is {}.", getpid());
    println!("[main] Making myself a child subreaper.");
    unsafe {
        prctl(PR_SET_CHILD_SUBREAPER, 1, 0, 0, 0);
    }

    match fork() {
        Ok(ForkResult::Child) => {
            
            
            
            println!("[child 1] I'm alive! My PID is {} and PPID is {}.", getpid(), getppid());

            match fork() {
                Ok(ForkResult::Child) => {
                    
                    
                    
                    for _ in 0..6 {
                        println!("[child 2] I'm alive! My PID is {} and PPID is {}.", getpid(), getppid());
                        sleep(Duration::from_millis(500));
                    }
                    println!("[child 2] Bye Bye");
                    exit(0);
                }

                Ok(ForkResult::Parent { child, .. }) => {
                    println!("[child 1] I forked a grandchild with PID {}.", child);
                }

                Err(err) => panic!("[child 1] fork() failed: {}", err),
            };

            println!("[child 1] I'm gonna sleep for a while and then just exit...");
            sleep(Duration::from_millis(1500));
            exit(0);
        }

        Ok(ForkResult::Parent { child, .. }) => {
            println!("[main] I forked a child with PID {}.", child);
        }

        Err(err) => panic!("[main] fork() failed: {}", err),
    };

    println!("[main] I'll be waiting for the child termination...");
    match waitpid(Pid::from_raw(-1), None) {
        Ok(status) => println!("[main] Child exited with status {:?}", status),
        Err(err) => println!("[main] waitpid() failed: {}", err),
    }

    println!("[main] I'll be waiting for the grandchild termination as well...");
    sleep(Duration::from_millis(500));  
    match waitpid(Pid::from_raw(-1), None) {
        Ok(status) => println!("[main] Grandchild exited with status {:?}", status),
        Err(err) => println!("[main] waitpid() failed: {}", err),
    }
    println!("[main] Bye Bye!");
}
```

The idea is pretty simple again. The main process forks a child which, in turn, forks its own child, i.e. a grandchild of the main process. The grandchild continuously reports its parent id. In a while, the child process exits and since the main process marked itself a subreaper, the grandchild gets reparented to the main process instead of the *init* process. The following output produced by the program proves it:

```
[main] Hi there! My PID is 6851.
[main] Making myself a child subreaper.
[main] I forked a child with PID 6893.
[main] I'll be waiting for the child termination...
[child 1] I'm alive! My PID is 6893 and PPID is 6851.
[child 1] I forked a grandchild with PID 6894.
[child 2] I'm alive! My PID is 6894 and PPID is 6893.
[child 1] I'm gonna sleep for a while and then just exit...
[child 2] I'm alive! My PID is 6894 and PPID is 6893.
[child 2] I'm alive! My PID is 6894 and PPID is 6893.
[child 2] I'm alive! My PID is 6894 and PPID is 6893.
[main] Child exited with status Exited(Pid(6893), 0)
[main] I'll be waiting for the grandchild termination as well...
[child 2] I'm alive! My PID is 6894 and PPID is 6851.
[child 2] I'm alive! My PID is 6894 and PPID is 6851.
[child 2] Bye Bye
[main] Grandchild exited with status Exited(Pid(6894), 0)
[main] Bye Bye!
```

## Catching the parent process termination

Well, we already know, that a parent process gets notified by the kernel on its children's status change via *SIGCHLD* signal, even though the default action for this signal is just to ignore it. But can a child process catch its parent termination? Apparently yes, but some extra effort is required. Calling [`prctl()`](https://linux.die.net/man/2/prctl) with the *PR\_SET\_PDEATHSIG* flag in the child process allows us to register a signal that will be sent to the child process on its parent death. We can use an arbitrary signal number as a second argument to the `prctl()` call. For instance, invoking `prctl(PR_SET_PDEATHSIG, SIGKILL)` will lead to the child process termination as well. We also can use a harmless signal, like *SIGUSR1* with a custom handler.

Now, let's try to put together all the things we've learned today. First, we will make a program catching its parent termination:

```rust
use std::ffi::{c_void};
use std::thread;
use std::time::Duration;

use libc::{prctl, PR_SET_PDEATHSIG, STDOUT_FILENO, write};
use nix::sys::signal::{sigaction, SaFlags, SigAction, SigHandler, SigSet, SIGUSR1};
use nix::unistd::{getpid, getppid, Pid};

extern "C" fn handle_sigusr1(_: libc::c_int) {
    print_signal_safe("[sleepy] Parent died!\n");
}

fn main() {
    println!("[sleepy] Hi there! My pid is {}", getpid());
    println!("[sleepy] Binding SIGUSR1 to the parent termination event");
    unsafe {
        prctl(PR_SET_PDEATHSIG, SIGUSR1);
    }

    let sig_action = SigAction::new(
        SigHandler::Handler(handle_sigusr1),
        SaFlags::empty(),
        SigSet::empty(),
    );

    if let Err(err) = unsafe { sigaction(SIGUSR1, &sig_action) } {
        println!("[sleepy] sigaction() failed: {}", err);
    };

    loop {
        let ppid = getppid();
        println!("[sleepy] My parent is {}. Zzzz...", ppid);
        thread::sleep(Duration::from_millis(500));
        if ppid == Pid::from_raw(1) {
            println!("[sleepy] My parent is init process. I don't like it much...");
            break;
        }
    }

    println!("[sleepy] Bye Bye!");
}

fn print_signal_safe(s: &str) {
    unsafe {
        write(STDOUT_FILENO, s.as_ptr() as (* const c_void), s.len());
    }
}
```

And now we will just `exec()` this program in a grandchild process while making our main process a child subreaper:

```rust
use std::ffi::CString;
use std::process::{exit};
use std::thread::{sleep};
use std::time::Duration;

use libc::{prctl, PR_SET_CHILD_SUBREAPER};
use nix::sys::wait::waitpid;
use nix::unistd::{execv, fork, ForkResult, getpid, getppid, Pid};

fn main() {
    println!("[main] Hi there! My pid is {}", getpid());
    println!("[main] Making myself a child subreaper.");
    unsafe {
        prctl(PR_SET_CHILD_SUBREAPER, 1, 0, 0, 0);
    }

    match fork() {
        Ok(ForkResult::Parent { child, .. }) => {
            println!("[main] Forked new child with pid {}", child);
        }
        Ok(ForkResult::Child) => {
            
            
            
            println!("[child 1] I'm alive! My PID is {} and PPID is {}.", getpid(), getppid());

            match fork() {
                Ok(ForkResult::Child) => {
                    
                    
                    
                    println!("[child 2] I'm alive! My PID is {} and PPID is {}.", getpid(), getppid());
                    println!("[child 2] Exec-ing...");
                    exec_or_die("target/debug/sleepy");
                }

                Ok(ForkResult::Parent { child, .. }) => {
                    println!("[child 1] I forked a child with PID {}.", child);
                }
                Err(err) => panic!("[child 1] fork failed: {}", err),
            }

            println!("[child 1] I'm gonna sleep for a while and then just exit...");
            sleep(Duration::from_millis(1500));
            exit(0);
        }
        Err(err) => panic!("main: fork failed: {}", err),
    };

    println!("[main] I'll be waiting for the child termination...");
    match waitpid(Pid::from_raw(-1), None) {
        Ok(status) => println!("[main] Child exited with status {:?}", status),
        Err(err) => println!("[main] waitpid() failed: {}", err),
    }

    println!("[main] I'll not be waiting for the grandchild though.");
    sleep(Duration::from_millis(1000));
    println!("[main] Bye Bye!");
}

fn exec_or_die(name: &str) {
    let name_cstr = CString::new(name).unwrap();
    match execv(&name_cstr, &vec![name_cstr.clone()]) {
        Ok(_) => unreachable!("execv() succeed! Wait, what?!"),
        Err(err) => unreachable!("execv() failed: {}", err),
    }
}
```

If we build the `sleepy.rs` and then launch the `combined.rs`, they will produce the following combined output:

```
[main] Hi there! My pid is 7241
[main] Making myself a child subreaper.
[main] Forked new child with pid 7243
[main] I'll be waiting for the child termination...
[child 1] I'm alive! My PID is 7243 and PPID is 7241.
[child 1] I forked a child with PID 7244.
[child 1] I'm gonna sleep for a while and then just exit...
[child 2] I'm alive! My PID is 7244 and PPID is 7243.
[child 2] Exec-ing...
[sleepy] Hi there! My pid is 7244
[sleepy] Binding SIGUSR1 to the parent termination event
[sleepy] My parent is 7243. Zzzz...
[sleepy] My parent is 7243. Zzzz...
[sleepy] My parent is 7243. Zzzz...
[sleepy] Parent died!
[main] Child exited with status Exited(Pid(7243), 0)
[main] I'll not be waiting for the grandchild though.
[sleepy] My parent is 7241. Zzzz...
[sleepy] My parent is 7241. Zzzz...
[main] Bye Bye!
[sleepy] Parent died!
[sleepy] My parent is 1. Zzzz...
[sleepy] My parent is init process. I don't like it much...
[sleepy] Bye Bye!
```

## Instead of conclusion

There is usually little motivation to study such basic topics as Linux processes. However, only having sufficient fundamental knowledge we can build reliable higher-level software. I realized it once again on my journey to [implementing a container runtime shim](https://iximiuz.com/en/posts/implementing-container-runtime-shim/) as a part of the educational [container manager](https://iximiuz.com/en/posts/conman-the-container-manager-inception/) project. There's plenty of books about Linux/Unix programming, but my favorite book so far is [Advanced Programming in the Unix Environment](https://en.wikipedia.org/wiki/Advanced_Programming_in_the_Unix_Environment). It's a comprehensive but still easy digestible resource with very reasonably ordered chapters. So, if you are interested in the topic, I strongly advise to check it out.

The full version of the source code in the article can be found [on my GitHub](https://github.com/iximiuz/reapme).

Level up your server-side game — join 6,500 engineers getting insightful learning materials straight to their inbox: