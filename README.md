# Kubernetes hybrid lab setup

TL;DR; **Lets build a hybrid (ARM and x64) Kubernetes cluster!**

Do you want to learn a lot about deployments, system architecture, and other
stuff, while creating a learning environment for Kubernetes?

Look no further.

When I got my hands on a cheap Dell R420, I set out to create my own bare
metal Kubernetes cluster at home. Since I o one hand wanted a pretty
production like environment, but did not want to pay for a second
server, I decided to use a Raspberry Pi 4 as the Kubernetes master, and said
server as the worker.

## Sections

00: Hard- and software decisions
01: Setting up the master
02: Setting up the worker
