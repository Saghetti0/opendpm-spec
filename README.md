# OpenDPM Specification

The work-in-progress specification for OpenDPM, the open-source distributed password manager.

**Caution:** everything here is a WIP. There is ***a lot*** missing, and tons of half baked ideas and incomplete documentation. This repository is solely a place to push the nightly work from my hard drive. Don't bother trying to make sense of this now.

**Note:** I'm not accepting pull requests for this repository until it's sufficiently complete.

## Quick Rundown of OpenDPM

**Caution:** This is incomplete, rough garbage that skips over a lot of important details. Also subject to change.

OpenDPM is a fully distributed password manager that relies on a network of servers to host password vaults.

### Local Service

The local service is the heart of the OpenDPM system. It's responsible for interacting with nodes, hardware security modules, and applications requesting passwords and secrets. It provides a TCP port on localhost, allowing programs to connect and request secrets. These programs could be web browsers, OpenDPM GUIs, SSH clients, etc. When a program requests a secret, the local service will prompt the user to allow or deny, and if using an HSM, will require the user to press a physical button on the HSM to authorize. This is done to prevent malicious programs from authenticating themselves by simulating a mouse click on the prompt.

### Nodes

Nodes are servers that store password vaults. When creating a password vault, the local service discovers all nodes on the network using a peer-to-peer discovery system, and picks multiple random servers from the list to upload the vault to. Each node can have a list of other nodes that it's aware of, allowing a client to recursively traverse the network and find all nodes that know of each other. 

**Note: **this discovery system, although only done once, is inefficient if the OpenDPM network too big. This is pending a redesign to increase efficiency.

### HSM

The HSM (hardware security module) is an optional component of the OpenDPM architecture. All encryption decryption of a vault can be offloaded to an HSM in order to increase security, preventing malicious programs from accessing data they shouldn't be able to by peering into the process memory (such as keys and secrets that they haven't been authorized to access). The HSM can be provided in software as part of the Local Service, but is less secure than using dedicated hardware.



## Checklist

**Caution:** like everything else, not even this checklist is finished.

- [ ] Introduction
- [ ] Vault format
  - [x] General format (definition of chunks, blocks)
  - [ ] Block types
  - [ ] Categories
  - [x] Linked lists
- [ ] Network Protocol
- [ ] Local Protocol
- [ ] HSM Protocol
- [ ] Implementation details
  - [ ] Record fragments
  - [ ] HSM on resource-constrained hardware
  - [ ] Key derivation with PBKDF2
  - [ ] Keyboxes (container for encrypted key, password hint, and server list)
  - [ ] Server discovery
  - [ ] Patch keys