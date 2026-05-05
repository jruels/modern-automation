# AAP 2.6 - Inventories, credentials, and ad-hoc commands

## Objective

This exercise will cover

- Locating and understanding:
  - Ansible Automation Controller **Inventory**
  - Ansible Automation Controller **Credentials**
- Running ad hoc commands via the Ansible Automation Platform web UI


### Log in to AAP

Access the Dashboard at the URL from the spreadsheet.

Log in to the dashboard with the username and the password from the credentials spreadsheet AAP tab.

## Create an Inventory

Let's get started: The first thing we need is an inventory of managed hosts. This is the equivalent of an inventory file in Ansible Engine. There is a lot more to it (like dynamic inventories), but let's start with the basics.

In the left navigation, expand **Automation Execution** → **Infrastructure** → **Inventories**, then click the **Create inventory** button and choose **Create inventory** from the dropdown (the other options are for smart and constructed inventories).

Provide the following:

* **Name**:  First Inventory-[your initials]
* **Description**: My first inventory file
* **Organization**: Default

Click **Create inventory**

On the inventory detail page, click the **Hosts** tab; the list will be empty since we have not added any hosts yet.

Let's add our hosts.

Click the **Create host** button and give a **Name** and **Description**:

* **Name**: Server1-[your initials]

* **Description**: Node from the spreadsheet

* Under **Variables,** confirm **YAML** is highlighted, and then paste the following:

  ```yaml
  ansible_host: <IP of TargetNode-1 from /home/ansible/inventory/inventory.yaml>
  ```

* Click **Create host**

You have now created an inventory with a new managed host.

Follow the same process to add Node 2. Return to **Automation Execution → Infrastructure → Inventories → First Inventory-[your initials] → Hosts** and click **Create host** again.

## Machine Credentials

One of the great features of the Ansible Automation Platform is to make credentials usable to users without making them visible. To allow AAP to execute jobs on remote hosts, you must configure connection credentials.

> **TIP**: This is one of the most important features of Automation Platform: **Credential Separation**! Credentials are defined separately and not with the hosts or inventory settings.

We need to configure the Ansible Automation Platform with the Controller SSH Private Key to enable it to connect to our managed nodes.

In the VS Code window that is connected to the Controller, expand `.ssh` and click `id_rsa`

Copy the **complete private key** (including "BEGIN" and "END" lines) and save it for the next step.

Now configure the credentials to access the managed hosts from Ansible Automation Platform.

In the left navigation, go to **Automation Execution** → **Infrastructure** → **Credentials**, and click **Create credential**, then fill in the following:

* **Name**: Linux credentials-[your initials]
* **Description**: Credentials to authenticate over SSH
* **Organization**: Default
* **Credential type**: Machine

Under **Type Details,** fill in:

* **Username**: ansible

* **SSH Private Key**: Paste the private key from above.

* **Privilege Escalation Method**: sudo

> **TIP**: Whenever you see a magnifying glass icon next to an input field, clicking it will open a list to choose from.

* Click **Create credential**

Go back to **Automation Execution → Infrastructure → Credentials → Linux credentials-[your initials]** and note that the SSH key is not visible.

You have now set up credentials for Ansible to access your managed host.

## Run Ad Hoc Commands

Ansible can run ad hoc commands from AAP as well.

In the web UI, go to **Automation Execution → Infrastructure → Inventories → First Inventory-[your initials]**

- Click the **Hosts** tab to change into the hosts view.
- Check the box next to each host you want to target (or select all).
- Click **Run command**. This opens a four-step wizard.

**Step 1 – Details**:

- **Module**: choose `ping`
- Leave the remaining fields (Arguments, Verbosity, Limit, Forks, Show Changes, Privilege Escalation, Extra Variables) at their defaults
- Click **Next**

**Step 2 – Execution Environment**:

- **Execution Environment**: Default execution environment
- Click **Next**

**Step 3 – Credential**:

- **Credential**: Linux credentials-[your initials]
- Click **Next**

**Step 4 – Review**:

- Confirm the values, then click **Finish** and watch the output.

The simple **ping** module doesn't need options. For other modules, you need to supply the command to run as an argument. Try the **command** module to find the user ID of the executing user using an ad-hoc command.

- **Module**: command
- **Arguments**: id

> **TIP**: After choosing the module to run, the controller will provide a link to the docs page for the module when clicking the question mark next to "Arguments". This is handy, give it a try.

## Challenge Lab: Ad Hoc Commands

Run an ad-hoc command to make sure the package `tmux` is installed on the host. If unsure, consult the documentation either via the web UI, as shown above, or by running `ansible-doc yum` on your AAP control host.

The instructor will provide the solution.

## Congrats!
