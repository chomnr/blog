+++
title = "technical breakdown: bruteexpose"
date = 2023-04-15
+++

A security monitoring tool that logs login attempts to OpenSSH servers, inspired by [Brute.Fail](https://brute.fail/).

[GitHub Repository](https://github.com/chomnr/live-security-monitor) (formerly BruteExpose)

The system captures credentials used, origin (IP & country), attack protocol, and timestamps from authentication attempts.

## Technical Stack

Built in Java primarily to add a Java project to my portfolio. In hindsight, C or C++ would have been more suitable for this use case. Java familiarity from Minecraft plugin development made it the comfortable choice at the time.

## Metrics & Analytics System

The analytics system uses IPInfo's .mmdb (MaxMind database) for geolocation data. This required manual updates to prevent errors - the IPInfo API would have been better.

The modular analytics system allows easy addition or removal of statistics. All metrics are tracked in JSON files that get updated as new data comes in.

Current analytics include:
- NumberOfAttemptsOverTime
- AttackTotalByDayOfWeek  
- DistributionOfAttackProtocols
- AttackOriginByCountry
- AttackOriginByIp
- CommonlyTargetedByCredential

Adding custom analytics follows a specific pattern. Here's how protocol-based metrics work:

**ProtocolBasedMetrics.java** populates values in the JSON analytics file:

```java
private DistributionOfAttackProtocols distributionOfAttackProtocols
public enum ProtocolBasedType {
  SSH,
  UNKNOWN
}
public ProtocolBasedMetrics() {
  distributionOfAttackProtocols = new DistributionOfAttackProtocols();
}
public DistributionOfAttackProtocols getDistributionOfAttackProtocols() {
  return distributionOfAttackProtocols;
}
public void populate(String name, int amount) {
  getDistributionOfAttackProtocols().insert(name, amount);
}
public void populate(ProtocolBasedType type, int amount) {
  getDistributionOfAttackProtocols().insert(type, amount);
}
public void populate(String type) {
  getDistributionOfAttackProtocols().insert(type, 1);
}
public void populate(ProtocolBasedType type) {
  getDistributionOfAttackProtocols().insert(type, 1);
}
```

**DistributionOfAttackProtocols.java** handles the actual stat tracking with HashMap:

```java
private HashMap protocols = new HashMap<>();
public DistributionOfAttackProtocols() {}
public void insert(String type, int amount) {
    ProtocolBasedType protocolType = getProtocolByName(type);
    addAttempts(protocolType, amount);
}
public void insert(ProtocolBasedType type, int amount) {
    addAttempts(type, amount);
}
private void addAttempts(ProtocolBasedType type, int amount) {
    String protocolName = getNameOfProtocol(type);

    if (protocols.get(protocolName) == null) {
        protocols.put(protocolName, amount);
    } else {
        protocols.put(protocolName, getAttempts(type)+amount);
    }
}
private Integer getAttempts(ProtocolBasedType type) {
    return protocols.get(getNameOfProtocol(type));
}
public ProtocolBasedType getProtocolByName(String protocol) {
    if (protocol.equalsIgnoreCase("sshd")) {
        return ProtocolBasedType.SSH;
    }
    if (protocol.equalsIgnoreCase("ssh")) {
        return ProtocolBasedType.SSH;
    }
    // or return UNKNOWN
    return ProtocolBasedType.UNKNOWN;
}
private String getNameOfProtocol(ProtocolBasedType type) {
    return type.name();
}
```

**BruteMetricData.java** instantiates all analytics components:

```java
private TimeBasedMetrics timeBasedMetrics;
private GeographicMetrics geographicMetrics;
private ProtocolBasedMetrics protocolBasedMetrics;
private CredentialBasedMetrics credentialBasedMetrics;

public BruteMetricData() {
    timeBasedMetrics = new TimeBasedMetrics();
    geographicMetrics = new GeographicMetrics();
    protocolBasedMetrics = new ProtocolBasedMetrics();
    credentialBasedMetrics = new CredentialBasedMetrics();
}

public TimeBasedMetrics getTimeBasedMetrics() {
    return timeBasedMetrics;
}
public GeographicMetrics getGeographicMetrics() { return geographicMetrics; }
public ProtocolBasedMetrics getProtocolBasedMetrics() { return protocolBasedMetrics; }
public CredentialBasedMetrics getCredentialBasedMetrics() { return credentialBasedMetrics; }
```

## OpenSSH Modification

Standard OpenSSH includes safety mechanisms that prevent password leaking through timing attacks. When trying to dump passwords, you'll see:

```
^M^?INCORRECT^@
```

To bypass this protection, remove the following line in the OpenSSH source code: https://github.com/openssh/openssh-portable/blob/df56a8035d429b2184ee94aaa7e580c1ff67f73a/auth-pam.c#L1198

For credential logging, a custom PAM module dumps credentials to a text file:

```c
#include "library.h"
#include <security/pam_appl.h>
#include <security/pam_modules.h>

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BE_LOG_FILE "/var/log/brute_tracker.txt"
#define BE_DELAY 700

PAM_EXTERN int pam_sm_authenticate(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    char    *username,
            *password,
            *protocol,
            *hostname;

    pam_get_item(pamh, PAM_USER, (void*)&username);
    pam_get_item(pamh, PAM_AUTHTOK, (void*)&password);
    pam_get_item(pamh, PAM_RHOST, (void*)&hostname);
    pam_get_item(pamh, PAM_SERVICE, (void*)&protocol);

    // Added a delay to ensure that BruteExpose gets to read the entry.
    // In terms of practicality, I should have wrote the entire program in C,
    // but I am not familiar with the language.
    usleep(BE_DELAY);

    FILE *fd = fopen(BE_LOG_FILE, "a");
    if (fd != NULL) {
        fprintf(fd, "%s %s %s %s \n", username, password, hostname, protocol);
        fclose(fd);
    }

    return PAM_SUCCESS;
}
```

After compiling the PAM module:

1. Place the .pam file in `/lib/x86_64-linux-gnu/security/`
2. Edit common-auth: `sudo nano /etc/pam.d/common-auth`
3. Add `libbe_pam.so` before the password denial:

```
# here are the per-package modules (the "Primary" block)
auth    [success=2 default=ignore]      pam_unix.so nullok
# enable BruteExpose.
auth    optional                        libbe_pam.so
# here's the fallback if no module succeeds
auth    requisite                       pam_deny.so
```

This configuration makes the system check `pam_unix.so` first. If successful, it skips the next 2 lines. If authentication fails, it hits our module and then the denial module.

## What I'd Do Differently

Several design decisions proved suboptimal:

**Skip the modular analytics system** for a simpler hardcoded approach. The current complexity isn't justified for a project that requires an OpenSSH vulnerability to function.

**Use C for the entire project** rather than mixing Java and C. Using two languages introduces unnecessary complexity when C would handle everything more efficiently.

**Use the IPInfo API** instead of .mmdb files to avoid manual database updates and maintenance overhead.

**Replace JSON with SQLite** for better data handling. JSON has read performance issues as file size grows, while SQLite handles both read and write operations efficiently at any scale.