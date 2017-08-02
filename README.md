# marx-planning
Collection of planning, brainstorming, research, resources for porting KVM to Redox

## Design goals

- Small code base. Xen is ~270k, we want smaller than that.
- Reuses tried, tested, stable components
- Works as a drop-in replacement for Xen in Qubes. See Qubes architecture discussion of Xen vs KVM.
- Runs on Redox
- Modular and clean
- As little modification to the KVM codebase as possible. We want to be able to match upstream KVM in perpetuity.

## Redox

Where to get updates on Redox: 

* https://redox-os.org/news/
* https://www.reddit.com/r/Redox/
* https://www.reddit.com/r/Redox/

Initial project idea:

* https://discourse.redox-os.org/t/redox-as-a-virtualization-platform-host-hypervisor/594
* https://www.reddit.com/r/Redox/comments/6pt2zm/how_difficult_to_port_kvm_to_redox/
