# kestrel-it-lab

Hi 👋 This repo is where I'm building the IT setup for a made-up company from scratch, one piece at a time.

**Kestrel Freight Pty Ltd is not a real company.** It's a pretend freight and warehousing business with about 40 staff and a head office in Canberra. I wanted an actual "company" to work on instead of random one-off labs, so every lab here starts with a request from someone at Kestrel and builds on the lab before it.

Everything in this repo is lab work. No real company, no real users, and nothing taken from any job I've had.

## Why I'm doing this

I already work in IT support, but a lot of what you do on a service desk is fixing things someone else built. I wanted to build the whole thing myself, understand why each piece is set up the way it is, and then break it on purpose. Fixing a lockout teaches you more than reading about one.

## How each lab works

Every lab follows roughly the same pattern:

1. A ticket or request from someone at Kestrel
2. Learning the topic and how it shows up on a real service desk
3. Building it. I often follow a tutorial for the build and adapt it to Kestrel's names and IP ranges. When I do, the tutorial is credited in that lab's README.
4. Testing that it actually works
5. Breaking it and fixing it, written up like an incident ticket
6. A KB article someone else could follow

## Progress

| Lab | Topic | Status |
|-----|-------|--------|
| [F0](labs/00-lab-setup/) | Lab setup: Azure budget, network, admin access | Complete ✅ |
| [F1](labs/01-service-desk/) | Service desk (Jira Service Management) | Complete ✅ |
| [F2](labs/02-active-directory/) | Domain controller and first accounts | Complete ✅ |
| [F3](labs/03-file-shares/) | Shared folders and permissions | Complete ✅ |
| F4 | Group Policy | Not started |
| F5 | DNS and DHCP | Not started |
| F6 | Windows client troubleshooting | Not started |
| F7 | Printers | Not started |
| F8 | Restoring deleted files | Not started |
| F9 | Entra ID and MFA | Not started |
| F10 | Network basics on Cisco gear | Not started |
| F11 | Mac support basics | Not started |
| F12 | Onboarding and offboarding | Not started |

## Where things run

- **Azure (Australia Southeast)** for the Windows Server and Active Directory labs
- **My own Cisco switches and routers** for the networking and DHCP labs
- **VMs on my Mac** for the macOS and Windows 11 client labs

## Repo layout

```
kestrel-it-lab/
├── README.md
├── docs/
│   └── standards/      naming, tagging and IP plan
└── labs/              one folder per lab: README, evidence, incident write-ups
    ├── 00-lab-setup/
    ├── 01-service-desk/
    ├── 02-active-directory/
    └── 03-file-shares/
```

## Keeping the cost down

This is paid for out of my own pocket, so there's a budget alert at A$50 a month, VMs get shut down after every session, and anything I don't need gets deleted once the lab is written up. The service desk runs on Jira's free plan. One thing I learned from my bill: a stopped VM still pays for its disk, so a VM I'm not using costs money just by existing.

---

Shankar Baral · Canberra · [LinkedIn](https://www.linkedin.com/in/shankarbaral1)
