# marx-planning
Collection of planning, brainstorming, research, resources for porting KVM to Redox

Porting KVM to run on the Redox kernel.

Design goals:

- Small code base. Xen is ~270k, we want smaller than that.
- Reuses tried, tested, stable components
- Works as a drop-in replacement for Xen in Qubes. See Qubes architecture discussion of Xen vs KVM.
- Runs on Redox
- Modular and clean
- As little modification to the KVM codebase as possible. We want to be able to match upstream KVM in perpetuity.
