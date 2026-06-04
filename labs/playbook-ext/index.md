# Ansible Playbooks - Beyond the Basics

## The Scenario

The brochure-site deployment you built earlier was a success, and the team wants to expand the automation. The new requirements include:

- Multiple packages installed in one task
- A configurable virtual host so each deployment can serve a unique site name
- A firewall rule to open port 80
- Handlers to restart Apache only when configuration actually changes
- Tags so operators can run only part of the playbook (e.g., just update content without reinstalling packages)
- A post-deployment verification step that checks Apache is responding

Along the way you will learn *why* each Ansible feature exists, not just *what* it does.

### Prerequisites

In VS Code, create a new lab directory named `lab-playbook-ext`:

1. In the Explorer panel, right-click and select **New Folder**
2. Name it `lab-playbook-ext`
3. Right-click the new folder and select **Open in Integrated Terminal**

You should also have a working inventory from the previous playbook lab. If not, create one now.

---

## Step 1 – Create the Inventory

Create an `inventory` file. We are reusing the same two nodes but giving the group a more descriptive name: `webservers`.

```
[webservers]
node1 ansible_host=<IP of TargetNode-1 from /home/ansible/inventory/inventory.yaml>
node2 ansible_host=<IP of TargetNode-2 from /home/ansible/inventory/inventory.yaml>
```

**Why a named group?** Targeting `webservers` instead of `all` means you can safely add database or application nodes to the same inventory later without accidentally running web-server tasks on them.

---

## Step 2 – Understand Playbook Structure

Before writing any code, it helps to understand the three levels of structure Ansible uses:

| Level | What it is | Example |
|---|---|---|
| **Play** | Ties a set of tasks to a set of hosts | `- hosts: webservers` |
| **Task** | A single unit of work | Install a package, start a service |
| **Module** | The tool that does the actual work | `yum`, `service`, `copy` |

A playbook file can contain one play or many. In this lab you will write a single play and add features to it one section at a time.

---

## Step 3 – Add Variables

Hard-coding values like package names or file paths inside tasks makes a playbook brittle. If you need to change the site name you have to hunt through every task. **Variables** centralise those values so one change propagates everywhere.

Create `/home/ansible/lab-playbook-ext/web.yml` with the following content:

```yaml
---
- hosts: webservers
  become: yes

  vars:
    site_name: "demo.example.com"
    web_root: "/var/www/html"
    packages:
      - httpd
      - unzip
      - firewalld
```

**Why `become: yes`?** Installing packages and writing to `/etc` require root privileges. `become: yes` tells Ansible to use `sudo` for every task in this play unless a task overrides it.

**Why a `packages` list variable?** You will loop over it in the next step so you install all three packages with a single task, and adding a fourth package later is just one line in the `vars` section.

---

## Step 4 – Install Packages with a Loop

Append the `tasks` section to `web.yml`:

```yaml
  tasks:

    - name: Install required packages
      yum:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"
      tags: packages
```

**Why `loop`?** Without a loop you would need a separate task for each package. The loop variable `{{ item }}` is replaced by each entry in `packages` on successive iterations. Ansible shows individual pass/fail for each package, which makes troubleshooting easier.

**Why `state: present` instead of `state: latest`?** `latest` upgrades the package every time the playbook runs, which can introduce unplanned changes on a production server. `present` installs the package once and does nothing if it is already there — this is called **idempotency** and it is a core Ansible principle.

**Why `tags: packages`?** Tags let you run a subset of a playbook. Later you can run `ansible-playbook ... --tags packages` to install packages without touching the config or content. This is useful during a phased rollout.

---

## Step 5 – Add Handlers

A **handler** is a special task that only runs when it is *notified* by another task, and it runs at the end of the play, not immediately. This prevents a service from being restarted three times just because three config-related tasks each changed something.

Add the `handlers` block to `web.yml`, at the same indentation level as `tasks`:

```yaml
  handlers:

    - name: Restart httpd
      service:
        name: httpd
        state: restarted

    - name: Reload firewalld
      service:
        name: firewalld
        state: reloaded
```

**Why handlers instead of a `service` task with `state: restarted`?** A regular restart task runs every time the playbook runs, even when nothing changed. Handlers only trigger on actual changes, so they make your playbook safe to run repeatedly.

---

## Step 6 – Start and Enable Services

Add these tasks under your package installation task:

```yaml
    - name: Start and enable httpd
      service:
        name: httpd
        state: started
        enabled: yes
      tags: services

    - name: Start and enable firewalld
      service:
        name: firewalld
        state: started
        enabled: yes
      tags: services
```

**Why `enabled: yes`?** `state: started` brings the service up right now. `enabled: yes` makes it start automatically after a reboot. Both are almost always needed together.

---

## Step 7 – Open the Firewall

Add this task to allow HTTP traffic through the firewall:

```yaml
    - name: Open HTTP port in firewall
      firewalld:
        service: http
        permanent: yes
        state: enabled
      notify: Reload firewalld
      tags: firewall
```

**Why `notify: Reload firewalld`?** The `firewalld` module updates the permanent configuration, but firewalld only picks up permanent rules after a reload. The `notify` keyword schedules the `Reload firewalld` handler to run at the end of the play — but only if this task reports a change.

---

## Step 8 – Deploy a Virtual Host Configuration

A virtual host configuration file tells Apache which directory to serve for a given domain name. Instead of writing the file contents directly in the playbook, we will use the `copy` module with an inline `content` block and the `site_name` variable.

Add this task:

```yaml
    - name: Deploy virtual host configuration
      copy:
        dest: /etc/httpd/conf.d/{{ site_name }}.conf
        mode: "0644"
        content: |
          <VirtualHost *:80>
              ServerName {{ site_name }}
              DocumentRoot {{ web_root }}
              ErrorLog /var/log/httpd/{{ site_name }}-error.log
              CustomLog /var/log/httpd/{{ site_name }}-access.log combined
          </VirtualHost>
      notify: Restart httpd
      tags: config
```

**Why inline `content` instead of a separate file?** For a short config block, keeping the content in the playbook makes the lab self-contained. In production you would use the `template` module with a `.j2` file so the configuration can have more complex logic.

**Why notify `Restart httpd`?** A configuration change only takes effect after Apache restarts. The handler ensures that happens automatically if and only if the config file actually changed.

---

## Step 9 – Create the Web Root and Deploy Content

Add these tasks to create the document root directory and deploy the website:

```yaml
    - name: Ensure web root directory exists
      file:
        path: "{{ web_root }}"
        state: directory
        mode: "0755"
        owner: apache
        group: apache
      tags: content

    - name: Download website archive
      get_url:
        url: https://github.com/jruels/ansible-best-practices/raw/main/labs/playbook-fun/files/website.zip
        dest: /tmp/website.zip
      tags: content

    - name: Deploy website content
      unarchive:
        remote_src: yes
        src: /tmp/website.zip
        dest: "{{ web_root }}/"
        owner: apache
        group: apache
      tags: content
```

**Why set `owner` and `group` to `apache`?** The Apache process runs as the `apache` user. If the files are owned by root, Apache can still read them (world-readable), but setting the correct owner follows the principle of least privilege and is required in some SELinux configurations.

**Why `remote_src: yes`?** This tells Ansible that `/tmp/website.zip` already lives on the managed node (we just downloaded it there with `get_url`) rather than on the control node. Without this flag Ansible would try to copy the archive from the machine running the playbook.

---

## Step 10 – Add a Verification Task

The final task uses the `uri` module to make an HTTP request from each managed node to itself and confirm Apache is serving content correctly.

```yaml
    - name: Verify web server is responding
      uri:
        url: "http://localhost/home.html"
        status_code: 200
      tags: verify
```

**Why `http://localhost/home.html` instead of just `/`?** The website archive uses `home.html` as its main page rather than `index.html`. Requesting `/` would return a 403 Forbidden because Apache's directory listing is disabled and there is no `index.html` to fall back on. Pointing the check at the actual page gives a meaningful 200 result.

**Why check `status_code: 200`?** The `uri` module fails by default if the HTTP status is 4xx or 5xx, but being explicit about `200` makes the intent clear to anyone reading the playbook later.

**Note on external verification:** In a production environment you would also test from outside the server — either using `delegate_to: localhost` on the control node, or from a monitoring system — to confirm the firewall and network path are open to end users. In this lab environment the cloud security group does not expose port 80 externally, so we verify locally on each node instead.

---

## Step 11 – Review the Complete Playbook

Your finished `web.yml` should look like this:

```yaml
---
- hosts: webservers
  become: yes

  vars:
    site_name: "demo.example.com"
    web_root: "/var/www/html"
    packages:
      - httpd
      - unzip
      - firewalld

  handlers:

    - name: Restart httpd
      service:
        name: httpd
        state: restarted

    - name: Reload firewalld
      service:
        name: firewalld
        state: reloaded

  tasks:

    - name: Install required packages
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
      tags: services

    - name: Start and enable firewalld
      service:
        name: firewalld
        state: started
        enabled: yes
      tags: services

    - name: Open HTTP port in firewall
      firewalld:
        service: http
        permanent: yes
        state: enabled
      notify: Reload firewalld
      tags: firewall

    - name: Deploy virtual host configuration
      copy:
        dest: /etc/httpd/conf.d/{{ site_name }}.conf
        mode: "0644"
        content: |
          <VirtualHost *:80>
              ServerName {{ site_name }}
              DocumentRoot {{ web_root }}
              ErrorLog /var/log/httpd/{{ site_name }}-error.log
              CustomLog /var/log/httpd/{{ site_name }}-access.log combined
          </VirtualHost>
      notify: Restart httpd
      tags: config

    - name: Ensure web root directory exists
      file:
        path: "{{ web_root }}"
        state: directory
        mode: "0755"
        owner: apache
        group: apache
      tags: content

    - name: Download website archive
      get_url:
        url: https://github.com/jruels/ansible-best-practices/raw/main/labs/playbook-fun/files/website.zip
        dest: /tmp/website.zip
      tags: content

    - name: Deploy website content
      unarchive:
        remote_src: yes
        src: /tmp/website.zip
        dest: "{{ web_root }}/"
        owner: apache
        group: apache
      tags: content

    - name: Verify web server is responding
      uri:
        url: "http://localhost/home.html"
        status_code: 200
      tags: verify
```

---

## Step 12 – Run the Playbook

Execute the full playbook:

```
ansible-playbook -i inventory web.yml
```

Watch the output carefully. You should see:

- Each package listed separately in the loop output
- The firewalld reload and httpd restart handlers fire **at the end** of the play, not inline
- The verification task succeed with status 200 for each node

### Run Only Specific Tags

Try re-running with a single tag to see how tags limit scope:

```
ansible-playbook -i inventory web.yml --tags content
```

Only the three content tasks will run. This is useful when you have updated the website archive and want to redeploy content without touching packages or configuration.

### Confirm Idempotency

Run the full playbook a second time without changing anything:

```
ansible-playbook -i inventory web.yml
```

Every task should show `ok` instead of `changed`, and neither handler should fire. This confirms your playbook is safe to run repeatedly — the definition of an idempotent playbook.

---

## Step 13 – Change a Variable and Observe the Handler

Open `web.yml` and change `site_name` to `"staging.example.com"`, then re-run the playbook:

```
ansible-playbook -i inventory web.yml --tags config
```

Because the virtual host configuration file changed, the `Restart httpd` handler fires. Every other task reports `ok` because nothing else changed. This demonstrates how handlers prevent unnecessary service restarts.

After verifying, change `site_name` back to `"demo.example.com"` and re-run to restore the original state.

---

## Concepts Summary

| Feature | What it solves |
|---|---|
| **Variables** | Avoid hard-coding values; one change updates everywhere |
| **Loop** | Install multiple packages in one task without repetition |
| **Idempotency** (`state: present`) | Safe to run repeatedly without unintended side effects |
| **Handlers** | Restart/reload services only when a real change occurred |
| **Tags** | Run a subset of the playbook for faster, safer partial updates |
| **`uri` module** | Verify a web service is responding as part of the playbook itself |

---

## Conclusion

You have written a production-style Ansible playbook that installs packages, manages services, opens a firewall port, deploys a virtual host configuration, publishes website content, and verifies the result — all without repeating yourself and all safe to run multiple times.

The features you practised here (variables, loops, handlers, tags, idempotency) appear in virtually every real-world playbook. Congratulations!
