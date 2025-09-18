# 🐧 Gentoo Lab Project

Welcome to my **Gentoo Linux learning lab**.  
This project documents my journey installing, configuring, and customizing Gentoo Linux from scratch.  
The goal is to **learn Linux deeply**, experiment with system internals, and share the process with others.

---

## 🎯 Project Goals

- Read the documentation 😭
- Understand the Linux boot process (bootloader → kernel → init → user space).
- Learn how to compile and configure a custom Linux kernel.
- Explore Gentoo’s Portage system, USE flags, and source-based package management.
- Document pitfalls, fixes, and lessons learned for future reference.
- Build automation scripts to simplify repetitive steps.
- Share knowledge with others via GitHub and GitHub Pages.

---

## 📂 Repository Structure

gentoo-lab/
│
├── README.md # Overview (this file)
├── installation/ # Step-by-step installation logs
│ ├── 01-booting.md
│ ├── 02-partitioning.md
│ ├── 03-kernel.md
│ ├── 04-bootloader.md
│ └── ...
│
├── configs/ # Config files
│ ├── make.conf
│ ├── kernel.config
│ ├── fstab
│ └── ...
│
├── scripts/ # Automation or helper scripts
│ ├── auto-partition.sh
│ ├── rebuild.sh
│ └── ...
│
├── notes/ # Reflections + learning notes
│ ├── 2025-09-18-first-boot.md
│ ├── use-flags-experiments.md
│ ├── kernel-tuning.md
│ └── ...
│
└── experiments/ # Benchmarks, testing, side projects
├── boot-speed.md
├── python-build-flags.md
└── ...

---

## 🚀 How to Use This Repo

- Browse `installation/` for a step-by-step guide to setting up Gentoo.
- Check `configs/` if you want reference configurations.
- Explore `notes/` for reflections and Linux lessons.
- Look into `scripts/` if you want automation examples.
- See `experiments/` for performance and customization tests.

---

## 🌱 Learning Roadmap

- [ ] Partition and set up filesystems
- [ ] Configure Portage and USE flags
- [ ] Build and install the Linux kernel
- [ ] Install and configure bootloader
- [ ] Configure system services
- [ ] Install a desktop environment
- [ ] Experiment with kernel tuning and benchmarks

---

## 🛠️ Tools Used

- **Gentoo Linux** – source-based Linux distribution
- **Portage** – Gentoo’s package management system
- **Shell scripting** – automation and setup helpers
- **QEMU/VirtualBox** – testing environment

---

## 📖 Resources

- [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:Main_Page)
- [Gentoo Wiki](https://wiki.gentoo.org/)
- [Kernel Newbies](https://kernelnewbies.org/)

---

## ✨ Author

Project by **Priviledge Zvidzai**  
Feel free to fork, open issues, or contribute suggestions!
