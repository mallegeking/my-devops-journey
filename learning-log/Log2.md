# Learning Linux Basics

## Logging

- System Logging is a process of recording and monitoring the services, Kernel, and daemons on the system that are constantly running.
- System Logs --> /var

## syslog & Rsyslog

- The syslog service manages and sends logs to the system logger. Rsyslog is an advanced version of syslog
> logger -s Hello --> command to send a message to the syslog service

>dmesg --> command to view kernel messages
- /var/log/dmesg --> location of kernel messages, typically cleared and rewritten on every reboot   
- /var/log/kern.log --> persistent record of kernel activities 

# Managing Log files
- Log Rotation 