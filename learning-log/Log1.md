# 📅 Month (July) 2025 — My DevOps Progress

## Here is where I will write about everything I have covered during the first Month

## What I did so far:

- Set up this DevOps journey portfolio 🎉
- Started learning about version control (Git and Github)
  Here are some of the resources I used to learn Git and GitHub:
  - [Pro Git Book (Online and Free)](https://git-scm.com/book/en/v2): A step-by-step guide to learning Git and GitHub. I don't recommend reading the entire book, but it's a good starting point and it fits my initial goal which is to learn the essentials and get up to speed.
  - [](learngitbranching.js.org): An interactive online tool that helps you learn Git visually. It covers all the basics, has great excercises, and is fun to use. It's a great way to practice and understand how Git works behind the closed curtains.

## Sections Concentrated on:

Branching and Merging
Conflict Resolution
Cherrypicking
Git plumbing

## Next Steps:

Continue learning Git and GitHub with focus on remotes
Git Techniques and Tricks
Advanced Topics like Rebasing, Tagging and Stashing

## Aha moment 🤯

- When I started learning Git, I already knew the basics like cloning from a remote repository, adding files, committing changes, and pushing to a remote repository. So I didn't really need to learn it all from scratch. However all this time, I didn't know that Git and Github are not the same thing. Going through the book and reading more about the Git Internals helped me understand more. I still use the book as a reference when I need an indepth understanding of certain aspects of Git. Learning git was definitely worth the time. Now I can navigat the system more confidently and consciously.

## Learning Linux Basics:

Funfact: Everything in Linux is a file, even the command line. Let that sink in lol 🤯

# Resources:

[Hack the box](https://app.hackthebox.com/)
[Linux Journey](https://linuxjourney.com/)

- learned the fundamentals of the command line, navigating files and directories.
- I have become comfortable moving around a computer using the terminal.
- Currently learning how to manipulate strings.

## Linux Directory Structure:

Resources: [Linux Directory Structure](https://www.geeksforgeeks.org/linux-unix/linux-directory-structure/)

- The Linux file system (FHS -> Filesystem Hierarchy Standard) is a tree-like structure, with directories as the nodes and files as the leaves. It's a hierarchical organization that allows for easy organization and navigation of files and directories.
  -Types of files in Linux: General files, Directory Files and Device files

## Topics covered in the last time:

- working with the text editor Vim
- User Management
- Permissions and Processes.
  These topics are foundational and have helped me understand more about the idea and the power behind using linux systems
  The coolest thing I learned was about the Terminal. There are two types of terminals: regular terminal devices and pseudoterminal devices. On my Ubuntu system, I can switch back and forth between these using Ctrl + Alt + F3 and Ctrl + Alt + F2.
  Definitely try this out and look it up to learn more about it

## The Kernel

- The kernel is the core of the operating system and is responsible for managing the hardware and software components of the system.
  /boot --> location of the kernel

# Commands

> strace ls --> command to see system calls made by the ls command

> uname -r --> command to see the kernel version

> lsmod --> command to see loaded modules

## Init

- System V Overview:
- the primary role of an init system is to start and stop essential processes
- 3 Major init implementations in linux are: upstart, systemd, and System V initaka SysV

# Commands

> service example service --status-all --> command to see status of all services
> initctl list --> command to see a list of all known Upstart jobs and their current states
> systemctl list-units --> command to see a list of all active units systemd is currently managing

## Process Utilization

# Commands

> top --> tracks what your processes are doing in real time
> htop --> like top but is more user friendly
> lsof --> shows open files and processes that are using them
> fuser --> identifies which processes are using specific files, sockets or filesystems

- Process Threads: Threads are units of execution within a process and are often called light weight processes.
- There are single threaded and multi threaded processes.
- All processes start out single-threaded.

# CPU Monitoring

> uptime
> cat /proc/cpuinfo --> view the number of cores on your system

# I/O Monitoring

> iostat

# Memory Monitoring

> vmstat

# Continous Monitoring

> sudo sar -q
