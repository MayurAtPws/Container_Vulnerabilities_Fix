# Container Vulnerabilities & Best Practices

### *KUBERNETES VULNS*
## 1) Process can elevate its own privileges (KSV001) - Severity: MEDIUM

### Problem : 
A malicious process could gain root access or other elevated privileges within the container, potentially leading to a compromise of the entire node.

### Fix:

Modify the container's security context to drop all unnecessary capabilities and ensure privilege escalation is not allowed:

    securityContext:
        allowPrivilegeEscalation: false
        capabilities:
            drop:
            - ALL


## 2) Default capabilities not dropped (KSV003) - Severity: LOW

### Problem : 
Kubernetes containers have a set of default capabilities. If these are not explicitly dropped, the container might have more privileges than necessary.
An attacker might exploit these additional capabilities to escalate privileges or perform unauthorized actions within the container.


### Fix:
Explicitly drop all capabilities and add back only those that are necessary.

    securityContext:
    capabilities:
        drop:
        - ALL


##  3) Runs as root user (KSV012) - Severity: MEDIUM


### Problem : 
Running containers as the root user within the container can be dangerous, as it grants the process all privileges inside the container.

### Fix:
Define a non-root user in the Dockerfile and set the user in the Kubernetes manifest:
yaml

*Dockerfile*

    # Create a non-root user and group

    RUN groupadd -g 1001 mygroup && \
        useradd -r -u 1001 -g mygroup myuser


*Kubernetes*

    securityContext:
        runAsUser: 1001


## 4) Runs with low user ID (KSV020) - Severity: LOW

### Problem : 
Running with a low user ID (**typically IDs less than 1000**) might be problematic if those IDs are also used by system accounts or if there's a mapping in the host system.
Potential conflicts with host system accounts or unintended privilege escalations.

### Fix:
Configure the security context to use a higher user ID

    securityContext:
    runAsUser: 1000

## 5) Runs with low group ID (KSV021) - Severity: LOW

### Problem : 
Similar to the low user ID, running with a low group ID can pose risks due to conflicts with host system groups.
Potential conflicts with host system group IDs or unintended access privileges.

### Fix:
Configure the security context to use a higher group ID:

    securityContext:
    runAsGroup: 1000


## 6) Root file system is not read-only (KSV014) - Severity: LOW

### Problem : 
If the root file system is writable, applications or attackers can modify the file system, potentially leading to tampering or persistent compromise.

### Fix:
Make the root file system read-only to prevent unauthorized changes.

    securityContext:
    readOnlyRootFilesystem: true

## 7) Default Seccomp profile not set (KSV030) - Severity: LOW

### Problem : 
Seccomp (short for Secure Computing Mode) is a Linux kernel feature that restricts the system calls available to a process 
the default Seccomp profile may expose them to security vulnerabilities associated with unnecessary or dangerous system calls. 

### Fix:
Define a default Seccomp profile for Kubernetes clusters, ensuring that containers run with restricted system call access


        seccompProfile:
            type: RuntimeDefault


## 8) Container capabilities must only include NET_BIND_SERVICE (KSV106) - Severity: LOW

### Problem : 
This vulnerability suggests that container capabilities are not properly configured, and they include capabilities beyond what is necessary for the application to function. Specifically, it recommends that only the NET_BIND_SERVICE capability should be included.

### Fix:
 you can explicitly specify only the **NET_BIND_SERVICE** capability while dropping all other capabilities that are not needed.

    securityContext:
    capabilities:
        add:
        - NET_BIND_SERVICE
    # Optionally, you can also explicitly drop all other capabilities:
    # capabilities:
    #   drop:
    #   - ...


---

## FULL EXAMPLE 

    apiVersion: v1
    kind: Pod
    metadata:
    name: secure-pod
    spec:
    containers:
    - name: my-container
        image: my-image
        securityContext:
        allowPrivilegeEscalation: false
        capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        runAsUser: 1001
        runAsGroup: 1002
        readOnlyRootFilesystem: true
        seccompProfile:
            type: RuntimeDefault


---

### *DOCKER VULNS*

##  1) Ensure a user for the container has been created 
### Problem:
Running containers with a non-root user helps minimize the potential impact of security vulnerabilities.

### Fix:
Ensure that your Dockerfile contains instructions to create a non-root user and run the container process with that user. MAke sure to use a higher user ID.

    RUN groupadd --gid 65536 testgroup \
            && useradd --home-dir /home/test --create-home --uid 65536 \
                --gid 65536 --shell /bin/sh --skel /dev/null test


## 2) Ensure HEALTHCHECK instructions have been added to the container image	

### Problem : 
HEALTHCHECK instructions allow Docker to periodically check the status of a container. This can help detect and handle container failures more efficiently.

### Fix:
Add a HEALTHCHECK instruction to your Dockerfile to specify a command that Docker should run to determine if the container is still functioning correctly.

    HEALTHCHECK --interval=3s --timeout=4s CMD curl -f http://localhost || exit 1

## 3) Ensure update/upgrade instructions are not used in the Dockerfile

### Problem : 
Best Practice: It's generally not recommended to include update/upgrade instructions like apt-get update or yum upgrade within Dockerfiles because they can lead to inconsistencies between images.


### Fix:
Instead of including update/upgrade instructions in the Dockerfile, it's better to ensure that the base image used is up to date and contains the necessary dependencies.

    # ⚠️ Don't have these Update Commands ⚠️
    RUN apt-get update && apt-get upgrade -y


## 4) Ensure COPY is used instead of ADD in Dockerfile

### Problem : 
While both COPY and ADD instructions can be used to copy files into a Docker image, COPY is preferred as it is simpler and more transparent.

### Fix :
Replace any uses of the ADD instruction in your Dockerfile with the COPY instruction.

    # Use COPY instead of ADD
    COPY src/ /app/src/

