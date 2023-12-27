---
title: "Linux Interrupt Subsystem Intro(1): Interrupt Handling Initialization"
date: 2023-12-27 11:30:00
summary:
tags:
  - Linux
  - interrupt
  - ARMv8
categories: technical
---

> Linux version: v6.0
>
> Architecture: ARMv8

I have been digging into the details of interrupt processing in Linux in the past few weeks. Although there are quite some quality articles and videos explaining the interrupt subsystem that can be found on the internet, personally it felt hard to get the big picture. Only after I actually traced the code that everything came together. In this post I hope to give an overview of the interrupt architecture by quickly going through how Linux sets up and processes interrupts. First some concepts around interrupt processing is introduced, then a real example is given to show how the code works.

The example shown here is the ARM64 architecture with a GICv2 interrupt controller, setting up the IPI interrupt. To avoid the discussion from becoming too long and digressing, the explanation is going to be pretty specific to our example, hence some details would be omitted. There are a couple of related concepts at the end of the article for the interested readers to reference.

> The post became quite long as I progressed, so I decided to split it into two articles:
>
> “Interrupt Handling Initialization” and “Interrupt Handling Process”.

## Quick Intro to OS Interrupts

Needless to say, knowing that an interrupt is is necessary to understanding how Linux processes them. At a fundamental level, interrupts are just the outside world changing a CPU’s pin’s voltage. After this change of voltage, the internal circuit of the CPU detects and reacts to this change. In a modern computer, there are many peripheral devices that each need to interrupt the CPU to signal some events, or request some computation, e.g. the NIC receiving and transmitting packets, keyboard input, IPI’s, etc. An *Interrupt Controller* is the piece of hardware that helps manage these interrupts, typically CPUs can manage interrupts through the interrupt controller to:

- enable and disable each interrupt

- set the destination CPU of each interrupt (CPU affinity)

- set priorities of interrupts

- invoke a software interrupt

etc.

The CPU also communicates with the interrupt controller when processing interrupts, following is a typical series of steps involved in processing an interrupt:

1. The interrupt controller detects that peripheral device number 21 wishes to interrupt the CPU

2. The interrupt controller observes that device 21’s interrupt is set to be delivered to CPU 0

3. The interrupt controller writes the number “21” into the register corresponding to the current interrupt (this register is within the interrupt controller, not the CPU)

4. The interrupt controller changes the pin’s voltage connected to CPU 0 to inform an interrupt had arrived

5. CPU0 handles interrupt for device 21

6. After interrupt handling completion, CPU 0 writes to the EOI (End-Of-Interrupt) register in the interrupt controller, to inform the interrupt controller that it had completed handling the current interrupt

## Interrupt Management and Data Structures used in the Linux Kernel

### Linux IRQ number & Hardware IRQ number

Linux allocates an IRQ number for each interrupt in the system, which is called *Linux IRQ number* by many. It is used to identify the interrupts. Interrupt numbers shown in `/proc/interrupts` are Linux IRQ numbers. *Hardware IRQ numbers*, is another concept, the number “21” in the example above is a hardware IRQ number, it is the “hardware” interrupt number the CPU gets when it asks the interrupt controller for the current interrupt.

You might be wondering why Linux IRQ numbers are necessary, why not just use hardware IRQ numbers for everything?

The answer is yes it would be nice if hardware IRQ numbers are used everywhere, however, computers these days may be connected with more than one interrupt controller, and hardware IRQ numbers would not be enough to identify individual interrupts, as two controllers may have the same hardware IRQ number for two different devices. As a result, the concept of `IRQ Domains` is introduced to translate between hardware IRQ number and Linux IRQ numbers.

### IRQ Domain

Domain as defined in mathematics:

> Domain, the set of inputs accepted by the function. —wikipedia

In our case, it stands for the different ways for translating hardware IRQ numbers → Linux IRQ numbers. Each interrupt controller has its own IRQ domain, as each interrupt controller has its own way of translating its hardware IRQ numbers to Linux IRQ numbers. Hardware IRQ numbers for an interrupt controller should be translated to Linux IRQ numbers by the interrupt controller’s IRQ domain.

Linux supports two methods for actually translating the IRQ numbers:

1. radix tree: taking the hardware IRQ number to lookup a radix tree to retrieve the Linux IRQ number. (I am not familiar with radix trees :P)

2. array lookup: index an array with the hardware IRQ number to retrieve the Linux IRQ number

I will use the term “revmap” to refer to the data structure (radix tree or array) that does the translation.

The driver for the interrupt controller can choose one of the options as it sees fit, radix tree is more suitable for sparse hardware IRQ numbers though.

```c
struct irq_domain {
        struct list_head link;              // the irq_domains in the system is linked
        const char *name;
        const struct irq_domain_ops *ops;   // callback functions for the interrupt controller
        [...]
        unsigned int revmap_size;           // size of the revmap array, 0 stands for using radix tree
        struct radix_tree_root revmap_tree; // radix tree
        [...]
        struct irq_data __rcu *revmap[];    // array
};
```

### IRQ Desc & IRQ Data & IRQ Action & Interrupt Handlers

You can see that `revmap[]` in `irq_domain` does not store Linux IRQ numbers directly, but pointers to `irq_data` instead. Each interrupt is allocated an `irq_desc` as it gets initialized to store some metadata, and `irq_data` is embedded within `irq_desc` , therefore each interrupt corresponds to one `irq_desc` and `irq_data` . The Linux IRQ number (`irq`), hardware IRQ number `hwirq` , and the interrupt handling function (`handle_irq`) for the interrupt can all be found in its corresponding `irq_data`. `irqaction` is a linked list within `irq_desc` which stores a handler (`irqaction→handler`) in each node in the list.

```c
struct irq_data {
        u32                     mask;
        unsigned int            irq;            // Linux IRQ number
        unsigned long           hwirq;          // Hardware IRQ number
        struct irq_common_data  *common;
        struct irq_chip         *chip;
        struct irq_domain       *domain;        // the irq domain it belongs to
#ifdef  CONFIG_IRQ_DOMAIN_HIERARCHY
        struct irq_data         *parent_data;
#endif
        void                    *chip_data;
};

struct irq_desc {
        struct irq_common_data  irq_common_data;
        struct irq_data         irq_data;       // the embedded irq_data
        unsigned int __percpu   *kstat_irqs;
        irq_flow_handler_t      handle_irq;     // interrupt handler
        struct irqaction        *action;        /* IRQ action list */
        [...]
} ____cacheline_internodealigned_in_smp;

struct irqaction {
        irq_handler_t           handler;        // interrupt handler
        void                    *dev_id;
        void __percpu           *percpu_dev_id;
        struct irqaction        *next;
        irq_handler_t           thread_fn;
        struct task_struct      *thread;
        struct irqaction        *secondary;
        unsigned int            irq;            // Linux IRQ number
        unsigned int            flags;
        [...]
} ____cacheline_internodealigned_in_smp;
```

Here is the visualized connection of the data structures:

```markdown
┌────────────────────┐
│     irq_domain     │
│                    │
│ revmap             │
│                    │
└─┬─┬────────────────┘
  │ │
  │ │           ┌───────────────┐   ┌─────────────┐   ┌─────────────┐
  │ └──────────►│   irq_desc    │ ┌►│  irqaction  ├──►│  irqaction  │
  │             │               │ │ └──────┬──────┘   └──────┬──────┘
  │           ┌─┴────────────┐  │ │        │                 │
  │           │   irq_desc   │  │ │        ▼                 ▼
  │           │              │  │ │   ┌─────────┐       ┌─────────┐
  │           │ ┌──────────┐ │  │ │   │ handler │       │ handler │
  └──────────►│ │ irq_data │ │  ├─┘   └─────────┘       └─────────┘
              │ └──────────┘ ├──┘
              │              │   ┌─────────────┐   ┌─────────────┐
              │              ├──►│  irqaction  ├──►│  irqaction  │
              └──────────────┘   └──────┬──────┘   └──────┬──────┘
                                        │                 │
                                        ▼                 ▼
                                   ┌─────────┐       ┌─────────┐
                                   │ handler │       │ handler │
                                   └─────────┘       └─────────┘
```

For each interrupt, why do we need `irqaction→handler` if there is `irq_desc→handle_irq` already?

`irq_desc→handle_irq` is responsible for interrupt flow control and the interaction between the CPU and the interrupt controller, e.g. masking/unmasking interrupts, ACK (acknowledging the interrupt), EOI (signal end-of-interrupt to the interrupt controller), etc. `irqaction→handler` , on the other hand, `irqaction→handler` is responsible for device interaction, e.g. accessing the NIC, hard disk, etc. Normally, `irqaction→handler` is supplied by the device drivers, and `irq_desc_handle_irq` is set by the interrupt controller, here we name a few:

- `handle_level_irq`

- `handle_edge_irq`

- `handle_fasteoi_irq`

- `handle_percpu_devid_irq`

Some call these the IRQ’s *high level irq event handler.* Moreover, during initialization the interrupt controller has to provide another function for low level interrupt processing such as reading the hardware IRQ number, or lookup `irq_desc` for the current interrupt. This function should then be pointed to by the global function pointer `handle_arch_irq` . In our GICv2’s case, that function is `gic_handler_irq` , driver sets `handle_arch_irq` to the address of the function `gic_handle_irq` during initialization.

## Linux Interrupt Processing Flow

Here we go through a simplified interrupt processing flow to help understand all the data structures and interrupt handlers introduced above.

1. interrupt arrives, CPU jumps to the corresponding exception vector to save execution context, calls function pointed to by `handle_arch_irq`

2. the function reads hardware IRQ number from the interrupt controller, and translates hardware IRQ number to Linux IRQ number using the controller’s `irq_domain` , and retrieves the `irq_desc` of the interrupt

3. calls `irq_desc→handle_irq` (high level irq event handler)

4. high level irq event handler calls each `irq_desc→action→handler` in succession

Therefore, the initialization process must set up the three levels of interrupt handlers before interrupts can be handled:

### The global `handle_arch_irq`

Called by low level exception handling code, set up by the interrupt controller’s driver during controller initialization, this function is called each time an interrupt is being handled.

### `irq_desc→handle_irq`

Responsible for interrupt flow control, ACK, EOI, etc. one for each interrupt, also known as the high level event handler of the interrupt.

### `irqaction→handler`

`irq_desc→handle_irq` calls each `handler` in the linked list, there can be many `irqaction→handler` for each interrupt.

## Example: GICv2’s IPI Processing

Let’s now take a look at how GICv2 and its IPI (inter processor interrupt) are initialized. The following subsections each describes a step of the process, note that some functions appear in more than one subsection since they do a lot of things.

> It might be helpful to draw your own call graph for the functions mentioned below, there are a lot of them.

### 1\. Setting the global irq handler `handle_arch_irq`

`start_kernel` → `init_IRQ` → `irqchip_init` → `of_irq_init`

`of_irq_init` calls GICv2’s initialization callback `gic_of_init` → `__gic_init_bases` → `set_irq_handler` 

```c
static int __init __gic_init_bases(struct gic_chip_data *gic,
                                   struct fwnode_handle *handle)
{
        [...]

        if (gic == &gic_data[0]) {
                /*
                 * Initialize the CPU interface map to all CPUs.
                 * It will be refined as each CPU probes its ID.
                 * This is only necessary for the primary GIC.
                 */
                for (i = 0; i < NR_GIC_CPU_IF; i++)
                        gic_cpu_map[i] = 0xff;
                // sets the global handle_arch_irq
                set_handle_irq(gic_handle_irq);
                if (static_branch_likely(&supports_deactivate_key))
                        pr_info("GIC: Using split EOI/Deactivate mode\n");
        }
        
        ret = gic_init_bases(gic, handle);
        [...]
}

int __init set_handle_irq(void (*handle_irq)(struct pt_regs *))
{
        if (handle_arch_irq != default_handle_irq)
                return -EBUSY;
        // sets handle_arch_irq to the address of gic_handle_irq
        handle_arch_irq = handle_irq;
        pr_info("Root IRQ handler: %ps\n", handle_irq);
        return 0;
}
```

### 2\. Allocate the `irq_domain` and its revmap for the interrupt controller

similarly in `__gic_init_bases` , `__gic_init_bases` → `gic_init_bases` → `irq_domain_create_linear` → `__irq_domain_add`

```c
static int gic_init_bases(struct gic_chip_data *gic,
                          struct fwnode_handle *handle)
{
        int gic_irqs, ret;

        [...]

        if (handle) {           /* DT/ACPI */
                // allocate irq_domain and revmap for this GIC
                gic->domain = irq_domain_create_linear(handle, gic_irqs,
                                                       &gic_irq_domain_hierarchy_ops,
                                                       gic);
        } else {                /* Legacy support */
                [...]
        }

        [...]

        return ret;
}

static inline struct irq_domain *irq_domain_create_linear(struct fwnode_handle *fwnode,
                                         unsigned int size,
                                         const struct irq_domain_ops *ops,
                                         void *host_data)
{
        return __irq_domain_add(fwnode, size, size, 0, ops, host_data);
}

struct irq_domain *__irq_domain_add(struct fwnode_handle *fwnode, unsigned int size,
                                    irq_hw_number_t hwirq_max, int direct_max,
                                    const struct irq_domain_ops *ops,
                                    void *host_data)
{
        struct irqchip_fwid *fwid;
        struct irq_domain *domain;

        static atomic_t unknown_domains;

        [...]
        // allocate irq_domain and revmap for this GIC
        // size == 0 stands for using radix tree， size != 0 stands for using array
        // GICv2 uses array to translate
        domain = kzalloc_node(struct_size(domain, revmap, size),
                              GFP_KERNEL, of_node_to_nid(to_of_node(fwnode)));
        if (!domain)
                return NULL;

        [...]

        /* Fill structure */
        INIT_RADIX_TREE(&domain->revmap_tree, GFP_KERNEL);
        mutex_init(&domain->revmap_mutex);
        domain->ops = ops;
        domain->host_data = host_data;
        domain->hwirq_max = hwirq_max;

        if (direct_max) {
                domain->flags |= IRQ_DOMAIN_FLAG_NO_MAP;
        }
        // size == 0 stands for using radix tree， size != 0 stands for using array
        domain->revmap_size = size;
        [...]
        mutex_lock(&irq_domain_mutex);
        debugfs_add_domain_dir(domain);
        // insert this irq_domain to the global irq_domain_list
        list_add(&domain->link, &irq_domain_list);
        mutex_unlock(&irq_domain_mutex);
        [...]
        return domain;
}
```

### 3\. Allocate Linux IRQ numbers and `irq_desc`s by checking IPI’s hardware IRQ number

`__gic_init_bases` → `gic_smp_init` → `__irq_domain_alloc_irqs` → `irq_domain_alloc_descs` → `__irq_alloc_descs` → `alloc_descs` → `alloc_desc` & `irq_insert_desc`

```c
static __init void gic_smp_init(void)
{
        struct irq_fwspec sgi_fwspec = {
                .fwnode         = gic_data[0].domain->fwnode,
                .param_count    = 1,
        };
        int base_sgi;

        cpuhp_setup_state_nocalls(CPUHP_AP_IRQ_GIC_STARTING,
                                  "irqchip/arm/gic:starting",
                                  gic_starting_cpu, NULL);
        // gic_data[0].domain stores the address of the irq_domain previously
        // allocated
        // -1 stands for not specifying a Linux IRQ number to use
        // 8 is the number of Linux IRQ numbers and irq_descs it wishes to allocate
        // the reason for 8 is probably because GICv2 supports at most 8 CPUs,
        // so it allocates 8 IPI interrupts here
        base_sgi = __irq_domain_alloc_irqs(gic_data[0].domain, -1, 8,
                                           NUMA_NO_NODE, &sgi_fwspec,
                                           false, NULL);
        if (WARN_ON(base_sgi <= 0))
                return;

        set_smp_ipi_range(base_sgi, 8);
}

int __irq_domain_alloc_irqs(struct irq_domain *domain, int irq_base,
                            unsigned int nr_irqs, int node, void *arg,
                            bool realloc, const struct irq_affinity_desc *affinity)
{
        int i, ret, virq;

        [...]

        if (realloc && irq_base >= 0) {
                virq = irq_base;
        } else {
                // calling lower-level function
                virq = irq_domain_alloc_descs(irq_base, nr_irqs, 0, node,
                                              affinity);
                if (virq < 0) {
                        pr_debug("cannot allocate IRQ(base %d, count %d)\n",
                                 irq_base, nr_irqs);
                        return virq;
                }
        }

        [...]
        return ret;
}

int irq_domain_alloc_descs(int virq, unsigned int cnt, irq_hw_number_t hwirq,
                           int node, const struct irq_affinity_desc *affinity)
{
        [...]
                // calling lower-level function
                virq = __irq_alloc_descs(virq, virq, cnt, node, THIS_MODULE,
                                         affinity);
        [...]
}

int __ref
__irq_alloc_descs(int irq, unsigned int from, unsigned int cnt, int node,
                  struct module *owner, const struct irq_affinity_desc *affinity)
{
        int start, ret;

        if (!cnt)
                return -EINVAL;

        if (irq >= 0) {
                if (from > irq)
                        return -EINVAL;
                from = irq;
        } else {
                /*
                 * For interrupts which are freely allocated the
                 * architecture can force a lower bound to the @from
                 * argument. x86 uses this to exclude the GSI space.
                 */
                from = arch_dynirq_lower_bound(from);
        }

        mutex_lock(&sparse_irq_lock);
        // allocated_irqs is a global bitmap recording the allocation status
        // for all Linux IRQ numbers,
        // e.g. allocated_irqs[10] == 1 stands for Linux IRQ number 10 is in use
        // here we look for a continuous range of unused Linux IRQ numbers
        start = bitmap_find_next_zero_area(allocated_irqs, IRQ_BITMAP_BITS,
                                           from, cnt, 0);
        ret = -EEXIST;
        if (irq >=0 && start != irq)
                goto unlock;

        if (start + cnt > nr_irqs) {
                ret = irq_expand_nr_irqs(start + cnt);
                if (ret)
                        goto unlock;
        }
        // calling lower-level function
        ret = alloc_descs(start, cnt, node, affinity, owner);
unlock:
        mutex_unlock(&sparse_irq_lock);
        return ret;
}

static int alloc_descs(unsigned int start, unsigned int cnt, int node,
                       const struct irq_affinity_desc *affinity,
                       struct module *owner)
{
        [...]
        // each loop allocates a single irq_desc
        for (i = 0; i < cnt; i++) {
                const struct cpumask *mask = NULL;
                unsigned int flags = 0;

                [...]
                // actually calls kzmalloc to allocate irq_desc and init some fields
                // we don't dive into this function for simplicity
                desc = alloc_desc(start + i, node, flags, mask, owner);
                if (!desc)
                        goto err;
                // insert the allocated irq_desc into the global radix tree
                // irq_desc_tree using Linux IRQ number as its key
                // (do not confuse this with revmap)
                irq_insert_desc(start + i, desc);
                [...]
        }
        // set the index used in allocated_irqs to 1
        bitmap_set(allocated_irqs, start, cnt);
        return start;

        [...]
}

static void irq_insert_desc(unsigned int irq, struct irq_desc *desc)
{
        radix_tree_insert(&irq_desc_tree, irq, desc);
}
```

### 4\. Set up IPI’s `irq_desc→handle_irq` in GICv2’s callback

The previous step only allocated `irq_desc` , next is to set `irq_desc→handle_irq`.

`__irq_domain_alloc_descs` → `irq_domain_alloc_irqs_hierarchy` → `gic_irq_domain_alloc` → `gic_irq_domain_map` → `irq_domain_set_info` → `__irq_set_handler` 

```c
int __irq_domain_alloc_irqs(struct irq_domain *domain, int irq_base,
                            unsigned int nr_irqs, int node, void *arg,
                            bool realloc, const struct irq_affinity_desc *affinity)
{
        [...]

        if (realloc && irq_base >= 0) {
                virq = irq_base;
        } else {
                // allocate irq_desc & Linux IRQ numbers, as mentioned in 3.
                virq = irq_domain_alloc_descs(irq_base, nr_irqs, 0, node,
                                              affinity);
                if (virq < 0) {
                        pr_debug("cannot allocate IRQ(base %d, count %d)\n",
                                 irq_base, nr_irqs);
                        return virq;
                }
        }

        [...]

        mutex_lock(&irq_domain_mutex);
        // continue processing
        ret = irq_domain_alloc_irqs_hierarchy(domain, virq, nr_irqs, arg);
        [...]
        return ret;
}

int irq_domain_alloc_irqs_hierarchy(struct irq_domain *domain,
                                    unsigned int irq_base,
                                    unsigned int nr_irqs, void *arg)
{
        if (!domain->ops->alloc) {
                pr_debug("domain->ops->alloc() is NULL\n");
                return -ENOSYS;
        }
        // call irq_domain's alloc callback,
        // in this case it's gic_irq_domain_alloc
        // you can try to find where in the code did domain->ops got
        // assigned, hint: it can be found within this post
        return domain->ops->alloc(domain, irq_base, nr_irqs, arg);
}

static int gic_irq_domain_alloc(struct irq_domain *domain, unsigned int virq,
                                unsigned int nr_irqs, void *arg)
{
        int i, ret;
        irq_hw_number_t hwirq;
        unsigned int type = IRQ_TYPE_NONE;
        struct irq_fwspec *fwspec = arg;
        // get hardware IRQ number,
        // in this case GIC's IPI IRQ number ranges from 0 to 15,
        // we only use 8 here so it is 0-7
        ret = gic_irq_domain_translate(domain, fwspec, &hwirq, &type);
        if (ret)
                return ret;
        // each loop sets irq_desc->handle_irq for one IRQ
        for (i = 0; i < nr_irqs; i++) {
                ret = gic_irq_domain_map(domain, virq + i, hwirq + i);
                if (ret)
                        return ret;
        }

        return 0;
}

static int gic_irq_domain_map(struct irq_domain *d, unsigned int irq,
                                irq_hw_number_t hw)
{
        struct gic_chip_data *gic = d->host_data;
        struct irq_data *irqd = irq_desc_get_irq_data(irq_to_desc(irq));
        const struct irq_chip *chip;

        chip = (static_branch_likely(&supports_deactivate_key) &&
                gic == &gic_data[0]) ? &gic_chip_mode1 : &gic_chip;
        // two switch cases here because numbers 0-31 of GICv2's hardware
        // IRQ number are banked per PE, therefore they are processed
        // differently from the rest. In our case we are using 0-7 so
        // the first case is run
        switch (hw) {
        case 0 ... 31:
                irq_set_percpu_devid(irq);
                // set the high level IRQ event handler to the function
                // handle_percpu_devid_irq
                irq_domain_set_info(d, irq, hw, chip, d->host_data,
                                    handle_percpu_devid_irq, NULL, NULL);
                break;
        default:
                irq_domain_set_info(d, irq, hw, chip, d->host_data,
                                    handle_fasteoi_irq, NULL, NULL);
                irq_set_probe(irq);
                irqd_set_single_target(irqd);
                break;
        }

        /* Prevents SW retriggers which mess up the ACK/EOI ordering */
        irqd_set_handle_enforce_irqctx(irqd);
        return 0;
}

void irq_domain_set_info(struct irq_domain *domain, unsigned int virq,
                         irq_hw_number_t hwirq, const struct irq_chip *chip,
                         void *chip_data, irq_flow_handler_t handler,
                         void *handler_data, const char *handler_name)
{
        [...]
        // this is the function that actually assigns irq_desc->handle_irq
        // (desc->handle_irq = handler)
        // however it is not expanded since it does a lot of other
        // complicated work
        __irq_set_handler(virq, handler, 0, handler_name);
        [...]
}
```

### 5\. Create revmap for translating hardware IRQ number → Linux IRQ number

`__irq_domain_alloc_descs` → `irq_domain_insert_irq` → `irq_domain_set_mapping`

```c
int __irq_domain_alloc_irqs(struct irq_domain *domain, int irq_base,
                            unsigned int nr_irqs, int node, void *arg,
                            bool realloc, const struct irq_affinity_desc *affinity)
{
        [...]

        if (realloc && irq_base >= 0) {
                virq = irq_base;
        } else {
                // allocate irq_desc & Linux IRQ numbers, as mentioned in 3.
                virq = irq_domain_alloc_descs(irq_base, nr_irqs, 0, node,
                                              affinity);
                if (virq < 0) {
                        pr_debug("cannot allocate IRQ(base %d, count %d)\n",
                                 irq_base, nr_irqs);
                        return virq;
                }
        }

        [...]

        mutex_lock(&irq_domain_mutex);
        // set irq_desc->handle_irq, as mentioned in 4.
        ret = irq_domain_alloc_irqs_hierarchy(domain, virq, nr_irqs, arg);
        [...]
        // each loop set ups revmap for one IRQ
        for (i = 0; i < nr_irqs; i++)
                irq_domain_insert_irq(virq + i);
        [...]
        return ret;
}

static void irq_domain_insert_irq(int virq)
{
        struct irq_data *data;
        // in my understanding this loop is related to chained interrupt
        // controllers, hence it is run only once in our case
        for (data = irq_get_irq_data(virq); data; data = data->parent_data) {
                struct irq_domain *domain = data->domain;

                domain->mapcount++;
                // keep on processing
                irq_domain_set_mapping(domain, data->hwirq, data);

                /* If not already assigned, give the domain the chip's name */
                if (!domain->name && data->chip)
                        domain->name = data->chip->name;
        }

        irq_clear_status_flags(virq, IRQ_NOREQUEST);
}

static void irq_domain_set_mapping(struct irq_domain *domain,
                                   irq_hw_number_t hwirq,
                                   struct irq_data *irq_data)
{
        if (irq_domain_is_nomap(domain))
                return;

        mutex_lock(&domain->revmap_mutex);
        // use array or radix tree?
        // array (in GICv2's case)
        if (hwirq < domain->revmap_size)
                // simply assign the irq_data of IRQ into revmap array
                // with hardware IRQ number as the index
                rcu_assign_pointer(domain->revmap[hwirq], irq_data);
        // radix tree
        else
                radix_tree_insert(&domain->revmap_tree, hwirq, irq_data);
        mutex_unlock(&domain->revmap_mutex);
}
```

### 6\. allocate and insert `irqaction` in linked list `irq_desc→action` , in `request_percpu_irq`

Next, `gic_smp_init` → `set_smp_ipi_range` calls`request_percpu_irq` to allocate `irqaction` and insert it in `irq_desc→action`

```c
void __init set_smp_ipi_range(int ipi_base, int n)
{
        int i;

        WARN_ON(n < NR_IPI);
        nr_ipi = min(n, NR_IPI);
        // each loop sets up an irqaction for an IRQ
        for (i = 0; i < nr_ipi; i++) {
                int err;
                // set ipi_handler as irqaction->handler and insert it
                // in irq_desc->action
                err = request_percpu_irq(ipi_base + i, ipi_handler,
                                         "IPI", &cpu_number);
                WARN_ON(err);

                ipi_desc[i] = irq_to_desc(ipi_base + i);
                irq_set_status_flags(ipi_base + i, IRQ_HIDDEN);
        }

        [...]
}

static inline int __must_check
request_percpu_irq(unsigned int irq, irq_handler_t handler,
                   const char *devname, void __percpu *percpu_dev_id)
{
        return __request_percpu_irq(irq, handler, 0,
                                    devname, percpu_dev_id);
}

int __request_percpu_irq(unsigned int irq, irq_handler_t handler,
                         unsigned long flags, const char *devname,
                         void __percpu *dev_id)
{
        [...]

        desc = irq_to_desc(irq);
        [...]
        // allocate an irqaction
        action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
        if (!action)
                return -ENOMEM;
        // set its action->handler (ipi_handler)
        action->handler = handler;
        action->flags = flags | IRQF_PERCPU | IRQF_NO_SUSPEND;
        action->name = devname;
        action->percpu_dev_id = dev_id;

        [...]
        // this is a big function, responsible for inserting the newly
        // allocated irqaction into irq_desc->action
        retval = __setup_irq(irq, desc, action);

        [...]

        return retval;
}
```

## Linux Interrupt Handling Concepts not Mentioned

- threaded IRQs

- chained interrupt controllers

- nested interrupt controllers

- shared IRQs

- spurious interrupts

- softirqs

- tasklets

- workqueues

## References

- [中断子系统 - 蜗窝科技 (](http://www.wowotech.net/sort/irq_subsystem)[wowotech.net](wowotech.net)[)](http://www.wowotech.net/sort/irq_subsystem)

- [Understanding Linux Interrupt Subsystem - Priya Dixit, Samsung Semiconductor India Research](https://youtu.be/LOCsN3V1ECE?si=piviVroOdJ_UVgnl)

- [How Dealing with Modern Interrupt Architectures can Affect Your Sanity](https://youtu.be/YE8cRHVIM4E?si=ENdH1X5EEDCMxCwI)
