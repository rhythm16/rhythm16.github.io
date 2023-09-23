---
title: "Linux中斷子系統簡介(1): 中斷處理流程的建立"
date: 2023-09-23 15:50:15
summary:
tags:
  - Linux
  - interrupt
  - ARMv8
categories: technical
---

> Linux版本：v6.0
>
> 處理器架構：ARMv8

## 前言

這幾週花了不少時間在看Linux到底是怎麼處理中斷的，在閱讀網路上前輩們介紹Linux中斷的blogs時總覺得細節都講得不錯，但大方向好難掌握住，我也是真的下去trace一遍才比較有感覺。這篇文章希望用另一個角度，以***最快的速度***帶過一遍中斷處理流程的建立與執行，達到初步理解整體架構的目的。我會先講解Linux處理中斷的一些關鍵概念，再使用一個真實的例子來說明這些概念怎麼對應到核心中真實的code。

這裡使用的例子是ARM64架構配合一個GICv2的中斷控制器建立IPI的情況，為了不讓整體描述過於發散&過長導致理解混亂，這篇用了很case specific的方式來講述，從而有些細節不免被省略，文章最後會放一些沒有提到的概念，供各位深入研究。

> 因為寫起來過長，文章計畫分成 中斷處理流程建立 與 中斷處理流程執行 兩篇

## 作業系統中斷極簡介紹

在說明Linux如何處理中斷之前，當然要知道中斷是個什麼東西。中斷基本上來說就是CPU核心的一個針腳被外界改變電位，內部電路感知之後開始相應的處理來響應這件事。在現代大部分的電腦系統中，有各式各樣的設備需要中斷CPU的執行，像是網卡收發封包**、**鍵盤輸入**、**甚至是其他的CPU核心想要求某個特定的CPU核心處理某些事情等等，中斷控制器(interrupt controller)就是替CPU核心們管理這些的一個硬體設備，CPU可以藉由和中斷控制器溝通來彈性的管理各種系統上的中斷:

- 開關各個中斷

- 調整各個中斷要送達哪一個CPU核心

- 決定各個中斷的優先級

- 觸發軟體中斷

等等

另外，在處理中斷時，CPU核心也要和中斷控制器進行溝通，以下是處理中斷時的一個典型步驟:

1. 中斷控制器感知到外界第21號設備要中斷CPU核心

2. 中斷控制器觀察到21號設備先前被設定成要送達0號CPU核心

3. 中斷控制器把”21”這個數字存入代表目前中斷的暫存器中 (這個暫存器在中斷控制器中，不是CPU暫存器)

4. 中斷控制器藉由改變與CPU0的中斷針腳的電位通知CPU0有中斷到達

5. CPU0放下手邊工作，讀取中斷控制器代表目前中斷的暫存器，讀到21，了解第21號設備需要服務

6. CPU0處理21號中斷

7. 完成之後，CPU0寫入另一個中斷控制器的暫存器(EOI)，來告知中斷控制器它已完成此次中斷的處理

## Linux核心的中斷管理與資料結構

### Linux IRQ number & Hardware IRQ number

Linux在管理系統上的各式中斷時，會為每一個中斷分配一個屬於自己的IRQ號碼，許多人稱Linux IRQ number。該數字用於區分系統中所有的中斷，在`/proc/interrupts` 看到的中斷號碼就是Linux IRQ number。Hardware IRQ number則是另一個概念，中斷控制器在前面例子中的”21”號，就是hardware IRQ number，是CPU向中斷控制器詢問哪個中斷觸發了，所會得到的答案。

你可能會想問為什麼要有Linux IRQ number呢? 全部都用hardware IRQ number不好嗎?

答案是: 的確，如果能都使用hardware IRQ number的話最好，但是隨著電腦系統越來越複雜，有些系統會出現不只一個中斷控制器的情況，這樣子問題就很明顯，不同中斷控制器有可能有相同的hardware IRQ number，但是接到完全不同的設備。於是乎，Linux就設計了hardware IRQ number轉換到Linux IRQ number的機制: IRQ Domain。

### IRQ Domain

Domain在數學中的定義是:

> Domain，定義域，函數自變數所有可取值的集合。 —wikipedia

而在這邊的意義就是一個hardware IRQ number → Linux IRQ number的轉換範疇，每一個中斷控制器會有一個屬於它的IRQ Domain，屬於該中斷控制器的hardware IRQ number就要使用該IRQ Domain來轉換成全局唯一的Linux IRQ number，藉此繼續處理該中斷。

實際轉換時，Linux提供兩種方式:

1. radix tree: 使用hardware IRQ number lookup radix tree來得到Linux IRQ number (對radix tree不熟目前只能這樣講 :P)

2. array lookup: 以hardware IRQ number作為array index取得Linux IRQ number

以下我會把轉換的資料結構叫做"revmap”。

中斷控制器的驅動可以從兩種方式自由選用其一，radix tree適合不連續的hardware IRQ numbers。

```c
struct irq_domain {
        struct list_head link;              // 系統中各個irq_domain被串起來
        const char *name;
        const struct irq_domain_ops *ops;   // 該中斷控制器的callback functions
        [...]
        unsigned int revmap_size;           // revmap array的大小，0代表使用radix tree
        struct radix_tree_root revmap_tree; // radix tree
        [...]
        struct irq_data __rcu *revmap[];    // array
};
```

### IRQ Desc & IRQ Data & IRQ Action & 中斷處理函式們

在`irq_domain` 可以看到`revmap[]` 其實並不是單純存著Linux IRQ number (int)，而是`irq_data` 的位址。每一個中斷初始化時都會分配一個`irq_desc` 儲存這個中斷的metadata，而`irq_data` 是`irq_desc` 的一部分，一個中斷對應到一個`irq_desc` 和`irq_data` 。Linux IRQ number → Hardware IRQ number的轉換(`irq`, `hwirq`)，以及其中斷處理函式 (`handle_irq`)均可以在這裡找到。`irqaction` 則是`irq_desc`中的一個鏈表，每一個`irqaction` 都有一個handler(`irqaction→handler`)，存著該Linux IRQ number的中斷處理函式。

```c
struct irq_data {
        u32                     mask;
        unsigned int            irq;            // Linux IRQ number
        unsigned long           hwirq;          // Hardware IRQ number
        struct irq_common_data  *common;
        struct irq_chip         *chip;
        struct irq_domain       *domain;        // 所屬的irq domain
#ifdef  CONFIG_IRQ_DOMAIN_HIERARCHY
        struct irq_data         *parent_data;
#endif
        void                    *chip_data;
};

struct irq_desc {
        struct irq_common_data  irq_common_data;
        struct irq_data         irq_data;       // the embedded irq_data
        unsigned int __percpu   *kstat_irqs;
        irq_flow_handler_t      handle_irq;     // 中斷處理函式
        struct irqaction        *action;        /* IRQ action list */
        [...]
} ____cacheline_internodealigned_in_smp;

struct irqaction {
        irq_handler_t           handler;        // 中斷處理函式
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

以下是這四個資料結構的連結方式:

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

對於一個中斷來說，已經有`irq_desc→handle_irq` 了，為什麼還要有`irqaction` 鏈表中的handler?

`irq_desc→handle_irq` 主要負責該IRQ的flow control和中斷控制器互動，例如mask unmask, ACK(通知控制器該中斷已正被處理), EOI(通知控制器中斷已完成處理)等等操作，而`irqaction→handler` 則是device specific的中斷處理，例如和網卡**、**硬碟互動等等。通常`irqaction→handler` 由各設備的驅動程式提供，`irq_desc→handle_irq` 則是由中斷控制器驅動設置，基本上是`handle_*` 家族函式，如

- `handle_level_irq`

- `handle_edge_irq`

- `handle_fasteoi_irq`

- `handle_percpu_devid_irq`

有人會把上述`irq_desc→handle_irq` 叫做這個IRQ的high level irq event handler。

除此之外，中斷控制器初始化時還須提供一個函式，來執行從中斷控制器讀取hardware IRQ number，利用irq domain定位正確`irq_desc` 來處理中斷等等更加底層的操作。在我們的ARM64例子中，設定的方式是GICv2初始化時會讓`handle_arch_irq`這個函式指標指到`gic_handle_irq` 來處理中斷。

## Linux中斷處理流程

換一個方式，利用簡化的Linux中斷處理流程來理解前面介紹的各個資料結構和各個中斷處理函式:

1. 中斷發生，exception vector進行中斷現場保存，呼叫`handle_arch_irq` (function pointer) 指向的函式

2. 該函式讀取中斷控制器的暫存器得知hardware IRQ number，使用中斷控制器對應的`irq_domain`進行hardware IRQ number → Linux IRQ number轉換，並得到`irq_desc`

3. 呼叫`irq_desc→handle_irq` (high level irq event handler)

4. high level irq event handler依次呼叫`irq_desc->action` 鏈表中的`handler`

Linux在能夠處理中斷之前，要把這3個不同層級的handlers function pointer設定好:

### 全局的`handle_arch_irq`

這個函式由最底層負責中斷現場保存的程式呼叫，通常由中斷控制器初始化程式設定，所有中斷處理都會呼叫(全局唯一)

### `irq_desc→handle_irq`

此函式負責該中斷的flow control，ACK, EOI, etc. 一個中斷一個，也稱high level event handler

### `irqaction→handler`

`irq_desc→handle_irq` 會依次呼叫`irq_desc` 中`irqaction` 鏈表中的各個`handler` (一個中斷可以有多個)

## 實例: GICv2建立IPI處理流程

經過上面的描述之後我們來觀察GICv2本身和IPI(inter processor interrupt)中斷是如何初始化的，以下每個小節介紹中斷處理機制建立過程的一部分，各個小節按照執行順序編排，而因為有些函式做很多事情，所以會有些函式重複出現。

> 小提示: 以下列出的函式數量不少，實際把call graph畫出來會讓理解方便許多

### 1\. 設置全局irq handler `handle_arch_irq`

`start_kernel` → `init_IRQ` → `irqchip_init` → `of_irq_init`

`of_irq_init` 呼叫GICv2初始化的callback `gic_of_init` → `__gic_init_bases` → `set_irq_handler` 來設置全局irq handler。

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
                // 設置全局handle_arch_irq函式指標
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
        // 指定全局函式指標handle_arch_irq為gic_handle_irq
        handle_arch_irq = handle_irq;
        pr_info("Root IRQ handler: %ps\n", handle_irq);
        return 0;
}
```

### 2\. allocate屬於此中斷控制器的`irq_domain` 與revmap

同樣在`__gic_init_bases` ，`__gic_init_bases` → `gic_init_bases` → `irq_domain_create_linear` → `__irq_domain_add`

```c
static int gic_init_bases(struct gic_chip_data *gic,
                          struct fwnode_handle *handle)
{
        int gic_irqs, ret;

        [...]

        if (handle) {           /* DT/ACPI */
                // 為此GIC allocate irq_domain，還有負責轉換的revmap
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
        // 為此GIC allocate irq_domain，還有負責傳換的revmap
        // size == 0 代表使用 radix tree轉換， size != 0代表使用array轉換
        // GICv2 使用array轉換
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
        // size == 0 代表使用 radix tree， size != 0代表使用array轉換
        domain->revmap_size = size;
        [...]
        mutex_lock(&irq_domain_mutex);
        debugfs_add_domain_dir(domain);
        // 把這個irq_domain掛入全局irq_domain_list中
        list_add(&domain->link, &irq_domain_list);
        mutex_unlock(&irq_domain_mutex);
        [...]
        return domain;
}
```

### 3\. 利用IPI的hardware IRQ number來allocate對應數量的Linux IRQ number和 `irq_desc` 

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
        // 剛才allocate的domain的位址會被記錄在gic_data[0].domain中，所以傳入它
        // -1代表不特別指定要使用的Linux IRQ number
        // 8代表要allocate 8個Linux IRQ number & irq_desc
        // 猜測是因為GICv2最多只允許8個CPU，所以allocate 8個IPI中斷
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
                // 呼叫更底層的函式
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
                // 呼叫更底層的函式
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
        // allocated_irqs是一個全局bitmap，每個index記錄該Linux IRQ number是否
        // 已被使用
        // e.g. allocated_irqs[10] == 1 代表Linux IRQ number 10已被使用
        // 所以這邊找一塊連續尚未被使用的Linux IRQ numbers
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
        // 呼叫更底層的函式
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
        // 每個迴圈allocate單一一個irq_desc
        for (i = 0; i < cnt; i++) {
                const struct cpumask *mask = NULL;
                unsigned int flags = 0;

                [...]
                // 真正呼叫kzmalloc allocate irq_desc並填入fields
                // (不特別展開了)
                desc = alloc_desc(start + i, node, flags, mask, owner);
                if (!desc)
                        goto err;
                // 將剛allocate出的irq_desc用對應的Linux IRQ number插入全局
                // radix tree irq_desc_tree (不要和revmap搞混了)
                irq_insert_desc(start + i, desc);
                [...]
        }
        // 把我們開始使用的Linux IRQ numbers在bitmap中設成1
        bitmap_set(allocated_irqs, start, cnt);
        return start;

        [...]
}

static void irq_insert_desc(unsigned int irq, struct irq_desc *desc)
{
        radix_tree_insert(&irq_desc_tree, irq, desc);
}
```

### 4\. 使用GICv2的callback設置IPI的`irq_desc→handle_irq`

剛才3.只是allocate，還沒設置`irq_desc→handle_irq`中斷處理函式

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
                // 剛剛3.提到的allocate irq_desc & Linux IRQ numbers
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
        // 繼續處理
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
        // 直接呼叫irq_domain的alloc callback來繼續執行，
        // 會接到gic_irq_domain_alloc
        // 有興趣的讀者可以自行尋找domain->ops在哪裡被設置的
        // 提示: 這篇裡面找的到
        return domain->ops->alloc(domain, irq_base, nr_irqs, arg);
}

static int gic_irq_domain_alloc(struct irq_domain *domain, unsigned int virq,
                                unsigned int nr_irqs, void *arg)
{
        int i, ret;
        irq_hw_number_t hwirq;
        unsigned int type = IRQ_TYPE_NONE;
        struct irq_fwspec *fwspec = arg;
        // 取得hardware IRQ number
        // 以我們的例子GIC的IPI hardware IRQ number是從0~15，但在這邊我們
        // 只使用8個，所以是0~7
        ret = gic_irq_domain_translate(domain, fwspec, &hwirq, &type);
        if (ret)
                return ret;
        // 每一次迴圈幫一個IRQ設置其irq_desc->handle_irq
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
        // GICv2 hardware IRQ number 0-31是banked per PE，所以這樣區分2個case
        // 我們的case是0~7 也就是0 ... 31
        switch (hw) {
        case 0 ... 31:
                irq_set_percpu_devid(irq);
                // 將handler_percpu_devid_irq這個high level IRQ event handler
                // 設定成該irq_desc->handle_irq
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
        // 此函式真正設定irq_desc->handle_irq 
        // (desc->handler_irq = handler)
        // 不過也做了許多其他複雜的操作，所以不展開
        __irq_set_handler(virq, handler, 0, handler_name);
        [...]
}
```

### 5\. 建立hardware IRQ number → Linux IRQ number的轉換revmap

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
                // 剛剛3.提到的allocate irq_desc & Linux IRQ numbers
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
        // 剛剛4.提到的設置irq_desc->handle_irq
        ret = irq_domain_alloc_irqs_hierarchy(domain, virq, nr_irqs, arg);
        [...]
        // 每個迴圈為一個IRQ設置其revmap
        for (i = 0; i < nr_irqs; i++)
                irq_domain_insert_irq(virq + i);
        [...]
        return ret;
}

static void irq_domain_insert_irq(int virq)
{
        struct irq_data *data;
        // 此迴圈"應該"與chained interrupt controllers有關，在我們的例子可以假設
        // 只跑一次
        for (data = irq_get_irq_data(virq); data; data = data->parent_data) {
                struct irq_domain *domain = data->domain;

                domain->mapcount++;
                // 繼續處理
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
        // 使用array還是radix tree?
        // array (GICv2的case)
        if (hwirq < domain->revmap_size)
                // 把hardware IRQ number當作index，填入該IRQ的irq_data即可
                rcu_assign_pointer(domain->revmap[hwirq], irq_data);
        // radix tree
        else
                radix_tree_insert(&domain->revmap_tree, hwirq, irq_data);
        mutex_unlock(&domain->revmap_mutex);
}
```

### 6\. 使用`request_percpu_irq` allocate `irqaction` 並安裝到`irq_desc→action` 鏈表中

接著，`gic_smp_init` → `set_smp_ipi_range` 呼叫`request_percpu_irq` 來allocate `irqaction` 並安裝到`irq_desc→action`

```c
void __init set_smp_ipi_range(int ipi_base, int n)
{
        int i;

        WARN_ON(n < NR_IPI);
        nr_ipi = min(n, NR_IPI);
        // 每一個迴圈設定一個IRQ的irqaction
        for (i = 0; i < nr_ipi; i++) {
                int err;
                // 將ipi_handler這個函式設定到irqaction->handler並安裝到
                // irq_desc->action list
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
        // allocate一個irqaction出來
        action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
        if (!action)
                return -ENOMEM;
        // 設定其action->handler (ipi_handler)
        action->handler = handler;
        action->flags = flags | IRQF_PERCPU | IRQF_NO_SUSPEND;
        action->name = devname;
        action->percpu_dev_id = dev_id;

        [...]
        // 巨大的IRQ中斷子系統函式，做的其中一件
        // 重要的事就是將irq_desc->action接上這裡新allocate的irqaction
        retval = __setup_irq(irq, desc, action);

        [...]

        return retval;
}
```

## 未提到的中斷相關概念們

- threaded IRQs

- chained interrupt controllers

- nested interrupt controllers

- shared IRQs

- spurious interrupts

- softirqs

- tasklets

- workqueues

## 參考資料

- [中断子系统 - 蜗窝科技 (](http://www.wowotech.net/sort/irq_subsystem)[wowotech.net](wowotech.net)[)](http://www.wowotech.net/sort/irq_subsystem)

- [Understanding Linux Interrupt Subsystem - Priya Dixit, Samsung Semiconductor India Research](https://youtu.be/LOCsN3V1ECE?si=piviVroOdJ_UVgnl)

- [How Dealing with Modern Interrupt Architectures can Affect Your Sanity](https://youtu.be/YE8cRHVIM4E?si=ENdH1X5EEDCMxCwI)
