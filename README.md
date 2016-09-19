# Project 4: Geo-tagged File System

## OS Spring 2016

### DUE: Friday, 6/17/2015 at 8:59pm KST
### DESIGN REVIEW DUE: Monday, 6/7/2016 KST
### No slack allowed for Project 4

This project is the last OS project. Yay!

A key feature of modern mobile devices is the availability of various forms of physical sensor input such as accelerometer / orientation input, compass / directional input, and GPS / location input. In particular, mobile computing is taking advantage of location information to do things like geo-tag photos, provide turn-by-turn directions, locate friends / family, find restaurants, etc. Many of these "location aware" applications use the location information in an application specific way, and the code which uses this data must be re-written every time a new "location aware" application is developed.

In this project, you will develop a new kernel mechanism for embedding GPS location information into the filesystem, so that this information can be used by any application.

1.  **(10 pts) Kernel Device Location**

    Write a new system call which updates the kernel with the device's current location, then write a user space program `gpsupdate` that updates the kernel with the GPS location.

    The new system call number should be __`384`__ (check!), and the prototype should be:

    ```
    int set_gps_location(struct gps_location __user *loc);
    ```

    Where `struct gps_location` should be defined as:

	```
    struct gps_location {
    	double latitude;
    	double longitude;
    	float  accuracy;  /* in meters */
    };
	```

    All GPS related functions should be put in `kernel/gps.c` and `include/linux/gps.h`.

    For testing you can create an array of gps_location structs that include predefined gps values and simply round robin through the array every _x_ seconds to test in the device.

2.  **(35 pts) Ext2 GPS File System Modification**

    Modify the Linux inode operations interface to include GPS location manipulation functionality, then implement the new operations in the `ext2` file system.

    First of all, you need to enable the ext2 file system, because the ext2 file system is disabled in the default kernel build configuration. Tizen kernel uses the ext4 file system instead of ext2 or ext3 by turning on a `CONFIG_EXT4_USE_FOR_EXT23` option and off a `CONFIG_EXT2_FS` option. You should modify `arch/arm/configs/tizen_tm1_defconfig` file to enable the ext2 file system.

    To modify the inode operations interface, add the following two function pointers to the `struct inode_operations` structure in `include/linux/fs.h`:

	```
    int (*set_gps_location)(struct inode *);
    int (*get_gps_location)(struct inode *, struct gps_location *);
    ```

    Note that the `set_gps_location` function pointer does _not_ have an input gps location structure - it should use the latest GPS data available in the kernel as set by the `gpsupdate` you wrote for [Step 1](#p1).

    You only need to implement this GPS location feature **for the ext2 file system**. You should update a file's GPS coordinates whenever the file is **created _or_ modified**. You should also deal with **the directory as well as the file**. Look in the `fs/` directory for all the file system code, and in the `fs/ext2/` directory for ext2 specific code. You will need to change the physical representation of an inode on disk by adding the following three fields _in order_ to the _end_ of the appropriate ext2 structure:

    ```
    i_latitude (64-bits)
    i_longitude (64-bits)
    i_accuracy (32-bits)
    ```

    The three fields correspond to the `gps_location` structure fields.

    **HINT**: You will need to pay close attention to endian-ness of the fields you add to the ext2 physical inode structure. This data is intended to be read by both big _and_ little endian CPUs. Also note that the kernel does not have any floating point or double precision support.

    Now that you have modified the ext2 file system's physical representation on disk, you will also need to modify the user space tool which creates such a file system. You have to use the ext2 file system utilities in `[e2fsprogs](http://e2fsprogs.sourceforge.net/)`. Modify the appropriate file(s) in `e2fsprogs/lib/ext2fs/` to match the new physical layout. Compile the tool set using:
    ```
    os@m:~/proj4/e2fsprogs$ ./configure
    os@m:~/proj4/e2fsprogs$ make
    ```

    The tool you will be most interested in is `e2fsprogs/misc/mke2fs`. This tool should now create an ext2 file system with your custom modifications. You will use this tool _on your linux machine_ in [Step 3](#p3) to test the geo-tagged file system.

3.  **(15 pts) User Space Testing**

    Create a modified ext2 file system using the mke2fs tool from [Step 2](#p2), and write a user space utility named `file_loc` which will output a file's embedded location information including the GPS coordinates and a Google Maps link.

    In order to retrieve a file's location information, write a new system call numbered `385` with the following prototype:

    ```
    int get_gps_location(const char __user *pathname,
    		     struct gps_location __user *loc);
    ```

    On success, the system call should return 0. The `get_gps_location` system call should be successful only if the file is readable by the current user. It should return -1 on failure setting an appropriate error code which must include `ENODEV` if no GPS coordinates are embedded in the file.

    The user space utility, `file_loc`, should take exactly one argument which is the path to a file or directory. It should then print out the GPS coordinates / accuracy of the specified file or directory.

    In order to use this utility, you will need to create a modified ext2 file system using the `mke2fs` utility you modified in [Step 2](#p2). Create the ext2 file system _on your linux host_ using the loop back device:
    ```
    os@m:~/proj4$ dd if=/dev/zero of=proj4.fs bs=1M count=1
    os@m:~/proj4$ sudo losetup /dev/loop0 proj4.fs
    os@m:~/proj4$ sudo ./e2fsprogs/misc/mke2fs -I 256 -L os.proj4 /dev/loop0
    os@m:~/proj4$ sudo losetup -d /dev/loop0
    ```

    The file `proj4.fs` should now contain a modified ext2 file system. You can now push the file to your device and mount it: 
    ```
    os@m:~/proj4$ sdb push proj4.fs /home/developer/proj4.fs
    os@m:~/proj4$ sdb shell
    # mkdir /home/developer/proj4
    # mount -o loop -t ext2 /home/developer/proj4.fs /home/developer/proj4
    ```
    You can now create files and directories in `/home/developer/proj4` which should be geo-tagged. You need to include the `proj4.fs` file in you final git repository submission, and the filesystem must contain at least 1 directory and 2 files all of which must have unique GPS coordinates. Show the output of `file_loc` in your README when called on each of the files / directories you create. **NOTE**: be sure to un-mount the filesystem before pulling off the file from the device, otherwise you risk a corruption:     
    ```
    os@m:~/proj4$ sdb shell umount /home/developer/proj4
    os@m:~/proj4$ sdb pull /home/developer/proj4.fs
    ```
	**NOTE**: When we tested these steps in Tizen Z3, `umount` did not work. In that case, you can unmount the filesystem by rebooting your device.

    **HINT**: If you run into trouble using loop back devices, cross-compile the `losetup` utility. You can push this binary to the device and use the `-h` option for a description of the different operations it can perform.

    **HINT**: Wait until your device is fully booted before mounting the loopback device. Otherwise you may receive errors from the losetup utility.

4.  **(15 pts) Location-based File Access Authorization.**

    Update the file system so that files can be only readable from the location where they are created or modified. For example, if a file is created or modified lastly at Bldg. 301, it should not be allowed to be opened when your user program opens the file at Bldg. 302.
