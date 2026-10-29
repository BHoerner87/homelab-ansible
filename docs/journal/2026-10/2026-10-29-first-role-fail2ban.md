Today if my schedule stays clear I will be running my first Ansible playbooks to do whatever initializing needs done (I'm still wrapping my head around the fact it is agentless) and installing fail2ban anywhere that isn't running CrowdSec locally.

This was born out of a false positive that caught my attention in my Wazuh dashboard—basically every day, it would display 2 failed authentication attempts. After pursuing this, and learning more about event parsing in Wazuh, I determined that in fact, Lynis (a CLI tool for auditing local host security which is able to peer into Docker containers in a way that Wazuh cannot) was generating a recommendation every night pointing out that I *should* (in its opinion, as it were) install Fail2Ban to protect against failed login attempts. Because it was writing language related to failed authentication attempts, Wazuh pattern-matching was picking up on that as being actual, reported authentication failures.

I put a rule in place to suppress that detection in Wazuh, but I am proceeding with its recommendation regarding Fail2Ban because I want to get my hands on that piece of software in order to better understand it, and take advantage of it if I can. Even though CrowdSec is protecting OPNsense (my router) and CozyLab (my non-VM server), so hypothetically there shouldn't be an easy mode of ingress into my network...but if something should happen to appear inside my network despite this protection, Fail2Ban would help on hosts that don't have CrowdSec as a mechanism to recognize failed login attempts and bounce the hosts making them.

At least, that's my best understanding of this right now. I'm sure it will be refined and corrected as I learn more.

--

Terms, including ones I may have forgotten since the last time I worked on this:
- role
  - A folder with a conventional layout. Ansible looks for main.yml in each subfolder automatically, so one never points it at those files directly.
- ```tasks/main.yml```
  - The ordered list of steps. Each task calls a **module** and modules describe the desired *end state* rather than commands to run. That's why running the playbook twice changes nothing the second time (idempotence).
- idempotence
  - The property of certain operations in mathematics and computer science that can be applied multiple times without changing the result beyond the initial application
  - e.g. (f(f(x)) = f(x))

- ```tasks/``` - holds what a role does. Every role needs one.
- ```templates/``` - exists because Fail2Ban writes a config file (jail.local.j2).
- ```handlers/``` - exists because changing that config means restarting a service

*Inside of tasks/main.yml*
- ```apt``` with ```state: present``` means "make sure it's installed". If already present, it just reports "ok".
- ```update_cache:``` true runs the equivalent of apt update first. Add cache_valid_time: 3600 so it skips the refresh if the cache was updated in the last hour.
- ```template``` renders the .j2 file from templates/ (no path needed) and writes it to dest
  - .j2 file: a Jinja2 template file used to generate dynamic text files, configuration files, or scripts...a popular templating language for Python
- ```mode: "0644"``` is quoted on purpose. Unqouted, YAML can read a leading-zero number the wrong way and one gets odd permissions.
- ```notify: Restart fail2ban``` means "if this task changed the file, queue that handler." If the config is unchanged, nothing gets restarted.
- ```service``` with ```started``` and ```enabled``` means it's running **now** and **starts on boot**.

```handlers/main.yml``` holds tasks that run only when notified. They run once, at the end of the play, however many tasks notified them. The ```name``` has to match the ```notify``` string exactly, character for character. A typo there fails without any obvious error.

```templates/jail.local.j2``` is a Jinja2 template. Right now it contains no variables, so it is effectively a static file.

**Jail values**
One writes to ```jail.local``` rather than ```jail.conf``` beacuse package upgrades overwrite ```jail.conf```, while ```.local``` files are yours and override it.
- ```[DEFAULT]``` applies to every jail unless a jail overrides it.
- ```ignoreip``` lists addresses that are never banned.
- ```maxretry = 5```, ```findtime = 10m```, ```bantime = 1h```: 5 failures within any 10 minute window gets the source banned for an hour.
- ```backend = systemd``` reads the journal instead of log files. This matters because recent Debian doesn't write ```/var/log/auth.log``` by default. It requires the ```python3-systemd``` package, so it should be installed alongside fail2ban to be safe.
  - (I am only glancingly familiar with how systemd works so I'm making the following note)\n The **systemd-journald** is a built-in logging system in Linux that collects and stores structured data from the kernel, initrd, services, and user processes. Managed by the systemd-journald service, it replaces or works alongside traditional plain-text logging daemons like syslog.
- ```[sshd] enabled = true``` turns on the SSH jail. Its filter (the regex that detects failures), port, and ban action are inherited from the stock jail.conf, so one doesn't need to spell them out.
