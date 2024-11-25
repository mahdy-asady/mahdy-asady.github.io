---
title: OpenWrt init script
description: In this article, we explore OpenWrt’s init system and how it manages services on embedded devices. We cover the structure of init scripts, key functions like start, stop, and restart, and introduce the procd process manager for handling service dependencies. Whether you're new to OpenWrt or looking to create custom init scripts, this guide provides essential insights and best practices.
date: 2024-10-05 15:00:00 +0330
categories: [Programming, Embedded Linux]
tags: [openwrt, embedded-linux]
---
## Introduction to OpenWrt

OpenWrt is an open-source, Linux-based operating system tailored for embedded devices, primarily routers. It is widely used in the networking community due to its flexibility, stability, and ability to extend the functionality of consumer routers far beyond the manufacturer’s defaults. With OpenWrt, users can install custom software, fine-tune performance, and even use the device for advanced networking purposes such as VPN, firewalling, and traffic shaping.

One of the key strengths of OpenWrt is its ability to manage network services and daemons through init scripts, which we will explore in detail.

## OpenWrt Boot Process Flow

The OpenWRT boot process follows a structured flow that begins with the kernel's execution of the initial process, typically located at `/init`, which quickly transitions to `/sbin/init`. From there, the system runs the **preinit** script (`/etc/preinit`), responsible for early setup tasks such as configuring the root filesystem and entering failsafe mode if necessary. Once the preinit tasks are complete, the modern process manager, **Procd** (`/sbin/procd`), takes over. Procd is responsible for managing all services, handling system events, and ensuring service dependencies are met. Finally, Procd executes the startup scripts located in `/etc/rc.d/`, bringing the system and services fully online in the correct order. This structured flow ensures that the system initializes efficiently, with the ability to handle complex service management and recovery.

## OpenWrt Init System Overview

Like many Linux-based systems, OpenWrt uses an init system to manage the startup, shutdown, and maintenance of services. However, instead of using more common systems like systemd or sysvinit, OpenWrt relies on its own lightweight init system tailored for embedded environments.

The core init system in OpenWrt revolves around shell scripts that define how services are started, stopped, restarted, and reloaded. This init system also leverages the procd process manager, which is highly optimized for handling processes and events on resource-constrained devices like routers.

## Structure of OpenWrt Init Scripts

The init scripts in OpenWrt reside in the `/etc/init.d/` directory. Each script in this directory is responsible for managing the lifecycle of a specific service, following a standardized structure that details how services are started, stopped, restarted, and reloaded. Here’s an example of how the init scripts are organized:

```sh
/etc/init.d/
├── dnsmasq
├── network
├── uhttpd
└── firewall
```

Each script contains shell code that specifies how to handle service lifecycle events, such as starting and stopping the service.

### What Happens When You Enable a Service?

Each OpenWrt init script can define separate priorities for starting and stopping services using the **START** and **STOP** variables (we will explain them later). These priorities are critical for controlling the order in which services are started at boot or stopped during shutdown.

When you enable a service using the command `/etc/init.d/<service_name> enable`, two symbolic links are created in the `/etc/rc.d/` directory (Do you remember this directory from [OpenWrt Boot Process Flow](#openwrt-boot-process-flow)), pointing to the init script in `/etc/init.d/`. The symbolic link follows a specific naming convention based on the **START** and **STOP** priorities defined in the script. For example, enabling the firewall service might create symlinks like this:

```sh
/etc/rc.d/S50firewall -> /etc/init.d/firewall
/etc/rc.d/K50firewall -> /etc/init.d/firewall
```

Here:
- The S (Start) prefix indicates that the start function of the script will be called at boot time.
- The K (Kill) prefix signifies that the stop function of the script will be executed during shutdown to stop the service.
- The 50 refers to the service’s start or stop priority, meaning it will start after services with a lower number for start time and it will stop before services with a lower number for stop time.

When the system boots, OpenWrt runs the **S** links in `/etc/rc.d/` in ascending order of their **START** priority. The system passes the start argument to these scripts to start each service.

During the shutdown process, the system processes these **K** links in descending order (from the highest number to the lowest), ensuring that services are stopped in the reverse order of how they were started.


For example, consider the following structure:

```sh
/etc/rc.d/
├── S10network -> /etc/init.d/network       # The network service is started first
├── S50firewall -> /etc/init.d/firewall     # The firewall service is started next
├── S80uhttpd -> /etc/init.d/uhttpd         # The web server is started last
```

During shutdown, the services will be stopped as follows:

```sh

K80uhttpd -> /etc/init.d/uhttpd stop        # The web server is stopped first
K50firewall -> /etc/init.d/firewall stop    # The firewall is stopped next
K10network -> /etc/init.d/network stop      # The network service is stopped last
```

By carefully defining the **START** and **STOP** values, you can control the precise order in which services start and stop, ensuring proper dependency management between them.

## Understanding the Init Script Format

In OpenWRT, there are two main types of init scripts used to manage services: **legacy init scripts** and **Procd-based init scripts**. Understanding the differences between them is important for knowing when to use each one, as they serve different purposes based on service complexity, resource requirements, and management capabilities.

At its core, an OpenWrt init script consists of two key components:

- **START/STOP priority**: Each service script in OpenWrt defines its startup and shutdown priorities using the **START** and **STOP** variables. These values determine the order in which services are started during boot and stopped during shutdown.

    - Services with a lower START value are started earlier during boot. For example, S10network will start before S50firewall.
    - Conversely, services with a higher STOP value are stopped earlier during shutdown, ensuring dependencies are properly handled.

    > Priorities range from 1 to 99, and care must be taken to avoid gaps or overlaps that could lead to unpredictable behavior.
    {: .prompt-tip }




- **Control functions**: In OpenWRT, init scripts can use a variety of functions to control services, manage system processes, and handle dependencies. These functions differ between the legacy init system and the modern procd-based init system.

### 1. Legacy Init Scripts

Legacy init scripts are traditional shell scripts that handle service management entirely through shell commands. These scripts manually define how to start, stop, and restart services and are not managed by the modern process manager, **Procd**.

#### Structure of Legacy Init Scripts

Legacy init scripts follow a typical format:

They typically contain the `start()`, `stop()`, and `restart()` functions, as well as any custom logic for checking the status of the service.

Here’s a basic example of a legacy init script for a service:

```sh

#!/bin/sh /etc/rc.common

START=99
STOP=10

start() {
    echo "Starting legacy service..."
    /usr/sbin/legacy_service start
}

stop() {
    echo "Stopping legacy service..."
    /usr/sbin/legacy_service stop
}

restart() {
    stop
    start
}
```

#### Legacy Init System Functions

Legacy init scripts in OpenWRT were more manual and closely resembled traditional UNIX-style init scripts. These are functions commonly found in legacy `/etc/init.d/` scripts:

1. **start()**:

    **Purpose**: This function is responsible for starting the service.

    **Usage Example**:

    ```sh
    start() {
        echo "Starting MyService..."
        /usr/sbin/my_service &
        echo $! > /var/run/my_service.pid
    }
    ```

2. **stop()**:

    **Purpose**: Stops the service.

    **Usage Example**

    ```sh
    stop() {
        echo "Stopping MyService..."
        kill $(cat /var/run/my_service.pid) || true
        rm -f /var/run/my_service.pid
    }
    ```

3. **restart()**:

    **Purpose**: Stops and then starts the service again.

    **Usage Example**:

    ```sh
    restart() {
        stop
        start
    }
    ```

4. **reload()**:

    **Purpose**: Reloads the service's configuration without restarting the service (if supported by the application).

    **Usage Example**:

    ```sh
    reload() {
        echo "Reloading MyService configuration..."
        kill -HUP $(cat /var/run/my_service.pid)
    }
    ```

5. **enable()**:

    **Purpose**: Enables the service to start at boot by creating symlinks in /etc/rc.d/. 

    **Usage Example**:

    ```sh
    enable() {
        ln -s ../init.d/my_service /etc/rc.d/S99my_service
        echo "MyService enabled at boot."
    }
    ```

    > Note that this function has a default implementation that will create the symlink. If you need to do any additional work, then create the function in your init script and make the symlink yourself.
    {: .prompt-tip }

6. **disable()**:

    **Purpose**: Disables the service from starting at boot by removing symlinks from /etc/rc.d/.

    **Usage Example**:

    ```sh
    disable() {
        rm -f /etc/rc.d/S99my_service
        echo "MyService disabled at boot."
    }
    ```

    > Note that this function has a default implementation that will remove the symlink. If you need additional works during disabling your service, then create the function in your init script file and mind to remove the symlink yourself.
    {: .prompt-tip }

7. **boot()**:

    **Purpose**: Performs actions specifically related to the boot phase (not commonly used).

    **Usage Example**:

    ```sh
    boot() {
        echo "Running boot tasks for MyService..."
        # Any boot-specific commands here
    }
    ```
8. **Custom commands**:

    **Purpose**: You can add your own custom commands by using the `EXTRA_COMMANDS` variable (Use space to separate multiple commands), and provide help for those commands with the `EXTRA_HELP` variable (Add multiple lines with `\n` for multiple commands), then add sections for each of your custom commands: 

    **Usage Example**:

    ```sh
    EXTRA_COMMANDS="status"
    EXTRA_HELP="        status  Show the current status of the service."

    status() {
        if [ -f /var/run/my_service.pid ] && kill -0 $(cat /var/run/my_service.pid); then
            echo "MyService is running."
        else
            echo "MyService is stopped."
        fi
    }
    ```

### 2. Procd-based Init Scripts

**Procd** is OpenWRT’s lightweight process manager designed to handle the lifecycle of services in a more dynamic and efficient way. Procd-based init scripts leverage Procd for process management, leaving the script with less complexity while offering more advanced features.

OpenWrt’s procd is a critical component of the init system. It not only manages the lifecycle of daemons and services but also handles hotplug events, network configurations, and other system processes.

#### Structure of Procd-based Init Scripts

Procd init scripts are also placed in **/etc/init.d/**, but they delegate much of the service management (e.g., starting, stopping, respawning) to Procd.

The directive `USE_PROCD=1` tells OpenWrt to leverage the procd process manager for handling service dependencies and monitoring.

Here’s an example of a Procd-based init script:

```sh

#!/bin/sh /etc/rc.common

USE_PROCD=1
START=95
STOP=05

PROG=/usr/sbin/procd_service

start_service() {
    procd_open_instance
    procd_set_param command $PROG --option1 value1 --option2 value2
    procd_set_param respawn     # Automatically restart service if it crashes
    procd_set_param stdout 1    # Redirect standard output to system log
    procd_set_param stderr 1
    procd_close_instance
}

stop_service() {
    # No need to manually stop, Procd handles it
    :
}
```

By using procd, OpenWrt ensures that services are restarted if they crash, and it enables more efficient memory and process management.

This example tells procd to start procd_service, and the respawn directive ensures that the service is automatically restarted if it fails.

#### Procd Init System Functions

In a procd-based script (located in `/etc/init.d/`), you will commonly use the following functions to interact with the system:

1. **start_service()**:
    This function starts the service and registers it with procd. It's where the configuration for the service is set.

    **Example**:

    ```sh

    start_service() {
        procd_open_instance
        procd_set_param command /usr/sbin/my_service
        procd_set_param respawn
        procd_close_instance
    }
    ```

2. **stop_service()**:

    Defines how to stop the service. This function is typically simple, as Procd handles many things like killing processes.

    **Example**:

    ```sh

    stop_service() {
        killall my_service
    }
    ```

3. **service_triggers()**:

    This function is used to define conditions or events that will trigger the execution of a particular service. Procd monitors these triggers and automatically restarts, reloads, or stops the service when the specified conditions occur.
    
    **Common Trigger Types**

    - **Hotplug Events**: These are system events triggered by hardware changes or dynamic module loading, such as connecting or disconnecting USB devices or network interfaces coming up or going down.
    - **Dependent Service Restarts**: This ensures that when a related or dependent service is restarted, your service also adjusts its state accordingly.
    - **Configuration File Changes**: Procd watches specific configuration files for changes, triggering a reload or restart of the service if needed.

    **Example**:

    ```sh
    service_triggers() {
        # Monitor configuration file changes
        procd_add_reload_trigger "/etc/config/myconfig"

        # Trigger on hotplug events
        procd_add_hotplug_trigger "usb" # Trigger on USB hotplug events

        # Trigger when a dependent service restarts
        procd_add_service_trigger "restart" "network"
    }
    ```

    In this case, Procd will trigger the service when it detects a USB device event. You can further filter events in your script to ensure they only apply to printers.

    If your service relies on the network service, it should restart or reload whenever the network is restarted. You can set this dependency as above. This ensures that your service automatically synchronizes with changes in the network configuration.
    
    Many services rely on configuration files to operate correctly. For example, a firewall service might rely on /etc/config/firewall. Adding this as a reload trigger ensures the service reloads its rules whenever the file is updated.

4. **reload_service() (optional)**:

    This function can be defined to reload the service without stopping it, typically by sending a signal or triggering a configuration reload.

    **Example**:

    ```sh

        reload_service() {
            kill -HUP $(pidof my_service)
        }
    ```

5. **Custom commands**:

    **Purpose**: You can add your own custom commands by calling the `extra_command` with two arguments (name of the command and help of it) for each custom command, then adding sections for each of your custom commands: 

    **Usage Example**:

    ```sh
    status() {
        if pgrep -f "my_service" > /dev/null; then
            echo "MyService is running."
            return 0
        else
            echo "MyService is stopped."
            return 1
        fi
    }

    extra_command "status" "Check service status"
    ```
#### Characteristics of Procd-based Init Scripts

- **Managed by Procd**: Procd-based scripts hand off most of the heavy lifting to Procd. This includes starting and stopping services, logging, handling standard output and error streams, and respawning services if they crash. OpenWrt’s init system handles services (often daemons) by starting them in the background and monitoring their status. These daemons are typically programs that run continuously in the background, performing tasks like routing, DNS resolution, or firewall management.
- **Advanced Features**: Procd provides features such as:
    - **Service Respawn**: Automatically restarts services if they crash. To manage daemons effectively, OpenWrt relies on procd to monitor processes and ensure they run smoothly. The use of procd for this task makes it easier to handle complex service dependencies and recover from failures automatically.
    - **Dependency Management**: Procd can ensure services start in the correct order based on dependencies.
    - **Resource Efficiency**: Procd scripts tend to use fewer system resources since Procd optimizes the management of service processes.
- **Simplified Scripts**: The script itself is smaller and easier to maintain since it doesn’t need to explicitly handle all aspects of the service’s lifecycle.

## Custom Commands, Why?
Custom commands allow developers to enhance the flexibility and control of services running on OpenWrt, offering more specialized and granular management options.

- **Debugging Purposes**: Custom commands can provide tools for diagnosing service issues without needing external utilities or manual interventions.
- **Runtime Configuration**: Sometimes, a service needs fine-tuning or on-the-fly adjustments without requiring a full restart or reload.
- **Testing Service Components**: Developers can use commands to isolate and test specific parts of a service or script during development or troubleshooting.
- **Service Control Beyond Start/Stop/Restart**: Standard Procd scripts typically only offer basic control operations. Custom commands let you introduce specialized behavior tailored to your service.



### **Available Utility Functions in Init Scripts**

Look at the contents of the `/etc/rc.common` file (included at the beginning of init scripts) and you will see `functions.sh` included there. There are a number of utility functions in this file which simplify the development of init scripts and configuration management tools. These functions abstract common tasks like managing configurations, handling users and groups, and iterating over system states, ensuring consistency and efficiency in script execution. Below are two key groups of functions available in this library:
- **Configuration Handling**
    These functions handle the creation and management of configuration sections and options. The functions allow scripts to interact with configuration files, dynamically set or retrieve options, and iterate over configuration sections or lists. They include callback mechanisms for extensibility and are indispensable for maintaining structured configuration data in OpenWRT.

- **User and Group Management**
    These utilities manage the addition, existence checks, and association of users and groups on the system. They ensure that user accounts and permissions are set up correctly during package installation or system initialization. For instance, they can add a user, assign a group ID, and associate the user with a specific group in one seamless process.

This library provides powerful tools for developers working on init scripts, making complex tasks like configuration parsing and user management straightforward and consistent with OpenWRT’s design principles.


## Write Your Own Init Scripts

If you’re developing a custom service or modifying an existing one, writing an init script is straightforward. Here's a basic step-by-step guide to writing your own init script:

- Create a script in /etc/init.d/:

```sh

touch /etc/init.d/my_custom_service
chmod +x /etc/init.d/my_custom_service
```
- Write the script: Start by following the basic format outlined earlier.

- Enable the service: Use the following command to enable your new service:

```sh

/etc/init.d/my_custom_service enable
```
- Start the service:

```sh

    /etc/init.d/my_custom_service start
```

This process allows you to manage the lifecycle of your custom services just like any other built-in service.

## Debugging OpenWrt Init Scripts

For debugging init scripts consider the following steps:

- **Correct executable permission**: Ensure that your init script has executable permission by running `chmod +x` on your file.

- **Validate script syntax**: Use `sh -n /etc/init.d/<script>` to validate your script and find problems.

- **Using the logger command**: This sends messages to the system log, which can be viewed later.

    ```sh

    logger "Starting my_service"
    ```
- **Checking system logs**: You can view the logs using the `logread` command to check the output of your init scripts and debug any issues.

- **Manual execution**: You can run the init scripts manually to see their output in real-time, which can help isolate problems.

## Best Practices for OpenWrt Init Scripts

When writing or modifying OpenWrt init scripts, consider the following best practices:

- **Keep scripts lightweight**: OpenWrt runs on resource-constrained devices, so make sure your scripts are efficient and avoid unnecessary overhead.
- **Leverage procd**: Using procd ensures that your services are properly monitored and restarted if needed.
- **Use meaningful logging**: Log relevant information to help with future debugging.
- **Test thoroughly**: Run your scripts manually during development to ensure they behave as expected in different scenarios.

By following these practices, you can ensure that your OpenWrt init scripts are efficient, maintainable, and robust.
