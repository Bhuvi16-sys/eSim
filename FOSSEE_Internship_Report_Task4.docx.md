

| ![iitb.png][image1] | ![fossee.png][image2] |
| :---- | ----: |

**FOSSEE Semester Long Internship Report**

On

**eSim 2.5 on Ubuntu 25.04: Installation Problems, Diagnosis and Fixes**

*(Task 4: eSim Upgradation)*

**Submitted by**

**Bhuvi Jain**

*\[3rd, B.Tech\], Department of \[Computer Science\]*

*\[VIT Bhopal University\]*

November 2026

# **1\. Introduction**

## **1.1 Background**

eSim is an open-source electronic design automation (EDA) tool developed by FOSSEE, IIT Bombay. It integrates KiCad for schematic capture, Ngspice for circuit simulation, and NGHDL/GHDL for mixed-signal and VHDL simulation. It is installed through shell scripts that detect the Ubuntu release and call a script written for that release.

Ubuntu 25.04 (Plucky Puffin) is newer than the releases the official eSim 2.5 package targets (22.04, 23.04 and 24.04). Task 4 asks for eSim to be installed on 25.04, every failure to be documented, and the installer to be fixed where possible.

## **1.2 Objectives**

* Install the official eSim 2.5 package on a fresh Ubuntu 25.04 system.

* Find, diagnose and document each failure, with evidence.

* Fix the installer scripts where possible and commit each fix separately.

* State plainly what was and was not verified.

## **1.3 Scope Statement**

A complete, successful end-to-end installation of eSim on Ubuntu 25.04 was **not** obtained. The NGHDL/GHDL build was never run with the patches described here (see Section 6). The results below are therefore a diagnosis and a partial set of fixes, not a finished port.

# **2\. Technologies and Environment**

## **2.1 Technologies Used**

| Technology | Use in this task |
| ----- | ----- |
| Bash | Reading, tracing and editing the installer scripts (install-eSim.sh, install-eSim-scripts/, nghdl/install-nghdl.sh). Tools: bash \-n, grep, sed, nano. |
| Python 3.13 | The installer builds a virtual environment (\~/.esim/env) and installs PyQt5 5.15.11 with pip, in line with PEP 668\. |
| eSim 2.5 | Official package from esim.fossee.in. |
| Git and GitHub | Fork, upstream remote, the installers branch, git blame and git show for history; one fix per commit. |
| Ubuntu 25.04, VirtualBox | Test platform. |

## **2.2 Test Environment**

| Item | Value |
| ----- | ----- |
| Hypervisor | VirtualBox |
| Guest OS | Ubuntu 25.04 (Plucky Puffin), amd64 |
| Resources | 3 CPUs, 4096 MB RAM, 30.6 GB disk |
| Network | NAT |
| Snapshot | Taken before running the shipped package |

## **2.3 Method**

Each installer run used the command below, so that every run left a log for later analysis.

time ./install-eSim.sh \--install 2\>&1 \| tee \<logfile\>

**Caveat:** KiCad 8.0 was already installed from earlier partial attempts, so not every run started from a pristine VM.

# **3\. Summary of Results**

Seven installer issues were reported (\#1 to \#7). Four were fixed, committed and pushed. Two were patched but not tested end to end. One was reported only. A separate environment problem (\#0, a DNS failure in the VM) was fixed locally and is not an eSim defect.

| Outcome | Count | Issues |
| ----- | :---: | ----- |
| Installer issues reported | 7 | \#1 to \#7 |
| Fixed, committed and pushed | 4 | \#1, \#2, \#3, \#4 |
| Patched, not tested end to end | 2 | \#5, \#6 |
| Reported only | 1 | \#7 |
| Environment issue (VM, not eSim), fixed locally | 1 | \#0 |
| Documentation mismatch | 1 | Branch name |

## **3.1 Issue Overview**

| \# | Issue | Difficulty | Status |
| :---: | ----- | :---: | ----- |
| 0 | VM DNS failure (empty /etc/resolv.conf, systemd-resolved crash). Environment, not eSim. | Low | Fixed locally |
| 1 | Unclosed if in run\_version\_script() (installers branch) | Low | Fixed, pushed |
| 2 | Duplicate config block in the dispatcher writes to / | Low to medium | Fixed, pushed |
| 3 | Version regex needs three numbers, so the version is empty on 25.04 | Low | Fixed, pushed |
| 4 | exit 0 after "KiCad 8.0 is already installed" ends the whole installer (edge case) | Low | Fixed, pushed (limited verification) |
| 5 | NGHDL installer has the same regex bug | Low | Patched, untested |
| 6 | nghdl.zip has no 25.04 script; NGHDL dispatcher has no 25.04 case | Medium to high | Workaround committed, untested |
| 7 | Hard-coded patch release 22.04.4 in the NGHDL dispatcher | Low | Reported only |

# **4\. Procedure Followed**

1. **Prepared the VM.** apt could not resolve host names. The DNS fault was diagnosed and fixed (Issue 0), then git 2.48.1 was installed.

2. **Forked and cloned.** Forked FOSSEE/eSim, cloned the fork, and added the original repository as upstream. The task refers to an "installer branch", but git ls-remote \--heads upstream shows it is named installers. A local branch was created from upstream/installers.

3. **Inspected the Ubuntu/ directory.** It holds install-eSim.sh, install-eSim-scripts/ (22.04, 23.04, 24.04, 25.04), REPORT.md and READMEs.

4. **First attempts.** bash \-n on the original install-eSim.sh failed (Issue 1). After that fix, the config-writing lines produced seven "Is a directory" errors (Issue 2).

5. **Downloaded the official package** (eSim-2.5.zip) and checked its SHA256 against esim.fossee.in/downloads. It contains install-eSim.sh, nghdl.zip, library/kicadLibrary.tar.xz and install-eSim-scripts/ with only 22.04, 23.04 and 24.04.

6. **Baseline run of the shipped script.** Result: "Unsupported Ubuntu version: 25.04 ()" (explained by Issue 3).

7. **Run with the installers-branch scripts and the fixes** copied over the package. PyQt5 installed into the virtual environment. The KiCad step then reported that KiCad 8.0 was already installed and the script ended with a success code (Issue 4). After that fix, the installer finished the KiCad step, extracted nghdl.zip, and the NGHDL installer stopped with "Unsupported Ubuntu version: 25.04 ()" (Issues 5 and 6).

***Usability note (not counted as an issue):** running the installer directly from a git checkout of the installers branch fails with "tar: library/kicadLibrary.tar.xz: Cannot open". The installer expects the packaged layout (library/kicadLibrary.tar.xz and nghdl.zip next to the script), and the error message does not say so.*

# **5\. Issues in Detail**

## **Issue 0: DNS Failure Inside the VM (environment, not eSim)**

| Symptom | apt failed with "Temporary failure resolving 'in.archive.ubuntu.com'". |
| :---- | :---- |
| **Diagnosis** | ping 8.8.8.8 worked but ping google.com failed, so routing was fine and only name resolution was broken. resolvectl status showed "resolv.conf mode: foreign" and /etc/resolv.conf was a 0-byte file. Crash reports for systemd-resolved (and journald, logind, oomd, udevadm) were present in /var/crash. sudo also warned "unable to resolve host" because the hostname was missing from /etc/hosts. |
| **Fix** | Relinked /etc/resolv.conf to /run/systemd/resolve/stub-resolv.conf, restarted systemd-resolved, and added "127.0.1.1 \<hostname\>" to /etc/hosts. |
| **Status** | Fixed locally |

## **Issue 1: Unclosed if in run\_version\_script()**

| Symptom | The dispatcher aborts before installing anything. |
| :---- | :---- |
| **Reproduce** | git show upstream/installers:Ubuntu/install-eSim.sh \> /tmp/orig.sh && bash \-n /tmp/orig.sh |
| **Evidence** | line 70: syntax error near unexpected token '}' |
| **Root cause** | if \[\[ \-f "\$SCRIPT" \]\]; then has no matching fi. |
| **Fix** | Added fi directly after bash "\$SCRIPT" "\$ARGUMENT". bash \-n now passes. |
| **Note** | The official 2.5 zip does not have this syntax error; it is present in the installers branch version. |
| **Commit** | [8aa94709](https://github.com/Bhuvi16-sys/eSim/commit/8aa94709) |

## **Issue 2: Duplicate Config Block Writes to /**

| Symptom | After the version installer finished, seven errors appeared: "./install-eSim.sh: line 63: /: Is a directory" (lines 63 to 69). |
| :---- | :---- |
| **Root cause** | Lines 63 to 69 of install-eSim.sh are echo ... \>\> \$config\_dir/\$config\_file. Those variables are defined only inside install-eSim-25.04.sh, which runs as a separate bash process, so in the dispatcher they are empty and every redirect targets /. The version installer already writes config.ini itself (its lines 45 to 56). git blame shows the block is legacy code from 2020 left in the dispatcher. |
| **Fix** | Removed the seven redundant lines. |
| **Verification** | The same code path (child exits 0, dispatcher continues) produced the seven errors before the change and none after it. |
| **Commit** | [5d43c8c4](https://github.com/Bhuvi16-sys/eSim/commit/5d43c8c4) |

## **Issue 3: Version Regex Requires Three Numbers**

| Symptom | "Detected Ubuntu Version:" prints nothing and the error message ends in an empty (). |
| :---- | :---- |
| **Reproduce** | lsb\_release \-d prints "Ubuntu 25.04"; lsb\_release \-d \| grep \-oP '\\d+\\.\\d+\\.\\d+' finds no match (exit code 1). |
| **Root cause** | The pattern needs a patch number (such as 22.04.5), which 25.04 does not have. |
| **Fix** | Use \\d+\\.\\d+(\\.\\d+)?, which returns 25.04 with exit code 0\. |
| **Commit** | [69021322](https://github.com/Bhuvi16-sys/eSim/commit/69021322) |

## **Issue 4: exit 0 After "KiCad 8.0 is Already Installed"**

| Symptom | A run ended right after "KiCad 8.0 is already installed." with no error and a success exit code, skipping the KiCad library, NGHDL and the remaining steps. |
| :---- | :---- |
| **Location** | install-eSim-scripts/install-eSim-25.04.sh, line 136, inside the KiCad version check. |
| **Root cause** | exit 0 terminates the entire script, not only the KiCad step. |
| **Fix** | Replaced with a no-op (true) so the script continues. Commit message: "fix(Ubuntu): continue installation when KiCad 8.0 is already installed". |
| **Verification limits** | The stop was observed in one run, before the change. In a later run the install proceeded past KiCad, but KiCad had meanwhile been reinstalled from the PPA, so that run did not necessarily use the changed branch. The change is verified by bash \-n and code reading only. |
| **Scope** | Reached only when KiCad 8.0 is already installed (for example when re-running after a partial install), not on a clean first run. |

## **Issue 5: NGHDL Installer Has the Same Regex Bug**

| Symptom | "Detected Ubuntu Version:" is blank and "Unsupported Ubuntu version: 25.04 ()" appears during the NGHDL step. |
| :---- | :---- |
| **Location** | nghdl/install-nghdl.sh (inside nghdl.zip), line 26\. |
| **Fix** | Same pattern change as Issue 3, committed as Ubuntu/nghdl/install-nghdl.sh. |
| **Status** | Patched, not tested end to end. |
| **Commit** | [eb3fe081](https://github.com/Bhuvi16-sys/eSim/commit/eb3fe081) |

## **Issue 6: No 25.04 Script in nghdl.zip**

| Symptom | After the KiCad step the installer extracts nghdl.zip and then stops with "Unsupported Ubuntu version". nghdl/install-nghdl-scripts/ contains only the 22.04, 23.04 and 24.04 scripts. |
| :---- | :---- |
| **Workaround** | The "24.04" case now also matches "25.04" and uses the 24.04 script (same commit as Issue 5). |
| **Risk** | Ubuntu/REPORT.md records that GHDL does not support LLVM 20, so the 24.04 script may fail later in the GHDL build. It was not run. It was also not confirmed whether the main eSim files are installed when NGHDL aborts the run. |
| **Status** | Workaround committed, untested. |

## **Issue 7: Hard-coded Patch Release in the NGHDL Dispatcher**

| Observation | nghdl/install-nghdl.sh line 37 compares FULL\_VERSION to the exact string "22.04.4". Other 22.04 patch releases fall through to the 23.04 script. |
| :---- | :---- |
| **Status** | Reported only. |

## **Documentation Mismatch**

The task text says "Checkout to the installer branch", but the branch is named installers.

# **6\. Known Upstream Problems (Not Reproduced)**

The existing Ubuntu/REPORT.md on the installers branch documents problems that this task also hints at. They are listed for completeness and were not re-tested here.

* GHDL does not support LLVM 20 (the default on 25.04). The documented fix is LLVM 18 with the matching llvm-config path.

* libcanberra-gtk-module is no longer available on 25.04.

# **7\. Limitations**

* No complete end-to-end install and no eSim launch on 25.04 yet.

* The NGHDL changes (Issues 5 and 6\) are untested.

* The log line ".config/kicad/6.0 does not exist" appeared while the installed KiCad is 8.0, and the script copies sym-lib-table to a hard-coded \~/.config/kicad/6.0/. It was not verified whether the eSim symbol libraries register in KiCad 8\.

* The VM was not pristine for every run (KiCad pre-installed).

# **8\. Recommendations**

1. Add a real install-nghdl-25.04.sh that builds GHDL against LLVM 18, following Ubuntu/REPORT.md, and package it into nghdl.zip.

2. Make an NGHDL failure produce a clear warning instead of ending the installation silently.

3. Replace the hard-coded KiCad 6.0 config path with the installed major version.

4. Use \\d+\\.\\d+(\\.\\d+)? for version detection everywhere.

5. Add a pre-check that tells the user when the installer is run outside the packaged folder layout.

# **9\. Conclusion**

The installation of eSim 2.5 on Ubuntu 25.04 failed at several independent points: a syntax error, a leftover configuration block, a version-detection regex, an early exit, and the absence of a 25.04 NGHDL path. Four of these were fixed and committed with a clear root cause for each. Two NGHDL patches and one reported item remain unverified.

The main blocker for a complete installation is the missing NGHDL/GHDL script for 25.04, which needs an LLVM 18 build. Until that is added and tested, a full install on this release cannot be claimed.

# **Appendix A: Repository Guide**

| Path | Contents |
| ----- | ----- |
| TASK4\_REPORT.md | Original working report |
| Ubuntu/install-eSim.sh | Dispatcher (fixes \#1, \#2, \#3) |
| Ubuntu/install-eSim-scripts/install-eSim-25.04.sh | Version installer (fix \#4) |
| Ubuntu/nghdl/install-nghdl.sh | NGHDL dispatcher copy with the \#5 and \#6 patches (untested) |
| task4-evidence/ | Run logs: shipped-package baseline and fixed runs |

Fork: [github.com/Bhuvi16-sys/eSim](https://github.com/Bhuvi16-sys/eSim) (branch installers)

# **Appendix B: Commits**

| Commit | Change |
| ----- | ----- |
| [8aa94709](https://github.com/Bhuvi16-sys/eSim/commit/8aa94709) | Close unclosed if in run\_version\_script |
| [5d43c8c4](https://github.com/Bhuvi16-sys/eSim/commit/5d43c8c4) | Remove duplicate config.ini block from dispatcher |
| [69021322](https://github.com/Bhuvi16-sys/eSim/commit/69021322) | Detect two-part Ubuntu versions like 25.04 |
| \[add hash\] | Continue installation when KiCad 8.0 is already installed |
| [eb3fe081](https://github.com/Bhuvi16-sys/eSim/commit/eb3fe081) | NGHDL dispatcher: two-part versions, route 25.04 to the 24.04 script (untested) |

# **Appendix C: Mapping to Evaluation Criteria**

| Criterion | Where to look |
| ----- | ----- |
| Quality of documentation | This report; task4-evidence/ logs; Sections 4 and 5 |
| Number of issues reported | Section 3.1 and Section 5 |
| Number of issues fixed | Section 3.1, with commit links in Appendix B |
| Difficulty of fixes | Difficulty column in Section 3.1, with verification limits stated per fix |

[image1]: <data:image/png;base64,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> "iitb.png"

[image2]: <data:image/png;base64,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> "fossee.png"