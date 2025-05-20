+++
title = "Technical Breakdown: BruteExpose"
date = 2023-04-15
+++

# Live Security Monitor (Formerly BruteExpose)

_Project inspired by [Brute.Fail](https://brute.fail/)_

## Project Overview

[Live Security Monitor](https://github.com/chomnr/live-security-monitor) tracks login attempts to OpenSSH servers, recording:

- Credentials used
- Origin (IP address & country)
- Attack protocol
- Timestamp of attempt

This security monitoring tool provides valuable insights into potential brute force attacks on your server infrastructure (you really shouldn't because we have to patch OpenSSH to get to actually to work.).

## Technology Choice

I developed this project in Java primarily to diversify my portfolio. In retrospect, C or C++ would have been more suitable languages for this application because they're low level languages. My familiarity with Java from previous Minecraft plugin development made it a comfortable choice, though not necessarily the optimal one.

## Metrics & Analytics System

### Data Collection

For collecting geographic metrics, I implemented IPInfo's MaxMind database (mmdb). While functional, this approach requires regular database updates to maintain accuracy. In retrospect, implementing the IPInfo API would have provided better maintainability and more current data.

### Analytics Processing

The application includes a modular analytics system that processes collected data into meaningful insights. The system is designed for extensibility, allowing easy addition or removal of different statistical metrics that are automatically reflected in the JSON output.

#### Currently Supported Analytics (as of June 1, 2024):

- Number of attempts over time
- Attack totals by day of week
- Distribution of attack protocols
- Attack origins by country
- Attack origins by IP address
- Commonly targeted credentials

### Implementation Example

Below is an example of how analytics are implemented in the system:

#### ProtocolBasedMetrics.java

This class populates protocol-related metrics in the JSON tracking file. Multiple statistics can be processed through a single method since they often correlate:

```java
private DistributionOfAttackProtocols distributionOfAttackProtocols;

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

#### DistributionOfAttackProtocols.java

This class represents a specific tracked statistic using a simple HashMap structure that is processed by the JSON Object Mapper:

```java
private HashMap<String, Integer> protocols = new HashMap<>();

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
        protocols.put(protocolName, getAttempts(type) + amount);
    }
}

private Integer getAttempts(ProtocolBasedType type) {
    return protocols.get(getNameOfProtocol(type));
}

public ProtocolBasedType getProtocolByName(String protocol) {
    if (protocol.equalsIgnoreCase("sshd") || protocol.equalsIgnoreCase("ssh")) {
        return ProtocolBasedType.SSH;
    }
    return ProtocolBasedType.UNKNOWN;
}

private String getNameOfProtocol(ProtocolBasedType type) {
    return type.name();
}
```

#### BruteMetricData.java

This core class instantiates and manages all analytics and metrics components:

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

public GeographicMetrics getGeographicMetrics() {
    return geographicMetrics;
}

public ProtocolBasedMetrics getProtocolBasedMetrics() {
    return protocolBasedMetrics;
}

public CredentialBasedMetrics getCredentialBasedMetrics() {
    return credentialBasedMetrics;
}
```

## OpenSSH Integration

### Modifying OpenSSH

Standard OpenSSH includes a security mechanism that prevents credential leaking by overwriting incorrect password entries with `^M^?INCORRECT^@`. To capture these credentials, you must disable this protection by removing the following line from OpenSSH's source code:

```
https://github.com/openssh/openssh-portable/blob/df56a8035d429b2184ee94aaa7e580c1ff67f73a/auth-pam.c#L1198
```

**Note:** This modification intentionally introduces a security vulnerability to enable monitoring functionality.

### Credential Dumping

The application captures login credentials through a simple PAM module that writes to a text file:

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
    usleep(BE_DELAY);

    FILE *fd = fopen(BE_LOG_FILE, "a");
    if (fd != NULL) {
        fprintf(fd, "%s %s %s %s \n", username, password, hostname, protocol);
        fclose(fd);
    }

    return PAM_SUCCESS;
}
```

The Java application monitors `brute_tracker.txt` for changes, processing new entries as they appear.

### PAM Module Integration

After compiling the PAM module:

1. Copy the `.pam` file to `/lib/x86_64-linux-gnu/security/`
2. Edit the PAM configuration with `sudo nano /etc/pam.d/common-auth`
3. Add the following entry before the deny statement:

```
# here are the per-package modules (the "Primary" block)
auth    [success=2 default=ignore]      pam_unix.so nullok
# enable Live Security Monitor
auth    optional                        libbe_pam.so
# here's the fallback if no module succeeds
auth    requisite                       pam_deny.so
```

This configuration ensures that if `pam_unix.so` authentication succeeds, the next two lines are skipped. Otherwise, our monitoring module captures the credentials before access is denied.

## Lessons Learned

### Architecture Improvements

If I were to rebuild this project, I would make the following changes:

1. **Simplify Analytics**: Replace the modular analytics system with hardcoded solutions for better simplicity and performance. The current design, while flexible, introduces unnecessary complexity for this specific use case.

2. **Language Selection**: Implement the entire solution in C rather than using Java. The multi-language approach creates maintenance challenges and introduces unnecessary complexity.

3. **IP Data Source**: Use the IPInfo API instead of the mmdb database to eliminate the need for manual database updates and ensure more current geolocation data.

4. **Data Storage**: Replace the JSON-based storage with SQLite. JSON becomes inefficient at scale, particularly for reading operations, whereas SQLite would provide better performance for both reading and writing operations.

## Conclusion

While functional, this project serves primarily as a learning experience rather than a production-ready security tool. The intentional OpenSSH vulnerability makes it unsuitable for legitimate security applications, but the development process provided valuable insights into system-level programming, credential monitoring techniques, and data analytics.
