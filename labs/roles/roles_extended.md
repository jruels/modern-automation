# Ansible Roles – Extended

## The Scenario

The playbook you built in the previous lab deploys a web server — but it is a one-size-fits-all solution. The team now needs to deploy the same Apache setup to two different environments:

- **Production**: serves `prod.example.com`, document root `/var/www/prod`
- **Staging**: serves `staging.example.com`, document root `/var/www/staging`

Duplicating the playbook for each environment creates two files to keep in sync. Adding a third environment means a third file. Variables at the playbook level help, but every team that wants to run an Apache server still has to copy your playbook and understand every task in it.

A **role** solves this by packaging the entire web server setup — tasks, handlers, templates, and defaults — into a single reusable unit. Callers pass in the variables they care about and the role handles everything else. One role, any number of environments.

Along the way you will learn *why* each role feature exists, not just *what* it does.

### Prerequisites

In VS Code, create a new lab directory named `lab-roles-ext`:

1. In the Explorer panel, right-click and select **New Folder**
2. Name it `lab-roles-ext`
3. Right-click the new folder and select **Open in Integrated Terminal**

You should have completed the extended playbook lab. The concepts used here (handlers, templates, loops, tags, `uri` verification) are introduced there.

---

## Step 1 – Plan the Role Before Writing Anything

The role will be called `webserver`. Before creating files it is worth mapping out what the role will do and how the directory structure supports it.

### What the role does

| Concern | Task file | What it contains |
|---|---|---|
| Packages | `tasks/packages.yml` | Install httpd and firewalld, start and enable both |
| Configuration | `tasks/configure.yml` | Deploy virtual host template, open firewall port |
| Content | `tasks/content.yml` | Create document root, deploy a templated index page |
| Verification | `tasks/verify.yml` | HTTP check that Apache is serving the right page |

### What makes it reusable

All values that differ between environments live in `defaults/main.yml`. A caller can override any of them without touching the role itself.

### The full directory layout

```
lab-roles-ext/
├── inventory
├── site.yml
└── webserver/
    ├── defaults/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── tasks/
    │   ├── main.yml
    │   ├── packages.yml
    │   ├── configure.yml
    │   ├── content.yml
    │   └── verify.yml
    └── templates/
        ├── vhost.conf.j2
        └── index.html.j2
```

**Why plan the layout before writing code?** Roles follow Ansible's convention-over-configuration principle: Ansible discovers content by looking in specific subdirectories. If a template is in the wrong place Ansible will not find it and the play will fail with a confusing error. Sketching the layout first prevents that class of mistake.

---

## Step 2 – Create the Directory Structure

```
cd /home/ansible/lab-roles-ext
mkdir -p webserver/{defaults,handlers,tasks,templates}
```

**Why are there more subdirectories than in a basic role?** Each subdirectory corresponds to a role feature:

| Subdirectory | What Ansible loads from it | When you need it |
|---|---|---|
| `tasks/` | Task files — entry point is `main.yml` | Always |
| `templates/` | Jinja2 `.j2` files for the `template` module | When files need per-host values |
| `defaults/` | Default variable values — lowest priority | When callers should be able to override variables |
| `handlers/` | Handlers scoped to this role | When tasks in the role trigger service restarts |

The `intro` lab only needed `tasks/`, `templates/`, and `files/` because the baseline role had no configurable variables and no handlers. The `webserver` role needs all four.

---

## Step 3 – Define Default Variables

Create `webserver/defaults/main.yml`:

```yaml
---
http_port: 80
site_name: "default.example.com"
web_root: "/var/www/html"
packages:
  - httpd
  - firewalld
```

**Why `defaults/` instead of `vars/`?** Ansible has two places to define role variables, and they differ in priority:

| Location | Priority | Designed for |
|---|---|---|
| `defaults/main.yml` | Lowest of all variable sources | Values a caller *should* override |
| `vars/main.yml` | High — overrides most caller-supplied values | Internal constants the role relies on |

`defaults/` is intentionally the weakest variable source so that anything the caller passes in — host variables, group variables, playbook `vars:` — automatically wins. Think of defaults as the role's *documented API*: "here are the knobs you can turn." `vars/` is for internal values the role author does not want callers to change.

**Why define `packages` as a list?** The `packages.yml` task file will loop over this list. If a caller needs to add a package (say, `mod_ssl` for HTTPS), they override the list rather than editing the role. The role stays untouched.

---

## Step 4 – Define the Handlers

Create `webserver/handlers/main.yml`:

```yaml
---
- name: Restart httpd
  service:
    name: httpd
    state: restarted

- name: Reload firewalld
  service:
    name: firewalld
    state: reloaded
```

**Why do handlers live inside the role rather than in the playbook?** When handlers are defined in the playbook, every playbook that uses the role has to copy those same handler definitions. Putting handlers inside `handlers/main.yml` makes the role self-contained: the tasks that notify the handlers and the handlers themselves ship together. A caller gets working restart behaviour for free.

**Why two separate handlers?** Apache and firewalld require different operations to pick up changes. Apache needs a full restart when its configuration files change. Firewalld needs a reload (not a restart) to activate new permanent rules without dropping existing connections. Using separate handlers makes each trigger precise.

---

## Step 5 – Write the Packages Task File

Create `webserver/tasks/packages.yml`:

```yaml
---
- name: Install web server packages
  yum:
    name: "{{ item }}"
    state: present
  loop: "{{ packages }}"
  tags: packages

- name: Start and enable httpd
  service:
    name: httpd
    state: started
    enabled: yes
  tags: packages

- name: Start and enable firewalld
  service:
    name: firewalld
    state: started
    enabled: yes
  tags: packages
```

**Why loop over `packages` from `defaults/`?** The loop variable `{{ item }}` is replaced by each entry in the `packages` list. Because `packages` is defined in `defaults/main.yml`, a caller can add or swap packages without touching the task file. The task file describes *how* to install packages; `defaults/main.yml` describes *which* packages to install.

**Why `state: present` instead of `state: latest`?** `present` installs the package once and does nothing on subsequent runs if it is already installed — this is **idempotency**. `state: latest` would upgrade the package every time the playbook runs, which can introduce unplanned changes. In a role used across many servers, unexpected upgrades during a routine run are a reliability risk.

**Why tags on every task?** Tags in a role work the same as in a playbook: `ansible-playbook site.yml --tags packages` runs only the package tasks across every host. This is useful for a phased rollout where you want to confirm packages are in place before touching configuration.

---

## Step 6 – Write the Configure Task File

Create `webserver/tasks/configure.yml`:

```yaml
---
- name: Deploy virtual host configuration
  template:
    src: vhost.conf.j2
    dest: "/etc/httpd/conf.d/{{ site_name }}.conf"
    mode: "0644"
  notify: Restart httpd
  tags: config

- name: Open HTTP port in firewall
  firewalld:
    port: "{{ http_port }}/tcp"
    permanent: yes
    state: enabled
  notify: Reload firewalld
  tags: config
```

**Why `template` instead of `copy` for the virtual host file?** The virtual host configuration must contain the actual `site_name`, `web_root`, and `http_port` values for the environment being deployed. The `template` module processes Jinja2 expressions before writing the file, so `{{ site_name }}` in the template becomes `prod.example.com` for production and `staging.example.com` for staging. The `copy` module transfers files verbatim — it cannot substitute variables.

**Why `notify: Restart httpd` here but `notify: Reload firewalld` for the firewall task?** The handlers are named exactly as defined in `handlers/main.yml`. Ansible matches notify strings to handler names at the role level first, then the play level. The handlers will run at the end of the play — and only if the corresponding task reported a change.

**Why `port: "{{ http_port }}/tcp"` instead of `service: http`?** Using the variable means the role can serve on any port. The `service: http` shorthand hard-codes port 80. If `http_port` is later overridden to `8080`, the firewall rule updates automatically.

---

## Step 7 – Create the Virtual Host Template

Create `webserver/templates/vhost.conf.j2`:

```jinja2
# Managed by Ansible – do not edit by hand
<VirtualHost *:{{ http_port }}>
    ServerName {{ site_name }}
    DocumentRoot {{ web_root }}

    <Directory {{ web_root }}>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog  /var/log/httpd/{{ site_name }}-error.log
    CustomLog /var/log/httpd/{{ site_name }}-access.log combined
</VirtualHost>
```

**Why the `# Managed by Ansible` comment?** This is a widely used convention that signals to anyone who logs in manually: editing this file by hand is pointless because the next Ansible run will overwrite it. It also makes it easy to grep across a fleet for files under configuration management.

**Why `Options -Indexes`?** This directive disables directory listing. Without it, if the document root contains no `index.html`, Apache shows a browseable list of files — a common information disclosure vulnerability. Disabling it in the role template means every site the role creates is secure by default.

**Why are all three configurable values (`http_port`, `site_name`, `web_root`) from `defaults/main.yml`?** The template never contains hard-coded environment-specific values. All three come from role defaults and are overridable by the caller. The same `.j2` file produces a correct configuration for any combination of values.

---

## Step 8 – Write the Content Task File

Create `webserver/tasks/content.yml`:

```yaml
---
- name: Ensure document root exists
  file:
    path: "{{ web_root }}"
    state: directory
    mode: "0755"
    owner: apache
    group: apache
  tags: content

- name: Deploy index page
  template:
    src: index.html.j2
    dest: "{{ web_root }}/index.html"
    mode: "0644"
    owner: apache
    group: apache
  tags: content
```

**Why `file` with `state: directory`?** When `web_root` is overridden to a non-default path such as `/var/www/staging`, that directory probably does not exist yet. The `file` module with `state: directory` creates it — and if it already exists with the correct permissions, it reports `ok` and moves on. This is idempotent: safe to run whether or not the directory already exists.

**Why `owner: apache` and `group: apache`?** Apache runs as the `apache` user. Setting ownership explicitly ensures the process can read and write its own document root, and follows the principle of least privilege — the files are not owned by root.

**Why `template` for `index.html` instead of `copy`?** The index page will embed the hostname and IP address from Ansible facts so that each server's page identifies itself. A static `copy` cannot do this — a template can.

---

## Step 9 – Create the Index Page Template

Create `webserver/templates/index.html.j2`:

```jinja2
<!DOCTYPE html>
<html>
<head>
  <title>{{ site_name }}</title>
</head>
<body>
  <h1>{{ site_name }}</h1>
  <p>Served by: <strong>{{ ansible_hostname }}</strong></p>
  <p>Address: {{ ansible_default_ipv4.address }}</p>
</body>
</html>
```

**Why embed `ansible_hostname` and `ansible_default_ipv4.address`?** These are **facts** — values Ansible collects from the managed node at the start of every play. Using facts means the same template produces a unique, correctly labelled page on every server without any per-host customisation in the inventory or playbook. During troubleshooting, a page that identifies itself immediately tells you which node you are hitting.

**Why is this useful beyond the lab?** In production, the same pattern is used for health-check endpoints, status pages, and "canary" pages that load balancers can query to confirm which node is serving a request.

---

## Step 10 – Write the Verify Task File

Create `webserver/tasks/verify.yml`:

```yaml
---
- name: Register the site name on the managed host
  lineinfile:
    path: /etc/hosts
    line: "127.0.0.1 {{ site_name }}"
    state: present
  tags: verify

- name: Verify web server is responding
  uri:
    url: "http://{{ site_name }}:{{ http_port }}/index.html"
    status_code: 200
    return_content: yes
  register: response
  tags: verify

- name: Confirm page contains site name
  assert:
    that: site_name in response.content
    fail_msg: "Page did not contain '{{ site_name }}' — check the virtual host configuration"
    success_msg: "Page confirmed: '{{ site_name }}' found in response"
  tags: verify
```

**Why register the site name with `lineinfile` first?** The verification now reaches the server by its `site_name` (`prod.example.com`) instead of `localhost`. For a name to resolve it must exist somewhere — here we add a `127.0.0.1 {{ site_name }}` entry to `/etc/hosts` on the managed node so the mapping is in place before the `uri` task runs. Because the `uri` check runs *on the managed node*, the node only needs to resolve `site_name` to itself (`127.0.0.1`) for the request to reach its own Apache. `lineinfile` is idempotent: it adds the line only if it is not already present, so re-running the role does not create duplicate entries. `become: yes` is already set at the play level, which gives this task the root access needed to edit `/etc/hosts`.

**Why two tasks instead of one?** The `uri` task checks that Apache is running and responding with HTTP 200. The `assert` task checks that the *correct* virtual host is serving the request. A server could respond with 200 but serve the wrong site's content if the virtual host configuration is misconfigured. Two assertions catch two different failure modes.

**Why `register: response`?** `register` saves the full result of a task into a variable. The `assert` task can then inspect `response.content` — the raw body of the HTTP response — to confirm it contains the expected `site_name`. Without `register`, the response body is discarded after the `uri` task.

**Why `return_content: yes`?** By default the `uri` module only checks the status code and discards the body. `return_content: yes` instructs it to capture the response body and store it in `response.content` so the `assert` task can inspect it.

**Why `http://{{ site_name }}:{{ http_port }}/index.html`?** Verifying by `site_name` exercises the request path the way a real client would: it forces name resolution and makes Apache select the virtual host whose `ServerName` matches `site_name`. Hitting `localhost` would only confirm that *something* answers on the port — it would not prove that the named virtual host you just deployed is the one serving the page. The port still comes from `http_port`, so the URL stays correct whether the role is deployed on port 80 or an overridden port; hard-coding `80` would silently skip verification if `http_port` were overridden.

---

## Step 11 – Write `tasks/main.yml`

Create `webserver/tasks/main.yml`:

```yaml
---
- name: Install and start services
  import_tasks: packages.yml

- name: Configure virtual host and firewall
  import_tasks: configure.yml


- name: Deploy web content
  import_tasks: content.yml

- name: Flush handlers 
  meta: flush_handlers

- name: Verify deployment
  import_tasks: verify.yml
```

**Why does `main.yml` only import other files?** `main.yml` acts as a manifest: one glance shows the four phases of the role in order. The details live in the per-phase files. This structure scales well — you can add a fifth phase (say, `security.yml` for SELinux context fixes) by adding one line here, without touching any existing file.

**Why this order?** Services must be running before the firewall rule matters; the document root must exist before content is deployed; all of the above must be complete before verification can succeed. The order encodes these dependencies.

---

## Step 12 – Write the Playbook

The real payoff of a well-designed role is how simple the playbook becomes. Create `site.yml`:

```yaml
---
- name: Deploy production web server
  hosts: node1
  become: yes
  roles:
    - role: webserver
      vars:
        site_name: "prod.example.com"
        web_root: "/var/www/prod"

- name: Deploy staging web server
  hosts: node2
  become: yes
  roles:
    - role: webserver
      vars:
        site_name: "staging.example.com"
        web_root: "/var/www/staging"
```

**Why two plays in one playbook instead of two playbooks?** A single `site.yml` is the source of truth for the entire site. Running it deploys both environments in one command. Running `--limit node1` deploys only production. This is the standard pattern in Ansible: one playbook describes the whole system; inventory groups and `--limit` control scope.

**Why pass `vars:` on the `role:` key?** Variables passed here override the role's `defaults/main.yml` values for this play only. The role itself is unchanged. Production gets `/var/www/prod`; staging gets `/var/www/staging`; the role knows nothing about either — it just uses whatever `web_root` it receives. This is the role interface in action.

**Why is `http_port` not overridden?** Both environments use port 80, so the default value of `80` from `defaults/main.yml` is correct for both plays. Only override what differs. If staging were later moved to port 8080, you would add `http_port: 8080` to the staging play — one line, no role changes needed.

---

## Step 13 – Create the Inventory

Create an `inventory` file in `lab-roles-ext/`:

```
[webservers]
node1 ansible_host=<IP of TargetNode-1 from /home/ansible/inventory/inventory.yaml>
node2 ansible_host=<IP of TargetNode-2 from /home/ansible/inventory/inventory.yaml>
```

---

## Step 14 – Review the Final Directory Structure

Confirm your layout matches this before running:

```
lab-roles-ext/
├── inventory
├── site.yml
└── webserver/
    ├── defaults/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── tasks/
    │   ├── main.yml
    │   ├── packages.yml
    │   ├── configure.yml
    │   ├── content.yml
    │   └── verify.yml
    └── templates/
        ├── vhost.conf.j2
        └── index.html.j2
```

**Why verify the layout before running?** Ansible resolves role content by convention. A file in the wrong subdirectory produces a `file not found` error at runtime rather than a linting error at write time. Checking the structure first is faster than decoding a runtime failure.

---

## Step 15 – Run the Playbook

Execute the full site deployment:

```
ansible-playbook -i inventory site.yml
```

Watch the output. Because there are two plays, you will see Ansible complete the full role for `node1` (production) and then repeat it for `node2` (staging):

- Packages are installed on both nodes
- Each node gets a virtual host configuration file named after its `site_name`
- Each node gets a document root at its `web_root` path
- Each node gets an `index.html` embedding its own hostname and IP
- The `assert` task confirms each page contains the correct `site_name`

The handlers (`Restart httpd`, `Reload firewalld`) fire at the end of each play, after all tasks in that play have run.

---

## Step 16 – Explore Tags and Idempotency

### Run a single phase across both environments

```
ansible-playbook -i inventory site.yml --tags content
```

Only the content tasks run. This is useful when you have updated a template and want to redeploy pages without reinstalling packages or changing configuration.

### Confirm idempotency

Run the full playbook a second time without changing anything:

```
ansible-playbook -i inventory site.yml
```

Every task should report `ok`. Neither handler should fire. This is expected: the role already brought both nodes to desired state. Idempotency means the role is safe to run on a schedule (for drift detection) or in a pipeline (for continuous deployment).

### Change a variable and observe the handler

Edit `site.yml` and change the production `site_name` to `"prod2.example.com"`. Run:

```
ansible-playbook -i inventory site.yml --limit node1 --tags config
```

Because the virtual host filename and contents depend on `site_name`, the `template` task reports `changed` and the `Restart httpd` handler fires. The staging play is skipped entirely because of `--limit node1`. This demonstrates two role benefits at once: variable overrides and handler-driven restarts.

Revert `site_name` to `"prod.example.com"` and re-run to restore state.

---

## Concepts Summary

| Concept | What it solves |
|---|---|
| **`defaults/main.yml`** | Defines the role's public interface — values callers can override |
| **`handlers/main.yml`** | Scopes service restart logic to the role so callers do not need to define it |
| **Per-concern task files** | Each file has one responsibility; `main.yml` is a readable manifest |
| **Templates with role variables** | One template produces correct output for any environment |
| **`import_tasks`** | Tags and conditions propagate into imported files automatically |
| **`vars:` on `role:`** | Override defaults per-play to deploy the same role to different environments |
| **`register` + `assert`** | Verify not just that a service is running, but that it is serving the right content |
| **Idempotency** | Safe to run repeatedly; `ok` means desired state is already met |

---

## Conclusion

You have built a `webserver` role that encapsulates a complete Apache deployment — packages, handlers, virtual host configuration, document root, templated content, and end-to-end verification. The same role deploys two distinct environments from a single `site.yml` by overriding three default variables.

The features practiced here — `defaults/`, `handlers/`, per-concern task files, templates, variable overrides, and two-step verification — are the building blocks of every production-grade Ansible role. Congratulations!
