---
layout: post
title:  "Struct MachineClass in QEMU"
date:   2020-08-28 20:00:00 +0800
categories: jekyll update
---

> Auther: Percy
>
> Updated at 2020/8/28

## Intro

Before looking at the specific structure, let us have a glance on the use of this structure in the project.

Begin with the main entrance file [/softmmu/vl.c](https://github.com/qemu/qemu/blob/master/softmmu/vl.c). 

```Markdown
{% raw %}
{% mermaid %}
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
{% endmermaid %}
{% endraw %}
```


In func `qemu_init`, we will have two rounds of option parsing. In the first pass of option parsing, count the number of options and validate the legality of options.

```c
    /* first pass of option parsing */
    optind = 1;
    while (optind < argc) {
        if (argv[optind][0] != '-') {
            /* disk image */
            optind++;
        } else {
            const QEMUOption *popt;

            popt = lookup_opt(argc, argv, &optarg, &optind);
            switch (popt->index) {
            case QEMU_OPTION_nouserconfig:
                userconfig = false;
                break;
            }
        }
    }
```

And there is a speial case `QEMU_OPTION_nouserconfig`. It represents that if this argument exsits, the configuration process will use the corresponding config file. After this first pass, the second pass will start to fill the option list.

```c
            case QEMU_OPTION_machine:
                olist = qemu_find_opts("machine");
                opts = qemu_opts_parse_noisily(olist, optarg, true);
                if (!opts) {
                    exit(1);
                }
                break;
...
    machine_class = select_machine();
```

In func `select_machine`,  get a single-linked list of `MachineClass` at first with function `object_class_get_list`.  Walk this list to find the default value. Then get the machine_opts and extract the `type` property. Pass this string and previous list to func `machine_parse`. The target class will be given by the func `find_machine`  through the comparasion of name property. (If hlep option attached, all supported classes which are sorted and default class will be shown. )

---

After got the target class, qemu can do initialization with it. With the analysis of the properties which is hold by `MachineClass `, the understanding of its role may be easier. The detailed descriptions are all added in the appendix.



## Appendix

### MachineClass Definition

```c
struct MachineClass {
    /*< private >*/
    ObjectClass parent_class;
    /*< public >*/

    const char *family; /* NULL iff @name identifies a standalone machtype */
    char *name;
    const char *alias; /* another name */
    const char *desc;
    const char *deprecation_reason; /* If set, the machine is marked as deprecated. The
    string should provide some clear information about what to use instead. */

    /* provided func */
    void (*init)(MachineState *state);
    void (*reset)(MachineState *state);
    void (*wakeup)(MachineState *state);
    void (*hot_add_cpu)(MachineState *state, const int64_t id, Error **errp);
/**
 *    @kvm_type:
 *    Return the type of KVM corresponding to the kvm-type string option or
 *    computed based on other criteria such as the host kernel capabilities.
 */
    int (*kvm_type)(MachineState *machine, const char *arg);
    
/**
 *    @smp_parse:
 *    The function pointer to hook different machine specific functions for
 *    parsing "smp-opts" from QemuOpts to MachineState::CpuTopology and more
 *    machine specific topology fields, such as smp_dies for PCMachine.
 */
    void (*smp_parse)(MachineState *ms, QemuOpts *opts);

    BlockInterfaceType block_default_type;
    int units_per_default_bus;
    int max_cpus; /* maximum number of CPUs supported. Default: 1 */
    int min_cpus; /* minimum number of CPUs supported. Default: 1 */
    int default_cpus; /* number of CPUs instantiated if none are specified. Default: 1 */
    unsigned int no_serial:1,
        no_parallel:1,
        no_floppy:1,
        no_cdrom:1,
        no_sdcard:1,
        pci_allow_0_address:1,
        legacy_fw_cfg_order:1;
    bool is_default; /* If true QEMU will use this machine by default if no '-M' option is given. */
    const char *default_machine_opts;
    const char *default_boot_order;
    const char *default_display;
    GPtrArray *compat_props;
    const char *hw_version; /* Value of QEMU_VERSION when the machine was added to QEMU. */
    ram_addr_t default_ram_size;
    const char *default_cpu_type; /* specifies default CPU_TYPE, which will be used for parsing target specific features and for creating CPUs if CPU name wasn't provided explicitly at CLI */
    bool default_kernel_irqchip_split;
    bool option_rom_has_mr;
    bool rom_file_has_mr;
    int minimum_page_bits; /* If non-zero, the board promises never to create a CPU with a page size smaller than this */
    bool has_hotpluggable_cpus; /* If true, board supports CPUs creation with -device/device_add. */
    bool ignore_memory_transaction_failures; /* If this is flag is true then the CPU will ignore memory transaction failures which should cause the CPU to take an exception due to an access to an unassigned physical address */
    int numa_mem_align_shift;
    const char **valid_cpu_types;
    strList *allowed_dynamic_sysbus_devices;
    bool auto_enable_numa_with_memhp;
    bool auto_enable_numa_with_memdev;
    void (*numa_auto_assign_ram)(MachineClass *mc, NodeInfo *nodes,
                                 int nb_nodes, ram_addr_t size);
    bool ignore_boot_device_suffixes;
    bool smbus_no_migration_support;
    bool nvdimm_supported;
    bool numa_mem_supported; /* true if '--numa node.mem' option is supported  */
    bool auto_enable_numa;
    const char *default_ram_id; /* Specifies inital RAM MemoryRegion name */

/**
 * 	  @hotplug_allowed:
 *    If the hook is provided, then it'll be called for each device
 *    hotplug to check whether the device hotplug is allowed.  Return
 *    true to grant allowance or false to reject the hotplug.  When
 *    false is returned, an error must be set to show the reason of
 *    the rejection.  If the hook is not provided, all hotplug will be
 *    allowed.
 */
    HotplugHandler *(*get_hotplug_handler)(MachineState *machine,
                                           DeviceState *dev);
    
/**
 *    @get_hotplug_handler: this function is called during bus-less
 *    device hotplug. If defined it returns pointer to an instance
 *    of HotplugHandler object, which handles hotplug operation
 *    for a given @dev. It may return NULL if @dev doesn't require
 *    any actions to be performed by hotplug handler.
 */
    bool (*hotplug_allowed)(MachineState *state, DeviceState *dev,
                            Error **errp);
/**
 *    @cpu_index_to_instance_props:
 *    used to provide @cpu_index to socket/core/thread number mapping, allowing
 *    legacy code to perform maping from cpu_index to topology properties
 *    Returns: tuple of socket/core/thread ids given cpu_index belongs to.
 *    used to provide @cpu_index to socket number mapping, allowing
 *    a machine to group CPU threads belonging to the same socket/package
 *    Returns: socket number given cpu_index belongs to.
 */
    CpuInstanceProperties (*cpu_index_to_instance_props)(MachineState *machine,
                                                         unsigned cpu_index);
    
/**
 *    @possible_cpu_arch_ids:
 *    Returns an array of @CPUArchId architecture-dependent CPU IDs
 *    which includes CPU IDs for present and possible to hotplug CPUs.
 *    Caller is responsible for freeing returned list.
 */
    const CPUArchIdList *(*possible_cpu_arch_ids)(MachineState *machine);
    
/**
 *    @get_default_cpu_node_id:
 *    returns default board specific node_id value for CPU slot specified by
 *    index @idx in @ms->possible_cpus[]
 */
    int64_t (*get_default_cpu_node_id)(const MachineState *ms, int idx);
    
/**
 *    @fixup_ram_size
 *    Amends user provided ram size (with -m option) using machine specific 
 *    algorithm. To be used by old machine types for compat purposes only.
 */  
    ram_addr_t (*fixup_ram_size)(ram_addr_t size);
};
```



### Glossary

* bus-less device: [Waiting to be added]
* numa: Non-uniform memory access
* kvm: Kernel-based Virtual Machine
* smp: Symmetric multiprocessing
