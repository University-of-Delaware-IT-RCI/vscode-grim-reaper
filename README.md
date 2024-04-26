# vscode-grim-reaper

Microsoft has an IDE called Visual Studio Code — VSCode, for short.  Like any product with largescale corporate backing, it accomplishes an accelerated release schedule of a diverse set of features with cross-platform availability that no smaller outfit could manage.  The price tag is lucrative:  it's free.  As a result, it seems like every human being writing a scrap of code wants to use it, and every existing development tool/environment/paradigm wants to integrate itself into VSCode.

VSCode allows for this extension of its core features by means of...well, extensions.  That's literally what they're called.  One popular extension is *Remote-SSH*.  It allows the user to open an ssh tunnel to another computer in order to download and execute a number of processes on the remote side that will respond to directives sent by the local system:  basic client-server stuff across an encrypted ssh connection.  Browsing and editing files resident on the remote side on the local GUI is one major selling-point to many.  Opening an arbitrary number of remote shells to run programs from the comfort of VSCode also seems to be considered very useful (despite the fact that it duplicates a primary function of the ssh connection itself).

All of this seems pretty mundane, relatively secure (server runs as the user, communications are encrypted by ssh), and probably beneficial to development workflows.  But there is one major caveat:  //Remote-SSH server processes run forever//.  When the ssh tunnel breaks without explicitly shutting-down the remote software, the `vscode-server` process tree continues to run on the remote system.  The idea here is that the ssh connection *will* be restored, and the VSCode GUI will reconnect to that existing process tree.  This is wonderful, because any long-running processes the user may have started in one of those remote shells can keep running while the reconnection is pending — no loss of progress!

## Pattern mismatch

Those who designed the Remote-SSH plugin probably did not have HPC systems in mind.  The key tenets of the typical HPC system:

- access is gated through one or more login nodes
- useful resources on the cluster — GPUs, large memory — are **not** present on the login nodes
- HPC sysadmins tend to abhor long-running resource-intensive processes on login nodes

Quite often the multiple login nodes are load-balanced by means of DNS:  all login nodes' IPs appear together on a single DNS name.  When client systems successively resolve the name, the list of IPS returned is permuted.

On our Caviness cluster, with two login nodes (ignoring DNS caching for the moment) the user whose Remote-SSH connection dropped has a 50% chance of reconnecting to the same login node.  The other 50% of the time, that entire `vscode-server` process tree will never again be reconnected, yet it will keep running forever.  Or at least until a sysadmin kills it or the system is rebooted.

In conclusion, the Remote-SSH extension as currently implemented does not fit well with HPC systems.  We addressed this mismatch by writing a [proxy program](https://github.com/jtfrey/vscode-shell-proxy) that Remote-SSH executes in lieu of the user's remote shell.  The proxy funnels the connection through the Slurm job scheduler so that it runs on a compute node — possibly with GPU devices or large memory available to the `vscode-server`.  When the job ends, all of the processes are killed by the job scheduler.  No more orphaned VSCode process trees.

## Ongoing problems

Unfortunately, every cluster user who decides to give VSCode and Remote-SSH a try jumps right into the Microsoft documentation and does not consult our own documentation.  As a result, we routinely have users who have been directed to our documentation, who use the proxy; and we have users who continue to spawn orphaned `vscode-server` process trees on the login nodes.

As with any task a sysadmin finds himself performing repeatedly by hand, the identification and termination of `vscode-server` process trees on the login nodes eventually called for automation.  Thus, `vscode-grim-reaper` was written.

## Usage

We install the `vscode-grim-reaper` script in `/usr/local/bin` and assign ownership to root:root with mode 0744.

A systemd timer is provided in this repository that starts the one-shot systemd service of the same name at the top of every hour.  It assumes the program has been installed as `/usr/local/bin/vscode-grim-reaper` and directs verbose logging to `/var/log/vscode-grim-reaper.log`.

A typical verbose log file will contain lines like this:

```
[2024-04-25T23:00:00] Killed 1 process(es) under pid 76856 (uid XXXX)
[2024-04-25T23:00:00]    76856        1    /home/XXXX/.vscode-server/extens
[2024-04-25T23:00:00] Killed 10 process(es) under pid 75950 (uid XXXX)
[2024-04-25T23:00:00]    75950    75897    /home/XXXX/.vscode-server/code-e
[2024-04-25T23:00:00]    76086    75950      |-  sh /home/XXXX/.vscode-server/cli
[2024-04-25T23:00:00]    76092    76086      |-    |-  /home/XXXX/.vscode-server/cli/se
[2024-04-25T23:00:00]    76156    76092      |-    |-    |-  /home/XXXX/.vscode-server/cli/se
[2024-04-25T23:00:00]    76283    76156      |-    |-    |-    |-  /home/XXXX/.vscode-server/extens
[2024-04-25T23:00:00]    76171    76092      |-    |-    |-  /home/XXXX/.vscode-server/cli/se
[2024-04-25T23:00:00]    76425    76171      |-    |-    |-    |-  /bin/bash --init-file /home/XXXX
[2024-04-25T23:00:00]    76210    76171      |-    |-    |-    |-  /bin/bash --init-file /home/XXXX
[2024-04-25T23:00:00]    78946    76171      |-    |-    |-    |-  /bin/bash --init-file /home/XXXX
[2024-04-25T23:00:00]    76148    76092      |-    |-    |-  /home/XXXX/.vscode-server/cli/se
```

Without the `-v` option, the logging would omit the process tree listing:

```
[2024-04-25T23:00:00] Killed 1 process(es) under pid 76856 (uid XXXX)
[2024-04-25T23:00:00] Killed 10 process(es) under pid 75950 (uid XXXX)
```
