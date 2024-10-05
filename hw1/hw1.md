# Assignment I - Compiling a Custom Linux Kernel & Adding New System Calls
TA's Hackmd Link: https://hackmd.io/@-ZZtFnnqSZ2F1A-Uy-GMlw/HJJj4BcKC

## How to write the system call

### 1. Download and Prepare the Linux Kernel Source
follow the step in TA's note to clone Linux Kernel Source

### 2. System Call Implementation
Because I don't want to mess up the file structure of linux source code, so I create a `hw` directory under linux folder

Write a c code `revstr.c` and `Makefile` in `hw` folder.
We add this line into `Makefile`:
```
obj-y += revstr.o
```
We can find following line in `Makefile` under Linux folder:
```
core-y          +=
```
Add `hw/` at the end of line 
```
core-y          += hw/
```
- Implementaion of `revstr.c`:
```c++
#include <linux/kernel.h>
#include <linux/syscalls.h>
#include <linux/uaccess.h>  // for copy_from_user, copy_to_user
#include <linux/slab.h>     // for kmalloc, kfree

SYSCALL_DEFINE2(revstr, char __user *, user_str, size_t, len) {
    char *kstr;     // kernel space string
    size_t i, j;
    char temp;      // temporary variable for swapping characters

    // Check if the length is valid
    if (len <= 0) {
        printk("Invalid string length.\n");
        return -EINVAL;  // Invalid argument error
    }

    // Allocate memory for the string in kernel space
    kstr = kmalloc(len + 1, GFP_KERNEL);
    
    if (!kstr) {
        printk("Memory allocation failed.\n");
        return -ENOMEM;  // Memory allocation error
    }

    // Copy the string from user space to kernel space
    if (copy_from_user(kstr, user_str, len)) {
        printk("Failed to copy string from user space.\n");
        kfree(kstr);
        return -EFAULT;  // Bad address error
    }

    // Null-terminate the string
    kstr[len] = '\0';

    // Print the original string to the kernel log
    printk("The original string: %s\n", kstr);

    // Reverse the string in place
    for (i = 0, j = len - 1; i < j; i++, j--) {
        temp = kstr[i];
        kstr[i] = kstr[j];
        kstr[j] = temp;
    }

    // Print the reversed string to the kernel log
    printk("The reversed string: %s\n", kstr);

    // Copy the reversed string back to user space
    if (copy_to_user(user_str, kstr, len)) {
        printk("Failed to copy reversed string to user space.\n");
        kfree(kstr);
        return -EFAULT;  // Bad address error
    }

    // Free the allocated kernel memory
    kfree(kstr);

    return 0;  // Success
}
```

### 3. Modify the System Call Table
The first thing is register our system call function to system call table

The system call table maps system calls to their respective implementation functions. Depending on the architecture, the system call table file will vary. For x86, the file is located in:
```
linux/arch/x86/ syscalls /syscall_64.tbl
```
write the following code into table file
```c++
451  common  revstr  sys_revstr
```
- `451` is the next available system call number.
- `common` refers to the ABI (for standard Linux).
- `revstr` is the name of the system call that the user-space will use.
- `sys_revstr` is the name of the function kernel will call.

### 4. Declare the System Call Prototype
We need to declare the prototype of your system call in the appropriate header file.
- Open `linux/include/linux/syscalls.h` and add the following code: 
```
/* obselete: hw/revstr.c */
asmlinkage long sys_revstr(char __user *user_str, size_t len);
```

## Build the Kernel
There is a step-by-step guide we can follow, start from step 3:

[How to Build Linux Kernel From Scratch {Step-By-Step Guide}](https://phoenixnap.com/kb/build-linux-kernel)

### Some Notes
#### Copy the existing Linux config file
In step 4, we copy the existing Linux config file using the cp command:
```
cp -v /boot/config-$(uname -r) .config
```
We can also use the config file under `linux/arch/x86/configs/`, but I'm not sure if the builded kernel can run on the existing Linux or not

#### Change kernel local version
We can change kernel local version by adding suffix in `Gernel Setup of Local version - append to kernel release` when doing `make menuconfig`

#### Change boot config
In order to boot the system with the customized kernel, we need to change GRUB's config file after `make install`
- Open boot config file:
```
sudo nano /etc/default/grub
```
We can find some boot setting variable in `/etc/default/grub`:
```
GRUB_DEFAULT=''
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR=`( . /etc/os-release; echo ${NAME:-Ubuntu} ) 2>/dev/null || ec>
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""
```
Add the following line:
```
GRUB_DEFAULT='Advanced options for Ubuntu>Ubuntu, with Linux 6.1.0-os-310553023'
```
The end of the code is depend on the kernel name we want to boot

we can check the kernel name via:
```
ls /boot/vmlinuz-*
```

After change the GRUB config, we need to updata the change
```
sudo upgade-grub
```

#### Others
1. We can use `make -j $(nproc)` to make building kernel faster
2. If `sudo update-initramfs -c -k 6.0.7` step filed is ok

