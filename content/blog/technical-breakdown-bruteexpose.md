+++
title = "Technical Breakdown: BruteExpose"
date = 2023-04-15
+++

# Live Security Monitor

Project inspired by [Brute.Fail](https://brute.fail/)

## Project Overview

[Live Security Monitor](https://github.com/chomnr/live-security-monitor) (formerly BruteExpose) logs login attempts to OpenSSH servers. It captures:

- Credentials used
- Origin (IP & country)
- Attack protocol
- Timestamp

## Technical Implementation

### Language Choice

I chose Java because I wanted to add a Java project to my portfolio. In retrospect, C or C++ would have been more appropriate for this use case. Java familiarity from Minecraft plugin development made it a comfortable choice.

### Metrics & Analytics System

#### Data Collection

For gathering metrics, I used IPInfo's .mmdb (MaxMind database). This required manual updates to prevent errors. The IPInfo API would have been a better choice.

#### Analytics Processing

I implemented a modular analytics system where statistics can be easily added or removed. The system updates JSON files where metrics are tracked.

Current analytics (as of 6/1/2024):

- NumberOfAttemptsOverTime
- AttackTotalByDayOfWeek
- DistributionOfAttackProtocols
- AttackOriginByCountry
- AttackOriginByIp
- CommonlyTargetedByCredential

#### Implementing Custom Analytics

Here's how to add your own analytics:

**ProtocolBasedMetrics.java**

This file populates values in the JSON file that tracks analytics. Related metrics are grouped into a single function:

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

**DistributionOfAttackProtocols.java**

The actual stat tracking mechanism using a HashMap:

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

**BruteMetricData.java**

Where analytics are instantiated:

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

### Disabling Password Protection

Standard OpenSSH includes a safety mechanism that prevents password leaking through timing attacks. When trying to dump passwords, you'll see:

```
^M^?INCORRECT^@
```

To bypass this, remove the following line in the OpenSSH source code: https://github.com/openssh/openssh-portable/blob/df56a8035d429b2184ee94aaa7e580c1ff67f73a/auth-pam.c#L1198

### Credential Logging

To capture credentials, we need a PAM module to dump them to a text file:

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

### PAM Module Integration

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

This configuration makes the system check `pam_unix.so` first. If successful, it skips the next 2 lines. If not, it hits our module and then the denial module.

## Lessons Learned

What I would do differently:

1. Skip the modular analytics system for a simpler hardcoded approach. The current approach adds complexity for a project that requires an OpenSSH vulnerability to function.

2. Use C for the entire project rather than Java. Using two languages introduces unnecessary complexity.

3. Use the IPInfo API instead of .mmdb files to avoid manual updates.

4. Replace JSON with SQLite for better data handling. JSON has read limitations when file size grows, while SQLite handles both read and write operations efficiently.
