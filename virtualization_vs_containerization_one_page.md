# Virtualization vs Containerization

## One-Page Interview Notes

### 1. Core Difference

  -----------------------------------------------------------------------
  Virtualization                      Containerization
  ----------------------------------- -----------------------------------
  Creates **Virtual Machines (VMs)**  Creates **Containers**

  Each VM has its own **Guest OS**    Containers share the **Host OS
                                      Kernel**

  Uses a **Hypervisor**               Uses a **Container Engine** such as
                                      Docker Engine

  Generally heavier                   Generally lightweight

  VM startup is generally slower      Container startup is generally
                                      faster
  -----------------------------------------------------------------------

### 2. Architecture

**Virtualization**

`Physical Server → Hypervisor → VM → Guest OS → Application`

**Containerization**

`Physical Server → Host OS → Docker Engine → Container → Application`

### 3. Example

-   **AWS EC2:** Common example used to explain Virtual Machines /
    virtualization.
-   **Docker:** Common example used to explain containerization.

### 4. Key Interview Point

If you run multiple applications using VMs, each VM typically requires
its own Guest OS.

With containers, multiple isolated containers can run on the same host
while sharing the host kernel.

**Result:** Containers generally need fewer resources such as **CPU,
memory, and storage**, and usually start faster than VMs.

### 5. VM vs Container --- Quick Recall

-   **VM → Hardware virtualization**
-   **Container → OS-level virtualization**
-   **VM → Guest OS required**
-   **Container → Guest OS not required**
-   **VM → Hypervisor**
-   **Container → Container Engine**
-   **VM → Higher overhead**
-   **Container → Lower overhead**

### 6. Interview Question

**Q: Why are containers more lightweight than VMs?**

**Answer:** A VM includes a complete Guest OS for each virtual machine.
Containers share the host operating system's kernel, so they don't need
a separate Guest OS, reducing overhead and allowing faster startup.

> **Note:** Docker containers are not simply "mini VMs." They provide
> process-level isolation while sharing the host kernel.
