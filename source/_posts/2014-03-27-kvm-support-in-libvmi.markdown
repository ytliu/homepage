---
layout: post
title: "KVM support in Libvmi"
date: 2014-03-27 16:50
comments: true
categories: Security Tool KVM
---

Several months ago I've post 2 blogs to introduce [how to setup libvmi](http://ytliu.info/blog/2013/08/04/libvmi-setup/) and [how to write your own tools using libvmi](http://ytliu.info/blog/2013/08/14/write-introspection-tools-using-libvmi/). These two posts are based on Xen virtualization environment. Today I'm trying to use Libvmi for KVM virtual machine introspection, which need more effort to do such task. 

In the rest of this blog, I will briefly introduce why this effort need to be done and how to do that.

<!-- more -->

------

Before we start, let's firstly see how document in libvmi github page saying about *KVM Support*:

> If you would like LibVMI to work on KVM VM's, you must do some additional setup. This is because KVM doesn't have much built-in capability for introspection.

> You only need one memory access technique. LibVMI will first look for the QEMU-KVM patch and use that if it is installed. Otherwise it will fall back to using GDB.

And now Libvmi provide 3 ways to support KVM introspection:

* Enable GDB access to your KVM VM, which is the slowest approach;
* Patch QEMU-KVM with the provided patch, which is much faster;
* Use Shm-snapshot Support to introspect on a memory snapshot, which is the fatest one.

In this blog, I'll not consider the GDB method since it will introduce much overhead, and only discuss about the second QEMU-KVM patch way. While the third Shm-snapshot will be introduced in the future.

So as far as I'm concerned, why Libvmi require applying a qemu patch to introspect KVM virtual machine is for following reasons:

* Libvmi use libvirt framework API to manage and get data from guest virtual machine;
* However, unlike libxenctrl used in Xen, libvirt for KVM does not provide any API to map Guest Physical Address (GPA) to Host Virtual Address (HVA);
* While in qemu, there is a function called `cpu_physical_memory_map` in `qemu/exec.c` file which can map guest GPA to HVA.

So what the patch actually do is use Qemu Machine Protocal (QMP) mechanism for libvmi to pass the GPA, and call the internal function `cpu_physical_memory_map` located in `qemu/exec.c` in Qemu, and finally get the mapped HVA back.

------

Actually the patch are used for some specific qemu versions, so I think I should just tell why it works, and how to make it work in general.

#### Create a connection inside Qemu for Libvmi to communicate

The entry function is `do_physical_memory_access()`, we can put it anywhere, for example, Libvmi patch put it in `qemu/monitor.c` file:

{% codeblock lang:c %}
static int do_physical_memory_access(Monitor *mon, const QDict *qdict, QObject **ret_data)
{
  const char *path = qdict_get_str(qdict, "path");
  memory_access_start(path);
  return 0;
}
{% endcodeblock %}

This function will have a input parameter `path`，with which to invoke `memory_access_start()` function:

(Note: The following code are all located in `qemu/memory-access.c`)

{% codeblock lang:c %}
int
memory_access_start (const char *path)
{
  pthread_t thread;
  sigset_t set, oldset;
  int ret;

  // create a copy of path that we can safely use
  char *pathcopy = malloc(strlen(path) + 1);
  memcpy(pathcopy, path, strlen(path) + 1);

  // start the thread
  sigfillset(&set);
  pthread_sigmask(SIG_SETMASK, &set, &oldset);
  ret = pthread_create(&thread, NULL, memory_access_thread, pathcopy);
  pthread_sigmask(SIG_SETMASK, &oldset, NULL);

  return ret;
}
{% endcodeblock %}

This function will create a new thread, and run the `memory_access_thread()` function:

{% codeblock lang:c %}
static void *
memory_access_thread (void *path)
{
  struct sockaddr_un address;
  int socket_fd, connection_fd;
  socklen_t address_length;

  socket_fd = socket(PF_UNIX, SOCK_STREAM, 0);
  if (socket_fd < 0){
    printf("QemuMemoryAccess: socket failed\n");
    goto error_exit;
  }
  unlink(path);
  address.sun_family = AF_UNIX;
  address_length = sizeof(address.sun_family) + sprintf(address.sun_path, "%s", (char *) path);

  if (bind(socket_fd, (struct sockaddr *) &address, address_length) != 0){
  printf("QemuMemoryAccess: bind failed\n");
  goto error_exit;
}
if (listen(socket_fd, 0) != 0){
  printf("QemuMemoryAccess: listen failed\n");
  goto error_exit;
}

connection_fd = accept(socket_fd, (struct sockaddr *) &address, &address_length);
connection_handler(connection_fd);

close(socket_fd);
unlink(path);
error_exit:
return NULL;
}
{% endcodeblock %}

This function will create a socket, combine it with the `/tmp/path` named file, bind, listen for connection, and once any other process connect to such socket, it will invoke the `connection_handler()` function:

{% codeblock lang:c %}
static void
connection_handler (int connection_fd)
{
  int nbytes;
  struct request req;

  while (1){
    // client request should match the struct request format
    nbytes = read(connection_fd, &req, sizeof(struct request));
    if (nbytes != sizeof(struct request)){
      // error
      continue;
    }
    else if (req.type == 0){
      // request to quit, goodbye
      break;
    }
    else if (req.type == 1){
      // request to read
      char *buf = malloc(req.length + 1);
      nbytes = connection_read_memory(req.address, buf, req.length);
      if (nbytes != req.length){
        // read failure, return failure message
        buf[req.length] = 0; // set last byte to 0 for failure
        nbytes = write(connection_fd, buf, 1);
      }
      else{
        // read success, return bytes
        buf[req.length] = 1; // set last byte to 1 for success
        nbytes = write(connection_fd, buf, nbytes + 1);
      }
      free(buf);
    }
    else if (req.type == 2){
      // request to write
      void *write_buf = malloc(req.length);
      nbytes = read(connection_fd, &write_buf, req.length);
      if (nbytes != req.length){
        // failed reading the message to write
        send_fail_ack(connection_fd);
      }
      else{
        // do the write
        nbytes = connection_write_memory(req.address, write_buf, req.length);
        if (nbytes == req.length){
          send_success_ack(connection_fd);
        }
        else{
          send_fail_ack(connection_fd);
        }
      }
      free(write_buf);
    }
    else{
      // unknown command
      printf("QemuMemoryAccess: ignoring unknown command (%d)\n", req.type);
      char *buf = malloc(1);
      buf[0] = 0;
      nbytes = write(connection_fd, buf, 1);
      free(buf);
    }
  }

  close(connection_fd);
}
{% endcodeblock %}

This function will parse the request from the connection_fd, the types of request are divided to 3: 

* 0: quit
* 1: read
* 2: write

Once it receive read request, it will invoke the `connection_read_memory()` function:

{% codeblock lang:c %}
static uint64_t
connection_read_memory (uint64_t user_paddr, void *buf, uint64_t user_len)
{
    hwaddr paddr = (hwaddr) user_paddr;
    hwaddr len = (hwaddr) user_len;
    void *guestmem = cpu_physical_memory_map(paddr, &len, 0);
    if (!guestmem){
        return 0;
    }
    memcpy(buf, guestmem, len);
    cpu_physical_memory_unmap(guestmem, len, 0, len);

    return len;
}
{% endcodeblock %}

Once it receive write request, it will invoke the `connection_write_memory()` function:

{% codeblock lang:c %}
static uint64_t
connection_write_memory (uint64_t user_paddr, void *buf, uint64_t user_len)
{
    hwaddr paddr = (hwaddr) user_paddr;
    hwaddr len = (hwaddr) user_len;
    void *guestmem = cpu_physical_memory_map(paddr, &len, 1);
    if (!guestmem){
        return 0;
    }
    memcpy(guestmem, buf, len);
    cpu_physical_memory_unmap(guestmem, len, 0, len);

    return len;
}
{% endcodeblock %}

Then, what we need to do is to invoke the very beginning function `do_physical_memory_access()`. So here comes the QMP:

#### Patch QMP to provide a entry to invoke `do_physical_memory_access()` method

How QMP works will be introduced in a new blog in the future. Here I just need to tell what need to add:

in `qemu/qmp-command.hx` file, we add following code:

{% codeblock lang:xml %}
{
  .name       = "pmemaccess",
  .args_type  = "path:s",
  .params     = "path",
  .help       = "mount guest physical memory image at 'path'",
  .user_print = monitor_user_noop,
  .mhandler.cmd_new = do_physical_memory_access,
},

SQMP
pmemaccess
----------

Mount guest physical memory image at 'path'.

Arguments:

- "path": mount point path (json-string)

Example:

-> { "execute": "pmemaccess",
             "arguments": { "path": "/tmp/guestname" } }
<- { "return": {} }

EQMP
{% endcodeblock %}

Note that the code we need here is:

{% codeblock lang:xml %}
{
  .name       = "pmemaccess",
  .args_type  = "path:s",
  .params     = "path",
  .help       = "mount guest physical memory image at 'path'",
  .user_print = monitor_user_noop,
  .mhandler.cmd_new = do_physical_memory_access,
},
{% endcodeblock %}

the other code between `SQMP` and `EQMP` are just added to the documentation. But it is required!

After adding this command to Qemu QMP, we can invoke `do_physical_memory_access()` outside of Qemu using such format shown in Example section:

{% codeblock lang:xml %}
-> { "execute": "pmemaccess",
             "arguments": { "path": "/tmp/guestname" } }
<- { "return": {} }
{% endcodeblock %}

For example, in Libvmi, you can find how it invokes this in `libvmi/driver/kvm.c` file:

{% codeblock lang:c %}
//
// QMP Command Interactions
static char *
exec_qmp_cmd(
    kvm_instance_t *kvm,
    char *query)
{
  ......

  snprintf(cmd, cmd_length, "virsh qemu-monitor-command %s %s", name,
   query);

  ......

  p = popen(cmd, "r");

  ......
}

static char *
exec_memory_access(
  kvm_instance_t *kvm)
{
  char *tmpfile = tempnam("/tmp", "vmi");
  char *query = (char *) safe_malloc(256);

  sprintf(query,
  "'{\"execute\": \"pmemaccess\", \"arguments\": {\"path\": \"%s\"}}'",
  tmpfile);

  ......

  char *output = exec_qmp_cmd(kvm, query);

  ......
}

{% endcodeblock %}

And when it needs to read a page, it needs to map from GPA to HVA, then the libvmi will follow the control from:

  vmi_read_va -> (get GPA from vmi_translate_uv2p) -> kvm_read_page -> memory_cache_insert -> 
  create_new_entry -> get_memory_data -> get_memory_callback -> kvm_get_memory_patch

For the last function, it can be found in file `libvmi/driver/kvm.c`:

{% codeblock lang:c %}
void *
kvm_get_memory_patch(
    vmi_instance_t vmi,
    addr_t paddr,
    uint32_t length)
{
  char *buf = safe_malloc(length + 1);
  struct request req;

  req.type = 1;   // read request
  req.address = (uint64_t) paddr;
  req.length = (uint64_t) length;

  int nbytes =
  write(kvm_get_instance(vmi)->socket_fd, &req,
    sizeof(struct request));
  if (nbytes != sizeof(struct request)) {
    goto error_exit;
  }
  else {
    // get the data from kvm
    nbytes =
    read(kvm_get_instance(vmi)->socket_fd, buf, length + 1);
    if (nbytes != (length + 1)) {
      goto error_exit;
    }

    // check that kvm thinks everything is ok by looking at the last byte
    // of the buffer, 0 is failure and 1 is success
    if (buf[length]) {
      // success, return pointer to buf
      return buf;
    }
  }

  // default failure
  error_exit:
  if (buf)
  free(buf);
  return NULL;
}
{% endcodeblock %}

So it uses the socket_fd to communicate with the connection openned in Qemu to invoke the `cpu_physical_read_memory()`. So the same with write.

------

Above are the principle how Qemu-patch work in Libvmi for KVM support, you can find the whole patch [here](https://github.com/bdpayne/libvmi/tree/master/tools/qemu-kvm-patch), now it only support 0.14 and 1.2 versions. If you know how it works, you can modify the patch and apply your own Qemu version. For example, following is my patch for the Qemu in stable-1.6 branch, e82ee0845c3240541e79b9b521779b3f8743f1b4 commit:

{% codeblock lang:c %}
 diff --git a/Makefile.target b/Makefile.target
index 9a49852..be93dd0 100644
--- a/Makefile.target
+++ b/Makefile.target
@@ -113,7 +113,7 @@ endif #CONFIG_BSD_USER
 #########################################################
 # System emulator target
 ifdef CONFIG_SOFTMMU
-obj-y += arch_init.o cpus.o monitor.o gdbstub.o balloon.o ioport.o
+obj-y += arch_init.o cpus.o monitor.o gdbstub.o balloon.o ioport.o memory-access.o
 obj-y += qtest.o
 obj-y += hw/
 obj-$(CONFIG_FDT) += device_tree.o
diff --git a/memory-access.c b/memory-access.c
new file mode 100644
index 0000000..2c81c48
--- /dev/null
+++ b/memory-access.c
@@ -0,0 +1,205 @@
+/*
+ * Access guest physical memory via a domain socket.
+ *
+ * Copyright (C) 2011 Sandia National Laboratories
+ * Author: Bryan D. Payne (bdpayne@acm.org)
+ */
+
+#include "memory-access.h"
+//#include "cpu-all.h"
+#include "qemu-common.h"
+#include "exec/cpu-common.h"
+#include "config.h"
+
+#include <stdlib.h>
+#include <stdio.h>
+#include <string.h>
+#include <pthread.h>
+#include <sys/types.h>
+#include <sys/socket.h>
+#include <sys/un.h>
+#include <unistd.h>
+#include <signal.h>
+#include <stdint.h>
+
+struct request{
+  uint8_t type;      // 0 quit, 1 read, 2 write, ... rest reserved
+  uint64_t address;  // address to read from OR write to
+  uint64_t length;   // number of bytes to read OR write
+};
+
+typedef uint64_t target_phys_addr_t;
+
+  static uint64_t
+connection_read_memory (uint64_t user_paddr, void *buf, uint64_t user_len)
+{
+  target_phys_addr_t paddr = (target_phys_addr_t) user_paddr;
+  target_phys_addr_t len = (target_phys_addr_t) user_len;
+  void *guestmem = cpu_physical_memory_map(paddr, &len, 0);
+  if (!guestmem){
+    return 0;
+  }
+  memcpy(buf, guestmem, len);
+  cpu_physical_memory_unmap(guestmem, len, 0, len);
+
+  return len;
+}
+
+  static uint64_t
+connection_write_memory (uint64_t user_paddr, void *buf, uint64_t user_len)
+{
+  target_phys_addr_t paddr = (target_phys_addr_t) user_paddr;
+  target_phys_addr_t len = (target_phys_addr_t) user_len;
+  void *guestmem = cpu_physical_memory_map(paddr, &len, 1);
+  if (!guestmem){
+    return 0;
+  }
+  memcpy(guestmem, buf, len);
+  cpu_physical_memory_unmap(guestmem, len, 0, len);
+
+  return len;
+}
+
+  static void
+send_success_ack (int connection_fd)
+{
+  uint8_t success = 1;
+  int nbytes = write(connection_fd, &success, 1);
+  if (1 != nbytes){
+    printf("QemuMemoryAccess: failed to send success ack\n");
+  }
+}
+
+  static void
+send_fail_ack (int connection_fd)
+{
+  uint8_t fail = 0;
+  int nbytes = write(connection_fd, &fail, 1);
+  if (1 != nbytes){
+    printf("QemuMemoryAccess: failed to send fail ack\n");
+  }
+}
+
+  static void
+connection_handler (int connection_fd)
+{
+  int nbytes;
+  struct request req;
+
+  while (1){
+    // client request should match the struct request format
+    nbytes = read(connection_fd, &req, sizeof(struct request));
+    printf("req is %d\n", req.type);
+    if (nbytes != sizeof(struct request)){
+      // error
+      continue;
+    }
+    else if (req.type == 0){
+      // request to quit, goodbye
+      break;
+    }
+    else if (req.type == 1){
+      // request to read
+      char *buf = malloc(req.length + 1);
+      nbytes = connection_read_memory(req.address, buf, req.length);
+      if (nbytes != req.length){
+        // read failure, return failure message
+        buf[req.length] = 0; // set last byte to 0 for failure
+        nbytes = write(connection_fd, buf, 1);
+      }
+      else{
+        // read success, return bytes
+        buf[req.length] = 1; // set last byte to 1 for success
+        nbytes = write(connection_fd, buf, nbytes + 1);
+      }
+      free(buf);
+    }
+    else if (req.type == 2){
+      // request to write
+      void *write_buf = malloc(req.length);
+      nbytes = read(connection_fd, &write_buf, req.length);
+      if (nbytes != req.length){
+        // failed reading the message to write
+        send_fail_ack(connection_fd);
+      }
+      else{
+        // do the write
+        nbytes = connection_write_memory(req.address, write_buf, req.length);
+        if (nbytes == req.length){
+          send_success_ack(connection_fd);
+        }
+        else{
+          send_fail_ack(connection_fd);
+        }
+      }
+      free(write_buf);
+    }
+    else{
+      // unknown command
+      printf("QemuMemoryAccess: ignoring unknown command (%d)\n", req.type);
+      char *buf = malloc(1);
+      buf[0] = 0;
+      nbytes = write(connection_fd, buf, 1);
+      free(buf);
+    }
+  }
+
+  close(connection_fd);
+}
+
+  static void *
+memory_access_thread (void *path)
+{
+  struct sockaddr_un address;
+  int socket_fd, connection_fd;
+  socklen_t address_length;
+    
+  printf("in memory_access_thread : %s\n", (char *)path);
+
+  socket_fd = socket(PF_UNIX, SOCK_STREAM, 0);
+  if (socket_fd < 0){
+    printf("QemuMemoryAccess: socket failed\n");
+    goto error_exit;
+  }
+  unlink(path);
+  address.sun_family = AF_UNIX;
+  address_length = sizeof(address.sun_family) + sprintf(address.sun_path, "%s", (char *) path);
+
+  if (bind(socket_fd, (struct sockaddr *) &address, address_length) != 0){
+    printf("QemuMemoryAccess: bind failed\n");
+    goto error_exit;
+  }
+  printf("in memory_access_thread : %d\n", socket_fd);
+  if (listen(socket_fd, 0) != 0){
+    printf("QemuMemoryAccess: listen failed\n");
+    goto error_exit;
+  }
+
+  connection_fd = accept(socket_fd, (struct sockaddr *) &address, &address_length);
+  connection_handler(connection_fd);
+
+  close(socket_fd);
+  unlink(path);
+error_exit:
+  return NULL;
+}
+
+  int
+memory_access_start (const char *path)
+{
+  pthread_t thread;
+  sigset_t set, oldset;
+  int ret;
+
+  // create a copy of path that we can safely use
+  char *pathcopy = malloc(strlen(path) + 1);
+  memcpy(pathcopy, path, strlen(path) + 1);
+
+  // start the thread
+  sigfillset(&set);
+  pthread_sigmask(SIG_SETMASK, &set, &oldset);
+  ret = pthread_create(&thread, NULL, memory_access_thread, pathcopy);
+  pthread_sigmask(SIG_SETMASK, &oldset, NULL);
+
+  return ret;
+}
diff --git a/memory-access.h b/memory-access.h
new file mode 100644
index 0000000..e538134
--- /dev/null
+++ b/memory-access.h
@@ -0,0 +1,8 @@
+/*
+ * Mount guest physical memory using FUSE.
+ *
+ * Copyright (C) 2011 Sandia National Laboratories
+ * Author: Bryan D. Payne (bdpayne@acm.org)
+ */
+
+int memory_access_start (const char *path);
diff --git a/monitor.c b/monitor.c
index 99bfcd9..7dd4ac2 100644
--- a/monitor.c
+++ b/monitor.c
@@ -67,6 +67,7 @@
 #include "qmp-commands.h"
 #include "hmp.h"
 #include "qemu/thread.h"
+#include "memory-access.h"
 
 /* for pic/irq_info */
 #if defined(TARGET_SPARC)
@@ -1252,6 +1253,14 @@ static void do_print(Monitor *mon, const QDict *qdict)
     monitor_printf(mon, "\n");
 }
 
+static int do_physical_memory_access(Monitor *mon, const QDict *qdict, QObject **ret_data)
+{
+    const char *path = qdict_get_str(qdict, "path");
+    printf("in do_physical_memory_access : %s\n", path);
+    memory_access_start(path);
+    return 0;
+}
+
 static void do_sum(Monitor *mon, const QDict *qdict)
 {
     uint32_t addr;
diff --git a/qmp-commands.hx b/qmp-commands.hx
index cf47e3f..41b3e1b 100644
--- a/qmp-commands.hx
+++ b/qmp-commands.hx
@@ -610,6 +610,33 @@ Example:
 EQMP
 
     {
+        .name       = "pmemaccess",
+        .args_type  = "path:s",
+        .params     = "path",
+        .help       = "mount guest physical memory image at 'path'",
+        .user_print = monitor_user_noop,
+        .mhandler.cmd_new = do_physical_memory_access,
+    },
+
+SQMP
+pmemaccess
+----------
+
+Mount guest physical memory image at 'path'.
+
+Arguments:
+
+- "path": mount point path (json-string)
+
+Example:
+
+-> { "execute": "pmemaccess",
+             "arguments": { "path": "/tmp/guestname" } }
+<- { "return": {} }
+
+EQMP
+
+    {
         .name       = "migrate",
         .args_type  = "detach:-d,blk:-b,inc:-i,uri:s",
         .mhandler.cmd_new = qmp_marshal_input_migrate,

{% endcodeblock %}

After we apply the Qemuu patch, we can re-compile the Qemu:

    $ cd qemu
    $ ./configure
    $ make -jj4
    $ sudo make install

In addition, as said in the Libvmi READM, you need to make sure your libvirt version is 0.8.7 or newer, and you should sure that the libvirt installation supports QMP commands, which can be done by install libyajl-dev (take my Debian as an example):

    $ sudo aptitude install libyajl-dev
    $ cd libvirt
    $ ./configure

And ensure that the configure script reports that it found yajl. Then you should compile the libvirt

    $ make -j4
    $ sudo make install

After that, you can setup libvmi as before.


