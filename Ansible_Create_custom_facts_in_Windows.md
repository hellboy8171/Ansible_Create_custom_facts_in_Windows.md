# Ansible: Create Local / Custom facts on Windows Servers
This demo follows on from [Ansible: How to manage Windows servers using winrm](https://youtu.be/aPN18jLRkJI)

Subscribe To Me On YouTube: https://bit.ly/lon_sub

Go through that video first to setup your environment.

**Please consider Subscribing to support my channel**.

If you manage a groups of servers, be they Linux or Windows using Ansible, sometimes you need to get a specific piece of information
from the servers you're managing. You can set this up quite easily by utilising "custom facts" in Ansible.

Custom Facts can be extremely useful if you only want to run a specific ansible tasks against a subset of servers that meet a condition.
Especially when that condition isn't available in the standard facts collected by the setup module.

### In this demo, I'll cover the following:

- How to setup ansible to create custom facts in Windows.
- Look at the syntax of the .ps1 file we run to generate the facts.
- Collect AWS facts like Region and AMI-ID
- Use these facts in a template.
- Finally we extend the template by collecting local environment variables (from the remote server) and add them into the template.
  
## The setup module
The setup module collects a load of facts from Linux and Windows hosts. You can run this command to list them out:

```
$ ansible -i hosts.ini all -m setup
```

Scrolling up you can see lots of useful information about the server.

### Custom facts:
Unlike Linux, if you want to create windows facts, you need to create a .ps1 file. When ansible runs using the "fact_path" option,
it will run the .ps1 script and make the new local facts available.

#### .ps1 file needed in Windows
The .ps1 file should look like this:

```
$InstanceType = $(Invoke-RestMethod -uri http://169.254.169.254/latest/meta-data/instance-type)
$AvailZone = $(Invoke-RestMethod -uri http://169.254.169.254/latest/meta-data/placement/availability-zone)
$AMI_ID = $(Invoke-RestMethod -uri http://169.254.169.254/latest/meta-data/ami-id)

@{
    local = @{
        local_facts = @{
            cloud = 'AWS'
            instance_type = $InstanceType
            avail_zone = $AvailZone
            ami_id = $AMI_ID
            environment = 'UAT'
            Support_Team = 'Windows_Team'
            Callout = '6-8'
        }
    }
}
```

At the top of the file, I'm using the "invoke-restmethod" to run an HTTP or HTTPS api call and save the results into a variable. 
More information [HERE](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod?view=powershell-7.1)

Because I want to collect AWS facts like Region and AMI-ID, this is a great way to do it.

Next in the file, I set the location of the facts to keep them in line with what I've previously done in Linux. It usually denotes
the name of the file, the title of the facts and then the facts themselves.

so if we want to call these facts using ansible we'd do this: ```ansible_local.local.local_facts.avail_zone```

Lets add the ```local.ps1``` file into our ansible playbook.

To keep it neat, lets create a "files" directory and add it in there.

```
$ mkdir files
$ vi files/local.ps1
$InstanceType = $(Invoke-RestMethod -uri http://169.254.169.254/latest/meta-data/instance-type)
$AvailZone = $(Invoke-RestMethod -uri http://169.254.169.254/latest/meta-data/placement/availability-zone)
$AMI_ID = $(Invoke-RestMethod -uri http://169.254.169.254/latest/meta-data/ami-id)

@{
    local = @{
        local_facts = @{
            cloud = 'AWS'
            instance_type = $InstanceType
            avail_zone = $AvailZone
            ami_id = $AMI_ID
            environment = 'UAT'
            Support_Team = 'Windows_Team'
            Callout = '6-8'
        }
    }
}
```

Save the Exit file.

Add the following tasks to a new local_facts.yml playbook.

```
$ vi local_facts.yml
---
- name: Manage windows servers
  hosts: win
  tasks:

  - name: Create Windows DIR
    win_file:
      path: C:\Temp\facts
      state: directory
    when: ansible_os_family == "Windows"

  - name: Copy powershell script to windows servers
    win_copy:
      src: files/local.ps1
      dest: 'C:\Temp\facts\'
    when: ansible_os_family == "Windows"

  - name: Add facts to a variable
    setup:
      fact_path: C:/TEMP/facts
    register: setupvar
```

Lets run the new plabook and make sure that ```local.ps1``` is copied over to the windows server.

```
$ ansible-playbook -i hosts.ini local_facts.yml

PLAY [Manage windows servers] *****************************************************************************

TASK [Gathering Facts] ************************************************************************************
ok: [15.237.60.55]

TASK [Create Windows DIR] *********************************************************************************
changed: [15.237.60.55]

TASK [Copy powershell script to windows servers] **********************************************************
changed: [15.237.60.55]

TASK [Add facts to a variable] ****************************************************************************
ok: [15.237.60.55]

PLAY RECAP ************************************************************************************************
15.237.60.55               : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Now that the fact file is in place, we can pull out the new facts using the setup module and ansible command line like this:

```
$ ansible -i hosts.ini all -m setup -a fact_path=C:/Temp/facts | sed '1c {'|jq '.ansible_facts.ansible_local.local.local_facts'
{
  "Callout": "6-8",
  "Support_Team": "Windows_Team",
  "ami_id": "ami-0d28ca2479c6273e4",
  "avail_zone": "eu-west-3b",
  "cloud": "AWS",
  "environment": "UAT",
  "instance_type": "t2.small"
}
```
We can go one further and pull out the AMI-ID or any of the other new facts lie this:

```
$ ansible -i hosts.ini all -m setup -a fact_path=C:/Temp/facts | sed '1c {'|jq '.ansible_facts.ansible_local.local.local_facts.ami_id'
"ami-0d28ca2479c6273e4"
```

Lets a new template file call ```facts.txt.j2``` and add the local facts in to get their values.
There are two ways to do this so pick the one you like the most.

```
AMI: {{ ansible_local.local.local_facts.ami_id }}
REGION: {{ ansible_local.local.local_facts.avail_zone }}

AMI: {{ ansible_local['local']['local_facts']['ami_id'] }}
REGION: {{ ansible_local['local']['local_facts']['avail_zone'] }}
```


Save the file. Now add to the ```local_facts.yml``` playbook and add in the new template we want to create on the windows server. Add this block to the bottom of the file:
```
  - name: Create a file from a Jinja2 template
    win_template:
      src: facts.txt.j2
      dest: C:\temp\folder\facts.txt
```

Then we can run Ansible again and check the file to see if it picks up the new local facts.
```
$ ansible-playbook -i hosts.ini local_facts.yml
...
```


Navigate to the facts.txt file on the Windows server and check the results. You should see something like this:
```

AMI: ami-0d28ca2479c6273e4
REGION: eu-west-3b

AMI: ami-0d28ca2479c6273e4
REGION: eu-west-3b
```

### Add local environment Variables
Now lets extend this by adding a local environment variable. One from the local server and one from the remote server to our template.

To add a local environment variable from the remote server, it's quite simple. Make sure you haven't disabled ```gather_facts``` in the local_facts.yml file.

#### Step one (remove environment variable):
Add this line to the ```facts.txt.j2``` file:

As I'm on windows, I'll use the ```HOMEPATH``` environment variable:

```
REMOTE_HOME_PATH: {{ ansible_env.HOMEPATH }}
```

Once you add this line, re-run Ansible and check it works. You should see this line in the template:

```
REMOTE_HOME_PATH: \Users\Administrator
```

#### Step two (Control node variable):
Now lets add a local ENV variable from the control node. For this we need to add the variable into the playbook like this under hosts:

```
- name: Manage windows servers
  hosts: win
  vars:
    shell: "{{ lookup('env', 'SHELL') }}"
  tasks:
```

And in the ```facts.txt.j2``` file, add this line:

```
CONTROL_NODE_SHELL: {{ shell }}
```

Run ansible again and you should see this line in the facts.txt file:

```
CONTROL_NODE_SHELL: /bin/bash
```

That's it. Have a play around with the local variables and environment variables and see what else you can do.

I hope you found this helpful. Don't forget to like and **Subscribe**
