# Ansible Roles – Extended

## The Scenario

Your team manages a growing fleet of web servers. The playbooks that started life as a few tasks have grown into long files that mix web-server setup, user administration, host-name configuration, and banner deployment. Every time someone updates one area, they risk breaking another. New team members struggle to understand where things are defined.

**Roles** are Ansible's answer: a standard directory layout that breaks a playbook into self-contained, reusable units. This lab converts a flat playbook into a role-based structure, and explains *why* each design decision exists so you can apply the same reasoning to your own automation.

### Prerequisites

In VS Code, connected to your Ansible controller:

1. In the Explorer panel, right-click and select **New Folder**
2. Name it `lab-roles`
3. Right-click the new folder and select **Open in Integrated Terminal**

---

## Step 1 – Understand Why Roles Exist

Before writing a single line, it is worth understanding the problem roles solve.

Consider a playbook that installs Apache, configures the firewall, adds a login banner, manages the `/etc/hosts` file, and creates a system user. Putting all of that in one file creates several problems:

| Problem | Effect |
|---|---|
| Everything in one file | A 300-line playbook is hard to read and harder to review |
| Repeated logic across playbooks | Updating the banner means editing every playbook that sets it |
| No clear ownership | Which team owns the user-management section vs. the web section? |
| Hard to share | You cannot hand another team "just the user part" without copy-pasting |

A **role** solves all of these by giving each concern its own directory with a predictable layout. Ansible knows where to look for tasks, templates, files, variables, and handlers — so all the wiring is automatic.

---

## Step 2 – Create the Role Directory Structure

Navigate into your lab directory and create the structure for a role called `baseline`:

```
cd /home/ansible/lab-roles
mkdir -p baseline/{templates,tasks,files}
echo "---" > baseline/tasks/main.yml
```

**Why `baseline`?** The name communicates intent. A `baseline` role captures everything that *every* managed node should have, regardless of what else runs on it. Web servers get `baseline` plus a web role; database servers get `baseline` plus a database role. The name signals "apply this everywhere."

**Why `mkdir -p baseline/{templates,tasks,files}`?** Ansible looks for content in specific subdirectories relative to the role root. The brace expansion creates three subdirectories in one command:

| Subdirectory | What Ansible looks for there |
|---|---|
| `tasks/` | Task files (entry point is always `main.yml`) |
| `templates/` | Jinja2 template files (`.j2`), used by the `template` module |
| `files/` | Static files to be copied verbatim, used by the `copy` module |

Other standard subdirectories (`handlers/`, `vars/`, `defaults/`, `meta/`) are optional and only needed when you use those features.

**Why `echo "---" > baseline/tasks/main.yml`?** Ansible requires `tasks/main.yml` to exist in every role, even if it only imports other task files. Starting it with `---` (the YAML document marker) is a convention that makes YAML linters happy and signals "this is the entry point."

Your directory tree should now look like this:

```
lab-roles/
└── baseline/
    ├── files/
    ├── tasks/
    │   └── main.yml
    └── templates/
```

---

## Step 3 – Deploy the MOTD Template

The **Message of the Day** (MOTD) is the text displayed to users when they log in over SSH. Deploying a consistent, managed MOTD across all servers is a common baseline requirement — it identifies the system and signals that access is monitored.

### 3a – Copy the Template File

```
cp /home/ansible/automation-dev/labs/roles/resources/motd.j2 baseline/templates/
```

**Why copy rather than create from scratch?** The provided `motd.j2` uses `{{ ansible_hostname }}` — an Ansible *fact* that resolves to the managed node's hostname at run time. This single template produces a correctly labelled banner on every server without any per-host customisation.

**Why a `.j2` extension?** `.j2` signals that the file is a **Jinja2** template. Jinja2 is the templating engine Ansible uses. Any `{{ variable }}` or `{% logic %}` blocks inside the file are expanded by Ansible before the file is written to the target. Naming the file `.j2` makes it immediately obvious to teammates that the file is not a static copy.

### 3b – Create the MOTD Task File

```
vi baseline/tasks/deploy_motd.yml
```

Add the following content:

```yaml
---
- template:
    src: motd.j2
    dest: /etc/motd
```

Save and exit with **Escape** followed by `:wq`.

**Why the `template` module instead of `copy`?** The `copy` module transfers a file exactly as it is. The `template` module processes Jinja2 expressions first, so `{{ ansible_hostname }}` in `motd.j2` becomes the real hostname of each target node. If you used `copy`, every node would literally display the text `{{ ansible_hostname }}` rather than its actual name.

**Why `dest: /etc/motd`?** `/etc/motd` (Message of the Day) is the Linux standard for the file that SSH and login daemons display after a successful authentication. Writing here requires root privileges, which is why `become: yes` is set at the play level in the playbook you will create later.

**Why put this in its own file (`deploy_motd.yml`) instead of directly in `main.yml`?** Separating task groups into individual files keeps each file focused on one concern. `main.yml` becomes a table of contents: it shows *what* the role does at a glance, and each imported file shows *how* one piece works. This also makes it easy to disable a section — just comment out the `import_tasks` line in `main.yml`.

### 3c – Register the Task in `main.yml`

```
vi baseline/tasks/main.yml
```

Add:

```yaml
- name: configure motd
  import_tasks: deploy_motd.yml
```

Save and exit with **Escape** followed by `:wq`.

**Why `import_tasks` instead of `include_tasks`?** Both load an external task file, but they behave differently:

| | `import_tasks` | `include_tasks` |
|---|---|---|
| When processed | At parse time (before the play runs) | At run time (when the task is reached) |
| Tags and conditions | Inherited by every task in the file | Not inherited — must be applied individually |
| Best for | Static, always-run task lists | Conditional or looped task loading |

For a baseline role whose task files are always loaded, `import_tasks` is the right choice: tags applied to the `import_tasks` line automatically propagate to every task in the imported file.

---

## Step 4 – Manage the `/etc/hosts` File

Linux resolves short hostnames to IP addresses using `/etc/hosts` before it queries DNS. For internal tooling — monitoring agents, configuration management callbacks, inter-service communication — adding a canonical entry ensures resolution works even when DNS is unavailable or misconfigured.

### 4a – Create the Hosts-Edit Task File

```
vi baseline/tasks/edit_hosts.yml
```

Add the following content:

```yaml
---
- lineinfile:
    line: "{{ ansible_default_ipv4.address }} {{ inventory_hostname_short }}.example.com"
    path: /etc/hosts
```

Save and exit with **Escape** followed by `:wq`.

**Why `lineinfile` instead of `copy` or `template`?** `/etc/hosts` is a system-managed file that already contains entries Ansible did not create (the loopback address, for example). Replacing the entire file with `copy` or `template` would delete those entries and could break the system. `lineinfile` adds or updates exactly one line, leaving everything else intact. It is the right tool whenever you need to manage a single entry in a file you do not own outright.

**Why `ansible_default_ipv4.address`?** Ansible collects **facts** about each managed node at the start of a play. `ansible_default_ipv4.address` is the IP address of the node's primary network interface — the address other hosts on the network use to reach it. Using a fact rather than a hard-coded IP means this task works correctly across every node without any per-host configuration.

**Why `inventory_hostname_short`?** `inventory_hostname_short` is the portion of the inventory hostname before the first dot. If the inventory entry is `node1.corp.example.com`, the short name is `node1`. Appending `.example.com` gives a fully qualified domain name that fits a standard internal naming convention. You can adjust the domain suffix in a role variable if your environment uses a different zone.

### 4b – Register in `main.yml`

```
vi baseline/tasks/main.yml
```

Add to the bottom of the file:

```yaml
- name: edit hosts file
  import_tasks: edit_hosts.yml
```

Save and exit with **Escape** followed by `:wq`.

---

## Step 5 – Create the `noc` User and Deploy an SSH Key

The Network Operations Center (NOC) needs passwordless SSH access to every managed server so operators can log in quickly during an incident without hunting for credentials. Creating the `noc` user and deploying its authorised key is a perfect baseline task: it applies to every server, and it needs to happen before any application role runs.

### 5a – Copy the Provided `authorized_keys` File

```
cp /home/ansible/automation-dev/labs/roles/resources/authorized_keys baseline/files/
```

**Why the `files/` subdirectory?** When the `copy` module receives a relative path (no `/` at the start), Ansible looks for the file in the role's `files/` directory automatically. You do not need to specify the full path in your task, which keeps the task clean and means the role is self-contained — the key travels with the role.

### 5b – Create the User Deployment Task File

```
vi baseline/tasks/deploy_noc_user.yml
```

Add the following content:

```yaml
---
- user:
    name: noc

- file:
    state: directory
    path: /home/noc/.ssh
    mode: "0600"
    owner: noc
    group: noc

- copy:
    src: authorized_keys
    dest: /home/noc/.ssh/authorized_keys
    mode: "0600"
    owner: noc
    group: noc
```

Save and exit with **Escape** followed by `:wq`.

**Why three separate tasks?** Each task has a single, clear responsibility: create the user, create the `.ssh` directory, copy the key. Separating them makes the play output easier to read — you see exactly which step succeeded or failed. It also makes each task idempotent on its own: if the user already exists, the `user` module reports `ok` and moves on without touching the directory or the key.

**Why `user: name=noc` without extra parameters?** The `user` module creates the user with sensible defaults (no login shell that accepts passwords, home directory under `/home/noc`) and is idempotent: if the user already exists it does nothing. Only add parameters when you need to deviate from the defaults.

**Why `mode: "0600"` on the `.ssh` directory?** SSH enforces strict permission checks. If `~/.ssh` or `~/.ssh/authorized_keys` is world-readable or world-writable, SSH refuses to use the key and logs a warning. `0600` (owner read/write only) satisfies SSH's requirements and is the minimum needed for the key to work.

**Why `owner: noc` and `group: noc` on both the directory and the file?** The `noc` user must be able to read its own key. Setting ownership explicitly prevents a situation where the files end up owned by `root` (because `become: yes` runs tasks as root) and the `noc` user cannot read them.

### 5c – Register in `main.yml`

```
vi baseline/tasks/main.yml
```

Add to the bottom of the file:

```yaml
- name: set up noc user and key
  import_tasks: deploy_noc_user.yml
```

Save and exit with **Escape** followed by `:wq`.

---

## Step 6 – Review the Completed `main.yml`

Your `baseline/tasks/main.yml` should now look like this:

```yaml
---
- name: configure motd
  import_tasks: deploy_motd.yml

- name: edit hosts file
  import_tasks: edit_hosts.yml

- name: set up noc user and key
  import_tasks: deploy_noc_user.yml
```

**Why does `main.yml` only contain `import_tasks` lines?** This pattern turns `main.yml` into a *manifest*: one glance tells you everything the role does, without you needing to scroll through individual task details. The details live in the per-concern files. This is the role equivalent of a table of contents.

---

## Step 7 – Create the Playbook That Uses the Role

Copy the provided starting playbook and edit it to use the `baseline` role:

```
cp /home/ansible/automation-dev/labs/roles/resources/web.yml /home/ansible/lab-roles/
vi /home/ansible/lab-roles/web.yml
```

Edit it to match the following:

```yaml
---
- hosts: webservers
  become: yes
  roles:
    - baseline
  tasks:
    - name: install httpd
      yum:
        name: httpd
        state: latest
    - name: start and enable httpd
      service:
        name: httpd
        state: started
        enabled: yes
```

Save and exit with **Escape** followed by `:wq`.

**Why `roles:` before `tasks:`?** Ansible always runs roles before the `tasks:` block, regardless of the order in the file. Listing `roles:` first makes this explicit and signals to the reader that the baseline configuration is applied as a foundation before the application-specific tasks run. Think of it as: "configure the server, then deploy the application."

**Why keep the web tasks in the playbook rather than creating a second role?** For a two-task web setup (install, start/enable) an inline task block is appropriate. Roles are worth the overhead when a concern has multiple task files, templates, handlers, or variables. Premature extraction into a role adds structure without adding clarity.

**Why `state: latest` for `httpd` here?** The introductory lab uses `latest` to keep the example simple. In a production environment you would pin to a specific version (`state: present`) to avoid unexpected upgrades during routine playbook runs — the same idempotency reasoning from the playbook lab applies here.

---

## Step 8 – Create the Inventory

Create an `inventory` file in `lab-roles/`:

```
[webservers]
node1 ansible_host=<IP of TargetNode-1 from /home/ansible/inventory/inventory.yaml>
node2 ansible_host=<IP of TargetNode-2 from /home/ansible/inventory/inventory.yaml>
```

**Why a `webservers` group?** The role is designed to run on web servers. Targeting `webservers` rather than `all` means that adding a database node to the inventory later will not accidentally receive Apache. Inventory groups are the primary mechanism for controlling *where* automation runs — always be explicit.

---

## Step 9 – Review the Final Directory Structure

Before running, confirm your layout looks like this:

```
lab-roles/
├── inventory
├── web.yml
└── baseline/
    ├── files/
    │   └── authorized_keys
    ├── tasks/
    │   ├── main.yml
    │   ├── deploy_motd.yml
    │   ├── edit_hosts.yml
    │   └── deploy_noc_user.yml
    └── templates/
        └── motd.j2
```

**Why does this structure matter?** Ansible discovers role content by convention, not configuration. If a template is in `baseline/templates/`, the `template` module finds it automatically when `src: motd.j2` is specified. If you put it in `baseline/files/` by mistake, Ansible would not find it and the play would fail. Understanding the convention prevents confusing path errors.

---

## Step 10 – Run the Playbook

Execute the playbook:

```
ansible-playbook -i inventory web.yml
```

Watch the output. You will see Ansible work through the role tasks before the inline tasks:

1. **MOTD template** — the rendered banner is written to each node
2. **Hosts file entry** — each node's IP and short name are added to `/etc/hosts`
3. **NOC user** — the user is created, the `.ssh` directory is set up, and the key is deployed
4. **Install httpd** — Apache is installed from the inline `tasks:` block
5. **Start and enable httpd** — Apache is started and set to launch on boot

---

## Step 11 – Verify the Results

### Confirm the MOTD

Log in to `node1`:

```
ssh <node1's IP address>
```

You should see the ASCII art banner with `node1`'s hostname filled in. This confirms the template module resolved `{{ ansible_hostname }}` correctly for that specific node.

**Why does the hostname appear in the banner?** The `template` module collected the `ansible_hostname` fact from `node1` during the play's fact-gathering phase and substituted it before writing the file. On `node2`, the same template would produce a banner showing `node2`.

### Confirm the `noc` User

While still on `node1`, run:

```
id noc
```

You should see output like:

```
uid=1001(noc) gid=1001(noc) groups=1001(noc)
```

This confirms the user was created. To confirm the key was deployed:

```
sudo cat /home/noc/.ssh/authorized_keys
```

The output should match the public key in the `authorized_keys` file you copied into the role.

### Confirm Idempotency

Return to the controller and run the playbook again without changing anything:

```
ansible-playbook -i inventory web.yml
```

Every task should report `ok` — no `changed`. This confirms:

- `template`: the file content matches; no write needed
- `lineinfile`: the hosts entry already exists; no change
- `user`: the `noc` user already exists; no change
- `file`: the `.ssh` directory already has the correct permissions; no change
- `copy`: `authorized_keys` is already in place; no change

**Why does idempotency matter for roles?** Roles are designed to be applied repeatedly — on new servers during provisioning, on existing servers after a policy change, and by scheduled runs that enforce desired state. If a role made changes every time it ran, operators could not tell from the output whether something genuinely changed. An all-`ok` run is a clean bill of health.

---

## Concepts Summary

| Concept | What it solves |
|---|---|
| **Roles** | Break large playbooks into focused, reusable units with a predictable layout |
| **`tasks/main.yml` as manifest** | One file shows what the role does; individual files show how each piece works |
| **`import_tasks`** | Statically includes a task file so tags and conditions propagate automatically |
| **`template` module** | Deploys files with dynamic content resolved per-host using Jinja2 and facts |
| **`lineinfile` module** | Manages a single line in a shared file without overwriting other content |
| **`user` + `file` + `copy`** | Three-step pattern to create a user, set up `.ssh`, and deploy an authorised key |
| **`roles:` before `tasks:`** | Makes execution order explicit: foundation first, application second |
| **Idempotency** | Safe to run repeatedly; `ok` means desired state is already met |

---

## Conclusion

You have converted a flat playbook into a role-based structure that separates three distinct concerns — login banners, host-name resolution, and user management — into individual, testable task files. The `baseline` role can now be applied to any group of servers by adding one line to a playbook, and each concern can be maintained independently without risk of breaking the others.

The patterns used here — a manifest `main.yml`, per-concern task files, templates for dynamic content, `lineinfile` for shared files, and explicit ownership for sensitive files — appear in virtually every production Ansible role. Congratulations!
