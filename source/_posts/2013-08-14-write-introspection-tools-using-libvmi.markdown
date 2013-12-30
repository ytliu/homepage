---
layout: post
title: "Write introspection tools using libvmi"
date: 2013-08-14 21:34
comments: true
categories: Security Tool
---

Last week I've discussed about how to [setup libvmi](http://ytliu.info/blog/2013/08/04/libvmi-setup/), in this post I will show you how to write introspection tools using libvmi.

The most typical example is `examples/process-list.c`:

{% codeblock lang:c %}
#include <libvmi/libvmi.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/mman.h>
#include <stdio.h>
{% endcodeblock %}

Firstly we need to include these header files, especially the `<libvmi/libvmi.h>`, it defines most of the variables and functions.

Then we should initialize the vmi environment:
{% codeblock lang:c %}
	/* initialize the libvmi library */
    if (vmi_init(&vmi, VMI_AUTO | VMI_INIT_COMPLETE, name) == VMI_FAILURE) {
        printf("Failed to init LibVMI library.\n");
        return 1;
    }
{% endcodeblock %}

<!-- more -->

It will initialize the vmi (`vmi_instance_t`) struct with some platform information.

{% codeblock lang:c %}
	/* init the offset values */
    if (VMI_OS_LINUX == vmi_get_ostype(vmi)) {
        tasks_offset = vmi_get_offset(vmi, "linux_tasks");
        name_offset = vmi_get_offset(vmi, "linux_name");
        pid_offset = vmi_get_offset(vmi, "linux_pid");
    }
{% endcodeblock %}
Then in this case, we should read some offset of tast_struct from the config file `/etc/libvmi.conf`. And if you want to add a config argument like `linux_files`, you can do as follows:

------
Add an item in `libvmi/config/grammar.y`:
{% codeblock lang:c %}
...
%token<str> 	LINUX_FILES
...
assignment:
	|
...
	linux_state_assignment
	|
...
{% endcodeblock %}

Then add an item in `libvmi/config/lexicon.l`:
{% codeblock lang:c %}
linux_files 	{ BeginToken(yytext); yylval.str = strndup(yytext, CONFIG_STR_LENGTH); return LINUX_FILES;}
{% endcodeblock %}

Then add corresponding `strncmp` in `libvmi/os/linux/core.c`:
{% codeblock lang:c %}
...
if (stncmp(key, "linux_files", CONFIG_STR_LENGTH) == 0) {
	linux_instance->files_offset = *(int *)value;
	goto _done;
}
...
} else if (strncmp(offset_name, "linux_files", max_length) == 0) {
	return linux_instance->files_offset;
}
...
{% endcodeblock %}

At last add the `files_offset` item to `struct linux_instance` in `libvmi/os/linux/linux.h`:
{% codeblock lang:c %}
struct linux_instance {
	...
	uint64_t files_offset;
	...
}
{% endcodeblock %}

Then compile again, after that your can add `linux_files` to your config file.
------

Now, let's continue to look at the vmi code:
{% codeblock lang:c %}
	/* pause the vm for consistent memory access */
    if (vmi_pause_vm(vmi) != VMI_SUCCESS) {
        printf("Failed to pause VM\n");
        goto error_exit;
    } // if
{% endcodeblock %}

Before we do the vmi, we need to pause the vm for consistent memory access, then we can read memory from guest memory:
{% codeblock lang:c %}
 	/* get the head of the list */
    if (VMI_OS_LINUX == vmi_get_ostype(vmi)) {
        current_process = vmi_translate_ksym2v(vmi, "init_task");
    }
    ...
    /* walk the task list */
    list_head = current_process + tasks_offset;
    current_list_entry = list_head;

    status = vmi_read_addr_va(vmi, current_list_entry, 0, &next_list_entry);
    if (status == VMI_FAILURE) {
        printf("Failed to read next pointer at 0x%lx before entering loop\n",
                current_list_entry);
        goto error_exit;
    }
{% endcodeblock %}

We firstly get the head of the task_struct list `current_process`, then use `vmi_read_addr_va` to read the memory in address `current_list_entry`, which is the address of `next_list_entry`.

Here `vmi_read_addr_va` is similar with following `vmi_read_32_va`, `vmi_read_str_va` and so on. These lib functions take the virtual address (`vaddr`) as one of there parameters, they firstly walk the page table with cr3 and get physical address (`paddr`), then they use the libxenctrl to map a memory page of that paddr, which return back the virtual address (`memory`) seen by Dom0, with the `memory`, they can use `memcpy` to copy the corresponding size of memory to the `buf`.
{% codeblock lang:c %}
	do {
        vmi_read_32_va(vmi, current_process + pid_offset, 0, &pid);
        procname = vmi_read_str_va(vmi, current_process + name_offset, 0);
        if (procname) {
            free(procname);
            procname = NULL;
        }

        current_list_entry = next_list_entry;
        current_process = current_list_entry - tasks_offset;

        /* follow the next pointer */
        status = vmi_read_addr_va(vmi, current_list_entry, 0, &next_list_entry);
        if (status == VMI_FAILURE) {
            printf("Failed to read next pointer in loop at %lx\n", current_list_entry);
            goto error_exit;
        }

    } while (next_list_entry != list_head);
{% endcodeblock %}

This is the main loop of process vmi: it uses `vmi_read_32_va` to read pid (`pid_offset`) of each process, use `vmi_read_str_va` to read name (`name_offset`) of each process, and at last use `vmi_read_addr_va` to read the next entry of tast_struct in the list.

Finally we should never forget to resume the guest vm and destroy the vmi environment:
{% codeblock lang:c %}
	/* resume the vm */
    vmi_resume_vm(vmi);

    /* cleanup any memory associated with the LibVMI instance */
    vmi_destroy(vmi);
{% endcodeblock %}

We can compile and run the vmi tool in the root directory of libvmi:

	$ make
	$ sudo ./examples/process-list vm_name

------

If you want to write your own vmi tools, you can imitate the `process-list` to write a new one (e.g. my_vmi_tool.c). After that, you need to modify the `examples/Makefile.am` file:
{% codeblock lang:c %}
...
bin_PROGRAMS = ... my_vmi_tool
...
my_vmi_tool_SOURCES = my_vmi_tool.c
{% endcodeblock %}
Actually after we run `make` in the root directory of libvmi, the `Makefile` in `examples/` directory will generate a new executable file called `my_vmi_tool`, which is a script file to invoke the actual code in your `my_vmi_tool.c`.

------

Last thing you should know is: currently vmi tool are all *user mode* application code. It does not support to write vmi tool as a kernel module. The environment you vmi code run is just the user space on the host OS.

Whenever you have any problem with libvmi, you can create a post in their [googlegroup](http://groups.google.com/group/vmitools/topics). The guys there are really very friendly, I've post 2 topics there and both receive very good feedbacks.