# Lab Setup 

# Move the lab repo into your repos folder

You already cloned the `modern-automation` repo during the **tf-publish-module** lab. Instead of cloning it again, move that copy into `C:\Users\tekstudent\Downloads\repos` so the rest of the labs can find it.

### Step 1: Close all Visual Studio Code windows

Windows will not move a folder that another program has open, so close **every** VS Code window before you start.

1. In each open VS Code window, click **File** > **Exit** (or click the **X** in the top right corner).
2. Make sure no VS Code windows remain on the taskbar. If one reopens, close it again.

### Step 2: Move the folder with Windows File Explorer

1. Press **Windows key + E** to open **File Explorer**.
2. In the address bar at the top, type the path below and press **Enter**:

   ```plaintext
   C:\Users\tekstudent\Downloads\terraform\tf-publish-module
   ```

   > If you cloned the repo somewhere else, browse to that location instead. You are looking for the folder named `modern-automation`.

3. Click once on the **`modern-automation`** folder to select it.
4. Press **Ctrl + X** to cut the folder.
5. In the address bar, type the destination path below and press **Enter**:

   ```plaintext
   C:\Users\tekstudent\Downloads\repos
   ```

   > If the `repos` folder does not exist, browse to `C:\Users\tekstudent\Downloads`, right-click in the empty space, choose **New** > **Folder**, and name it `repos`. Then open it.

6. Press **Ctrl + V** to paste the folder.
7. Confirm you now see `C:\Users\tekstudent\Downloads\repos\modern-automation` in File Explorer.

### Step 3: Open the repo in VS Code and pull the latest changes

1. Open a new Visual Studio Code window.
2. Click **File** > **Open Folder...**, browse to `C:\Users\tekstudent\Downloads\repos\modern-automation`, and click **Select Folder**.
3. If prompted about trusting the authors of the folder, choose **Yes, I trust the authors**.
4. Click the third icon in the left toolbar for source control. Next to **changes**, click the ellipses (three dots) and choose **pull**.

## Set up a remote SSH session in Visual Studio Code.   

### Create the SSH configuration file.

On the left sidebar, click the icon that looks like a computer with a connection icon.

In the Remote Explorer, hover your mouse cursor over **SSH**, click on the gear icon (⚙️) in the top right corner, and select the top option: `C:\Users\tekstudent\.ssh\config` This will open the SSH configuration file in a new editor tab.


### Add the SSH configuration for the lab servers.
Replace the following lines in the SSH configuration file, replacing `<IP of Tower server from the spreadsheet>` with the actual IP address of your Tower.

```plaintext
Host tower
  HostName <IP of Tower server from the spreadsheet>
  IdentityFile ~/Downloads/repos/modern-automation/keys/lab.pem
  User ansible
```

### Save the SSH configuration file.
Save the changes to the SSH configuration file and close it.


### Connect to the lab servers.
1. In the Remote Explorer, you should now see the entry for the Tower server under "SSH Targets."
2. Click on the entry to connect to the Tower server.
3. Visual Studio Code will open a new window connected to the Tower server.
4. When prompted for the Operating System, choose 'Linux'.  
5. Accept the SSH fingerprint
6. You can now open a terminal in this new window and run commands on the Tower server.

### Create a working directory

In Visual Studio Code, you can create a new folder or file as if it was on your local machine.
Click **Open Folder** and select `/home/ansible`.
In future labs, you will create a directory for each lab.

## Confirm Ansible is configured correctly 

In the VS Code window connected to your Ansible Tower, open a terminal and run: 
```bash 
ansible all -i inventory -m ping 
```

This command confirms ansible can connect to the managed nodes and they have Python installed. 

The expected output should look similar to below: 

```
ControlNode | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
TargetNode1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
TargetNode2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

## Congratulations!
You have successfully set up your lab environment and are ready to start working on the labs.
