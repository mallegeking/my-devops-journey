# Learning Linux Basics

## Logging

- System Logging is a process of recording and monitoring the services, Kernel, and daemons on the system that are constantly running.
- System Logs --> /var

## syslog & Rsyslog
- The syslog service manages and sends logs to the system logger. Rsyslog is an advanced version of syslog
> logger -s Hello --> command to send a message to the syslog service

> dmesg --> command to view kernel messages
> /var/log/dmesg --> location of kernel messages, typically cleared and rewritten on every reboot   
> /var/log/kern.log --> persistent record of kernel activities 

## Managing Log files
- Log Rotation

## Network sharing
- The secure copy command (SCP) --> works like cp but has extended capabilities across networks. Operates over SSH
- rsync stands for remote synchronization.  Essential tool for copying files between hosts.

## Commands 
> scp
> rsnyc

## Simple HTTP Server
- Python allows you to instantly create a webserver, useful for sharing files over the network.

> python -m http.server

## NFS

> NFS --> Network File System
 - Standard protocol for file sharing in Linux 

 ## Samba
 > SMB Protocol --> Server Message Block
 
# Network Basics
- I got most of the basics down


- Moving on to Subnetting

## Subnetting
> ifconfig -a --> finding your IP Address 
> ip addr show --> finding your IP Address, modern approach


# On the topic of Networks on this site --> https://labex.io/linuxjourney
Goal is to know the topics back to back 