# HackTheBox Solutions

A collection of solved HackTheBox machines with detailed walkthroughs and solutions for EJPT practice.

## Solved Machines

| Machine | Difficulty | Category | Status | Walkthrough |
|---------|-----------|----------|--------|-------------|
| Cape | Easy | Linux / IDOR / Linux Capabilities | ✅ Completed | [Cape Walkthrough](Cape/cape.md) |
| Connected | Easy | Linux / CVE-2025-57819 / FreePBX SQLi | ✅ Completed | [Connected Walkthrough](Connected/Connected.md) |
| Enigma | Easy | Linux / NFS / Roundcube / OliveTin | ✅ Completed | [Enigma Walkthrough](Enigma/Enigma.md) |
| Nexus | Easy | Linux / Gitea / Krayin CRM / Git path traversal | ✅ Completed | [Nexus Walkthrough](Nexus/nexus.md) |

Each solved machine has its own folder with the following structure:

```
machine-name/
└── machine-name.md (detailed walkthrough)
```

## How to Use

1. Navigate to the machine folder you're interested in
2. Read the **machine-name.md** file for the detailed walkthrough

## Machines Completed

### Easy
- Cape - IDOR vulnerability & Linux capabilities exploitation
- Connected - FreePBX unauthenticated SQLi (CVE-2025-57819) → RCE & privilege escalation
- Enigma - Season 11 Live machine with NFS credential leak, Roundcube reuse, OpenSTAManager upload, and OliveTin root RCE
- Nexus - Gitea credential leak, Krayin CRM file upload, and root escalation via template repository path traversal

### Medium

### Hard

### Insane

---

**Last Updated:** 2026-07-04 (Enigma and Nexus added)

**Note:** Walkthroughs are meant for learning purposes after attempting the machine yourself.
