# Tesla hacking 2016

## Tesla Model S

[Tesla hardware infomation](https://www.pentestpartners.com/security-blog/reverse-engineering-tesla-hardware/)

![](img/teslastruct.png)

- CID (Central Information Display) – the large console at the center of the dash
- IC (Instrument Cluster)
- Conventional CAN connected ECUs
- ECU (Electronic Control Unit)
- CAN - Controller Area Network
- VCM (Visual Computer Module)



## Tested Tesla  information

Tested model: Model S

software version: v7.1(2.32.23)

User Agent of web browser: Mozilla/5.0 (X11; Linux) AppleWebKit/534.34 (KHTML, like Gecko) QtCarBrowser Safari/534.34",



## Remote Attack Surface

`Tesla Gust` provided by Tesla body shop and superchargers can be automatically connected by Tesla, because the car remember the Wi-Fi SSID. If faking Wi-Fi hotspot and redirecting the traffic of `QtCarBrowser`, hacking remotely cars can be feasible.

## Vulnerability in Browser

**WebKit**: A open source web browser engine

From the version of User Agent, it can be deduced that the version of QtWebKit is 2.2.x.

[source of QtWebKit 2.2](https://github.com/NNemec/qtwebkit-2.2)

### ArrayStorage

`ArrayStorage` is a type which holds the actual data values of an array. There may be space before `ArrayStorage` for quick unshift / shift operation.

path: `qtwebkit-2.2/Source/JavaScriptCore/runtime/JSArray.h`

![](img/ArrayStorage.png)

It may be stored like this in memory.

![](img/ArrayStorageStructure.png)

### ShiftCount and UnShiftCount

**ShiftCount**: Pop some element in the vector and move array to right. Precisely, the array shift size of `JSValue` * number of removed element.

![](img/ShiftCount.png)

**UnShiftCount**: Pop some element in the vector and move array to left. Precisely, the array shift size of `JSValue` * number of removed element. In addition, if count larger than free space before the array, it will move to new place.

![](img/UnshiftCount.png)

### JSValue structure

Path: `qtwebkit-2.2/Source/JavaScriptCore/runtime/JSValue.h`

![](img/JSValue.png)



### Vulnerability in JSArray::sort

Function path: `qtwebkit-2.2/Source/JavaScriptCore/runtime/JSArray.cpp`

`JSArray::sort` mainly do following things:

1. Copy elements from storage into `AVLTree`(avltree?)
2. Call `compareFunction` for sorting
3. Copy sorted element back in storage

```c
void JSArray::sort(ExecState* exec, JSValue compareFunction, CallType callType, const CallData& callData) 
{ 
    checkConsistency(); 
    ArrayStorage* storage = m_storage; // local variable cause the vulnerability
    ...... 
    // Copy the values back into m_storage. 
    AVLTree<AVLTreeAbstractorForArrayCompare, 44>::Iterator iter; 
    iter.start_iter_least(tree); 
    JSGlobalData& globalData = exec->globalData(); 
    for (unsigned i = 0; i < numDefined; ++i) 
    { 
        // Vulnerability 1
        storage->m_vector[i].set(globalData, this, 
                                 tree.abstractor().m_nodes[*iter].value); 
        ++iter; 
    } 
    ......
    // Vulnerability 2
    storage->m_numValuesInVector = newUsedVectorLength; 
    ......
}
```

The vulnerability is that if `compareFunction` is shift function, `JSArray::shiftCount()` or `JSArray::unshiftCount()`, the length of the m_vector will be changed, also the entire m_vector structure will be shifted to another place. However, the local variable pointer storage still points to the old location, result in memory corruption.

![](img/ArrayShiftSample.png)

## Exploit Vulnerability in Browser

### Leaking JSCell address

Because of `Vulnerablity 2` in `JSArray::sort`, some cell can be overlapped with `m_numValueInVector`. Since *true address can be got after exploit the payload of `JSvalue`*,   we can overlap `tag` in `JSValue` with `num`. Then, the address can be targeted by finding the double value.

![](img/AddressLeaking.png)

### Free Storage at any Address

Because `m_allocBase` points to the head address of the array,  the target address can be got. In addition, by using the feature of `unshiftcount` , the space, which cannot contain the array because of many times of unshift operation, will be free. So, it is easy to free storage by following step:

1. shift the array so that the `m_allocBase` can be overlapped with `payload` in an `JSValue` element.
2. unshift the same array until `increaseVectorPrefixLength` is triggered.

However, there are some bugs when calling sort if we do so.

![](img/CrushWhenShift.png)

The local value `storage` find the map is not equal to zero after shift, and execute this statement `map->size()` which may cause crush. (`map` is actually the `m_length` of `m_storage`)

To solve this, we can unshift the array and overlap the `m_length` with zero. Fortunately, there is no assert about `m_length`.

![](img/FreeSpace.png)

Then by using Use After Free the pointer, the old space can be used.

Now, we can get the array address and arbitrarily allocate and free space



## CVE-2011-3928

`copyNonAttributeProperties` is unsafe when calling `static_cast`

Path: `qtwebkit-2.2/Source/WebCore/html/HTMLInputElement.cpp`

`qtwebkit-2.2/SourceWebcore/dom/InputElement.cpp`

`qtwebkit-2.2/Source/JavaScriptCore/wtf/text/StringImpl.h`



## Local Privilege Escalation

The kernel version used in CID at that time was v2.6.36.

BSP(Board Support Package):  containing hardware-specific drivers and other routines that allow a particular operating system (traditionally a real-time operating system, or RTOS) to function in a particular hardware environment (a computer), integrated with the RTOS itself. 

The (1) get_user and (2) put_user API functions in the Linux kernel before 3.5.5 on the v6k and v7 ARM platforms do not validate certain addresses, which allows attackers to read or modify the contents of arbitrary kernel memory locations via a crafted application.



[exploit: CVE-2013-6282](https://github.com/praetorian-inc/purple-team-attack-automation/tree/5e07f93720d2c8f8269c626d6a88801351784d01/external/source/exploits/CVE-2013-6282)

`fsync` pointer can be used. The structure `file_operations` is at `linux2.6.36/include/linux/fsh.h`. 

![](img/file_operation.png)

![](img/ptmx_fops.png)

The address of `fsync` of `ptmx_fops` can be deduced. It is `ptmx_fops`+0x38. :question:

By calling `/proc/kallsyms` , the address of `ptmx_fops` can be known.

```c
void * kallsyms_get_symbol_address(const char *symbol_name)
{
  ......
  fp = fopen("/proc/kallsyms", "r");
  ......
  while(!feof(fp)) {
    ret = fscanf(fp, "%p %c %s", &address, &symbol, function); // get address
    if (!strcmp(function, symbol_name)) {
      fclose(fp);
      return address;
    }
  }
  ......
}
// get ptmx_fops address by following statement:
// kallsyms_get_symbol_address("ptmx_fops");
```

we can exploit `put_user`, which called by `ioctl`, by following function:

```c
int pipe_write_value_at_address(unsigned long address, void* value)  
{  
	......  
    *(long *)&data = (long)value;  
	......
    for (i = 0; i < (int) sizeof(data) ; i++) {  
 		......
        //write data[i] number ofcharacters into buff
       	if (write(pipefd[1], buf, data[i]) != data[i])
		......
        //write the number of element in buff,which is data[i], into the address+i
        if (ioctl(pipefd[0], FIONREAD, (void *)(address + i)) == -1)
        ......
        // flush buff
        if (read(pipefd[0], buf, sizeof buf) != data[i])
        ......
    }
    return (i == sizeof (data));  
}
// pipe_write_value_at_address(ptmx_fops`+0x38, some_patch); 
```



Based on the CVE-2013-6282 , we can get the arbitrary read/write in kernel context.

**setresuid(ruid,euid,suid)** 

- Real user ID and real group ID. These IDs determine who owns the process.
- Effective user ID and effective group ID. These IDs are used by the kernel to determine the permissions that the process will have when accessing shared resources such as message queues, shared memory, and semaphores. On most UNIX systems, these IDs also determine the permissions when accessing files. However, Linux uses the file system IDs for this task.
- Saved set-user-ID and saved set-group-ID. These IDs are used in set-user-ID and set-group-ID programs to save a copy of the corresponding effective IDs that were set when the program was executed. A set-user-ID program can assume and drop privileges by switching its effective user ID back and forth between the values in its real user ID and saved set-user-ID.

path: `linux2.6.36/kernel/sys.c`

![](img/setresuid.png)

We just change the condition `retval` to `true`. Thus, whenever `sys_setresuid(0,0,0)` is called, It will succeed to get root.

Then invoking `reset_security_ops()` to disable `AppArmor`, the OS is fully controlled. 

Therefore, developing exploit becomes easier.



## Access Of other Embedded System

### IC

We can `ssh` into `IC` without password by using `ssh root@ic`. (ifconfig?)Re

In Addition, `CID` can be accessed from `IC` by `ssh` because the password is stored in `.ssh` in plaintext.

### Parrot

Parrot: The third-party module Parrot on Tesla Model S is FC6050W, which integrates the Wireless function and Bluetooth function. Parrot uses the USB Ethernet gadget so that Parrot can communicate with CID trough Ethernet. 

port 23 is opened for Telnet, and Telnet is anonymous. Using command `nc parrot 23` to make parrot under control.

### Gateway

The `gw-diag` binary file implies the gateway can be called by some function. Researchers find the function can be called through port 3500.  Reversing the binary file helps researchers find out the function `ENABLE_SHELL`. The command `printf"\x12\x01 | socat -udp:gw:3500"` can wake up Gateway's backdoor on port 23.

To enter the shell, the password of the shell is necessary. The password is static and written in the firmware of Gate. The security token can be got by reversing the firmware. :question:

Therefore, it is possible to enter the shell by command `nc gw 23` and password.

## ECU Programming

A SD Card without protection is directly connected to the Gateway. By examine `FAT FS` on this SD Card, there are some files:

![](img/filesonsdcard.png)

Exploring the file `udsdebug.log` which has detailed update log, we can get the overview of update.

`booted.img` (disk image) is used for software programing. It is renamed from `boot.img` for preventing boot into the file again, and will be loaded to 0x40000000 (1GB) of Gateway's RAM, then executed. 

In the first 1GB of SD Card, there may be the boot loader and Gateway software.

```c
#define BIGENDIAN // The Format of boot.img
typedef struct { 
	byte jump_command[4]; // 48000020 means jmp $+20
    uint32 crc32_value; // fill in FFh, calculate, then write back 
	int32 image_size; // filesize 
	int32 neg_image_size; // -filesize 
	int32 memoinfo_length; // length of memoinfo 
	byte memoinfo[memoinfo_length]; 
	byte image_content[0]; 
} tBtImg;
```

CRC collision



