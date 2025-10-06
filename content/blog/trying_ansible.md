+++
type = "post"
title = "Ansible for VM management"
date = 2025-02-27
+++

To run my ML experiments, I often have to manage a lot of small VMs: setting them up, configuring the proxy, setting up Python environments, mounting shared disks, and more. Larger servers are not always available when I need them, or I have to share them with other people, but small machines are usually easier to access. Initially, when it was only a few VMs, I had a sequence of commands that I ran manually. I also experimented with cloning images from previous setups, but in our fast-paced development environment, images quickly became impractical. There was always something that needed updating or tweaking. While images could serve as a starting point, they were never enough as I still had to apply incremental changes across multiple VMs, like mounting a new NFS disk.

Some time ago, I came across [this post](https://bsky.app/profile/howard.fm/post/3lbtqd35ldc26) by Jeremy Howard, where he shared a [script to setup a Linux VM](https://github.com/AnswerDotAI/fastsetup/blob/master/ubuntu-initial.sh) automatically. His script was already way better than the commands I was running manually, so I considered adopting it. But then, in the replies to his post, someone asked, "Why not Ansible?" That question intrigued me as I had never heard of Ansible.

[Ansible](https://docs.ansible.com/) is an open-source automation tool for managing and maintaining system configurations. That sounded like what I needed—not just for setting up my VMs, but also for handling incremental changes, like installing a new Python environment or mounting a new disk, without the hassle of SSHing into every single machine. So, I decided to give it a try.

## Setting up Ansible and defining tasks to automate
Installing Ansible was straightforward. Once installed, you define the tasks you want to automate in a YAML playbook file. As I was not familiar with Ansible, I asked ChatGPT to generate an Ansible playbook based on the steps I usually follow when setting up a VM.

```yaml
# setup.yml
- name: Configure Ubuntu VM
  hosts: all
  become: true
  vars:
    new_user: "new_user"
    new_user_password: "new_user_password"
    ssh_key: "/home/{{ new_user }}/.ssh/id_rsa"
    ssh_pubkey: "/home/username_on_local_machine/.ssh/id_rsa.pub"
    gitlab_ssh_url: "company.gitlab.com"
  
  tasks:
    - name: Create a new user
      user:
        name: "{{ new_user }}"
        password: "{{ new_user_password | password_hash('sha512') }}"
        shell: /bin/bash
        groups: sudo
        append: yes
        create_home: yes

    - name: Ensure .ssh directory exists for the user
      file:
        path: "/home/{{ new_user }}/.ssh"
        state: directory
        owner: "{{ new_user }}"
        group: "{{ new_user }}"
        mode: "0700"
  
    - name: Generate SSH key
      become_user: "{{ new_user }}"
      openssh_keypair:
        path: "{{ ssh_key }}"
        type: rsa
        size: 4096
        state: present
        owner: "{{ new_user }}"
        group: "{{ new_user }}"
        mode: '0600'

    - name: Get SSH key for manual upload to GitLab
      command: cat "{{ ssh_key }}.pub"
      register: ssh_key_output

    - name: Pause to let user add SSH key to GitLab
      pause:
        prompt: |
          Please add the following SSH key to your GitLab account:
          {{ ssh_key_output.stdout }}
          
          Once added, press Enter to continue.
  
    - name: Clone proxy repository
      become: true
      become_user: "{{ new_user }}"
      git:
        repo: "{{ proxy_repo }}"
        dest: "{{ proxy_clone_path }}"
        key_file: "{{ ssh_key }}"
        version: master
        accept_hostkey: yes

    - name: Run the proxy setup script
      become: true
      expect:
        command: "{{ proxy_clone_path }}/{{ proxy_script_path }}"

    - name: Update and upgrade system
      apt:
        update_cache: yes
        upgrade: yes

    - name: Copy local SSH public key to authorized_keys
      ansible.builtin.copy:
        src: "{{ ssh_pubkey }}"
        dest: "/home/{{ new_user }}/.ssh/authorized_keys"
        owner: "{{ new_user }}"
        group: "{{ new_user }}"
        mode: "0600"
    
    - name: Ensure correct permissions on authorized_keys
      file:
        path: "/home/{{ new_user }}/.ssh/authorized_keys"
        owner: "{{ new_user }}"
        group: "{{ new_user }}"
        mode: "0600"
```

This playbook automates several tedious setup steps: creating a new user, generating an SSH key, pausing for manual GitLab key registration, cloning a repository with a proxy setup script, updating the system, and configuring SSH access **This is just an example, please refer to best practices for proper VM setup**. I also added a task to mount a shared NFS disk (not shown here). You can use `ansible-vault` to encrypt your password.

Ansible can also rely on an inventory file to store VM IPs and other details, allowing it to execute the playbook across multiple machines.

```yaml
# inventory.ini
[setup]
ansible_host=<IP1> ansible_user=root
ansible_host=<IP2> ansible_user=root
...
```
I set `ansible_user` to `root`, meaning Ansible connects as `root` by default. It typically uses SSH keys for authentication, but you can also configure it to prompt for a password on the first connection. After the initial login, the playbook adds your SSH public key, making future connections passwordless.

To run the playbook while prompting for the root password:
```bash
ANSIBLE_SSH_ARGS="-o StrictHostKeyChecking=no" ansible-playbook -i inventory.ini setup.yml --ask-pass
```
This command disables strict host key checking (`-o StrictHostKeyChecking=no`), which is useful for first-time connections but can be a security risk.

## The challenge of setting up conda
Setting up a Python environment with `conda` and `pip` turned out to be a real headache for me. The usual SSH session commands didn’t work as expected, and ChatGPT was not really helpful.
Some key lessons:
* Ansible tasks are independent so `conda activate` must be called in the same task where packages are installed, which makes sense in hindsight.
* The Miniconda installer doesn’t automatically run `conda init`. I usually confirm this manually during installation, so I hadn’t realized it wouldn’t be done for me. I discovered that when following the default installation, and thus not running `conda init`, the following useful message appears
```bash
You have chosen to not have conda modify your shell scripts at all.
To activate conda's base environment in your current shell session:

eval "$(/home/<user>/miniconda3/bin/conda shell.YOUR_SHELL_NAME hook)"

To install conda's shell functions for easier access, first activate, then:

conda init
```
* `conda` commands require an interactive shell, meaning they must be executed using `bash -i -c "conda create/activate ..."` [^1].

This resulted in the following playbook:
```yaml
- name: Setup conda environment
  hosts: all
  vars:
    user: "new_user"
    miniconda_installer_url: "https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh"
    miniconda_install_path: "/home/{{ user }}/miniconda3"
    conda_env_name: "my_env"
    conda_python_version: "3.11.10"

  tasks:
    - name: Download Miniconda installer
      get_url:
        url: "{{ miniconda_installer_url }}"
        dest: "/home/{{ user }}/Miniconda3-latest-Linux-x86_64.sh"
        mode: '0755'

    - name: Install Miniconda
      shell: |
        bash /home/{{ user }}/Miniconda3-latest-Linux-x86_64.sh -b -p {{ miniconda_install_path }}
      args:
        creates: "{{ miniconda_install_path }}/bin/conda"

    - name: Initialize conda
      shell: |
        eval "$(/home/{{ user }}/miniconda3/bin/conda shell.bash hook)" && conda init
    
    - name: Create conda environment
      shell: |
        bash -i -c "conda create -n {{ conda_env_name }} python={{ conda_python_version }} -y"

    - name: Install packages
      shell: |
        bash -i -c "conda activate {{ conda_env_name }} && pip install numpy"
```

## The trade-offs of automating with Ansible
At the end of the day, automating my VM setup with Ansible was worth it, but it took more effort than I expected. This was my first time using Ansible, so there was a learning curve. Some of the frustrations, like dealing with `conda`, would have been just as painful in a bash script. But a downside of using another tool like Ansible is debugging. This is actually related to [the argument given by Jeremy Howard to why he was not using Ansible](https://bsky.app/profile/howard.fm/post/3lbxbvofy2c2c). Debugging a bash script is easy, you just run it line by line until something breaks. Ansible, on the other hand, adds an abstraction layer that can make troubleshooting more tedious. Sure, Ansible has a built-in debugger, but that’s yet another thing to learn and configure. 

That said, now that I have working playbooks, I do appreciate the convenience. Last week, I had to spin up new VMs, and it was a relief to just run the playbook instead of manually configuring everything.

[^1]: Another approach is to use `bash -c "source ~/miniconda3/etc/profile.d/conda.sh && conda create/activate ...` (see [this Stack Overflow answer](https://stackoverflow.com/a/55507956)). None of this is specific to Ansible—if I were writing a Bash script that SSHs into a machine to run `conda` commands, I’d run into the same issue.