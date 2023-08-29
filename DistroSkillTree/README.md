# Distro Skill Tree

This page contains an interactive chart for navigating paths to obtaining
permissions to upload changes to Ubuntu archive. This can be used as a model
to help build an application for things such as certain packagesets, 
[MOTU](https://wiki.ubuntu.com/MOTU), 
[CoreDev](https://wiki.ubuntu.com/UbuntuDevelopers#Ubuntu_Core_Developers) 
as examples.

![Skill Tree](./packaging.svg)

<svg width="400" height="300" xmlns="./packaging.svg">
    <a href="./packaging.svg">
        <rect x="50" y="50" width="200" height="100" fill="blue" />
    </a>

</svg>

# Generation

Checked in SVG is generated with [d2](https://d2lang.com) using the following
command:

```
d2 --theme=300 --dark-theme=200 -l elk --pad 0 ./packaging.d2
```
