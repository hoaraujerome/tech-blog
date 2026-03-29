+++
date = '2024-01-28T15:09:48-04:00'
title = 'Linux Cheat Sheet'
+++

*   Remove execution permission to "others" on every regular files inside a directory
    

```bash
find directory_name -type f -exec chmod a-x {} ';'
# Avoid using 'chmod -R' as the execution permission is interpreted differently on a directory compared to a regular file.
```

*   Edit your cron configuration file
    

```bash
crontab -e
# Always use -e and not crontab file_name 
# Also do NOT edit it directly in /var/spool/cron/<user>
```

*   Write errors to the filesystem playbook
    

```markdown
Determine which filesystem (FS) is full and which file is filling it up
1. `df -h` to look for FS that's 100% or more (possible) full
2. `du -h XX | sort -h` on the identified FS to determine which directory is using the most space. Rinse & repeat command until all the large files are discovered
3. Try `fuser` or `lsof` in case you cant determine which process is using a file

Note: `df` & `du` can have disparity in the space reported
```

*   Write useful shell scripts error messages
    

```markdown
* Error messages in STDERR
* Include name of the program that's issuing the error
* State what function / operation failed
* If a system call fails, include the perror string
* Exit with some code other than 0
```

*   Write shell script steps
    

```markdown
1. Develop the script as a pipeline, 1 step at a time, on the command line. Use bash.
2. Send output to stdout and check to be sure it looks right
3. At each step, use the shell's command history to recall pipelines and tweak them
4. Once the output is correct, execute the actual commands and verify they worked
5. Use `fc` to capture your work

Ex: `find . -type f -name '*.log' | grep -v .do-not-touch | while read fname; do echo mv $fname `echo $fname | sed s/.log/.LOG/`; done | sh -x`
```

*   Save systemd journal between reboots
    
    *   Ansible role : [here](https://github.com/hoaraujerome/kubernetes-the-hard-way-on-aws/tree/main/configuration/playbooks/roles/journald)
        
    *   Script :
        

```bash
# default journal configuration file /etc/systemd/journald.conf should not be edited directly
mkdir /etc/systemd/journald.conf.d
cat << END > /etc/systemd/journald.conf.d/storage.conf
[Journal]
Storage=persistent
END
systemctl restart systemd-journald
```
