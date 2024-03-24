---
title: "Linux中斷子系統簡介(2): 中斷處理流程執行"
date: 2024-03-04 15:50:15
summary:
tags:
  - Linux
  - interrupt
  - ARMv8
categories: technical
---

> Linux版本：v6.0
> 處理器架構：ARMv8

這篇接續[Linux中斷子系統簡介(1): 中斷處理流程的建立](../interrupt1/)

## 前言

距離前篇已經過了快要半年... 中間跑去研究其他的事了所以才一直留著這個坑沒填，現在回來看已經有點生疏了qaq，果然有想寫的東西就要趕快啊

上次說明了Linux中斷處理的基本架構，還有初始化程式做的準備，包括設定`handle_arch_irq`，`irq_desc→handle_irq`，`irqaction→handler` 這幾個中斷處理函式。這次就來看一下他們在真的中斷來臨的時候如何被呼叫到的吧。

## 中斷處理流程執行

一個CPU在接收另一個CPU傳來的的IPI時，PC會跳到exception vector的IRQ vector執行:

```c
// arch/arm64/kernel/entry.S

/*
 * Exception vectors.
 */
        .pushsection ".entry.text", "ax"

        .align  11
SYM_CODE_START(vectors)
        kernel_ventry   1, t, 64, sync          // Synchronous EL1t
        kernel_ventry   1, t, 64, irq           // IRQ EL1t
        kernel_ventry   1, t, 64, fiq           // FIQ EL1t
        kernel_ventry   1, t, 64, error         // Error EL1t

        kernel_ventry   1, h, 64, sync          // Synchronous EL1h
        kernel_ventry   1, h, 64, irq           // IRQ EL1h
        kernel_ventry   1, h, 64, fiq           // FIQ EL1h
        kernel_ventry   1, h, 64, error         // Error EL1h

        kernel_ventry   0, t, 64, sync          // Synchronous 64-bit EL0
        kernel_ventry   0, t, 64, irq           // IRQ 64-bit EL0
        kernel_ventry   0, t, 64, fiq           // FIQ 64-bit EL0
        kernel_ventry   0, t, 64, error         // Error 64-bit EL0

        kernel_ventry   0, t, 32, sync          // Synchronous 32-bit EL0
        kernel_ventry   0, t, 32, irq           // IRQ 32-bit EL0
        kernel_ventry   0, t, 32, fiq           // FIQ 32-bit EL0
        kernel_ventry   0, t, 32, error         // Error 32-bit EL0
SYM_CODE_END(vectors)
```

`kernel_ventry` 是一個assembly macro:

```c
        .macro kernel_ventry, el:req, ht:req, regsize:req, label:req
        .align 7

        [...] // 一些複雜的初始處理
        // 為中斷當下儲存的暫存器預留空間
        sub     sp, sp, #PT_REGS_SIZE

#ifdef CONFIG_VMAP_STACK

        [...] // SP 溢出檢查與處理

#endif
        b       el\el\ht\()_\regsize\()_\label
.org .Lventry_start\@ + 128     // Did we overflow the ventry slot?
        .endm
```

可以看到最後就是一個branch指令，假設原本CPU處於AArch64 EL0，

`b       el\el\ht\()_\regsize\()_\label` 就會展開成

`b        el0t_64_irq`

`el0t_64_irq` 被定義在同個檔案中，也是用一些macro生成的:

```c
// arch/arm64/kernel/entry.S

        .macro entry_handler el:req, ht:req, regsize:req, label:req
SYM_CODE_START_LOCAL(el\el\ht\()_\regsize\()_\label)
        kernel_entry \el, \regsize
        mov     x0, sp
        bl      el\el\ht\()_\regsize\()_\label\()_handler
        .if \el == 0
        b       ret_to_user
        .else
        b       ret_to_kernel
        .endif
SYM_CODE_END(el\el\ht\()_\regsize\()_\label)
        .endm

/*
 * Early exception handlers
 */
        entry_handler   1, t, 64, sync
        entry_handler   1, t, 64, irq
        entry_handler   1, t, 64, fiq
        entry_handler   1, t, 64, error

        entry_handler   1, h, 64, sync
        entry_handler   1, h, 64, irq
        entry_handler   1, h, 64, fiq
        entry_handler   1, h, 64, error

        entry_handler   0, t, 64, sync
        entry_handler   0, t, 64, irq
        entry_handler   0, t, 64, fiq
        entry_handler   0, t, 64, error

        entry_handler   0, t, 32, sync
        entry_handler   0, t, 32, irq
        entry_handler   0, t, 32, fiq
        entry_handler   0, t, 32, error
```

我們要看的是`entry_handler   0, t, 64, irq` ，展開後會是

```c
SYM_CODE_START_LOCAL(el0t_64_irq)
        kernel_entry 0, 64
        mov     x0, sp
        bl      el0t_64_irq_handler
        b       ret_to_user
        .endif
SYM_CODE_END(el0t_64_irq)
```

繼續(請看註解):

```c
// arch/arm64/kernel/entry-common.c
// 請從這個block最下面開始看
// 這邊就是一路從下面往上呼叫，再到do_interrupt_handler

static void do_interrupt_handler(struct pt_regs *regs,
                                 void (*handler)(struct pt_regs *))
{
        struct pt_regs *old_regs = set_irq_regs(regs);

        // 視情況改變stack，但最終都會呼叫handle_arch_irq (gic_handle_irq)
        if (on_thread_stack())
                call_on_irq_stack(regs, handler);
        else
                handler(regs);

        set_irq_regs(old_regs);
}

static void noinstr el0_interrupt(struct pt_regs *regs,
                                  void (*handler)(struct pt_regs *))
{
        enter_from_user_mode(regs);

        write_sysreg(DAIF_PROCCTX_NOIRQ, daif);

        if (regs->pc & BIT(55))
                arm64_apply_bp_hardening();

        irq_enter_rcu();
        // 重點在這
        do_interrupt_handler(regs, handler);
        irq_exit_rcu();

        exit_to_user_mode(regs);
}

static void noinstr __el0_irq_handler_common(struct pt_regs *regs)
{
        // 使用我們上次設定的handle_arch_irq!
        el0_interrupt(regs, handle_arch_irq);
}

asmlinkage void noinstr el0t_64_irq_handler(struct pt_regs *regs)
{
        __el0_irq_handler_common(regs);
}
```

那來看`gic_handle_irq`:

```c
static void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
        u32 irqstat, irqnr;
        struct gic_chip_data *gic = &gic_data[0];
        void __iomem *cpu_base = gic_data_cpu_base(gic);

        do {
                // 讀取GICC_IAR
                irqstat = readl_relaxed(cpu_base + GIC_CPU_INTACK);
                // 取出顯示hw irq number的部份
                irqnr = irqstat & GICC_IAR_INT_ID_MASK;
                // 沒有中斷等待的話GIC會把irqnr設成1023，所以迴圈會一直處理直到沒有中斷
                if (unlikely(irqnr >= 1020))
                        break;
                // 略
                if (static_branch_likely(&supports_deactivate_key))
                        writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);
                isb();

                /*
                 * Ensure any shared data written by the CPU sending the IPI
                 * is read after we've read the ACK register on the GIC.
                 *
                 * Pairs with the write barrier in gic_ipi_send_mask
                 */
                // inqnr <= 15 代表IPI (GIC中叫做SGI)
                if (irqnr <= 15) {
                        smp_rmb();

                        /*
                         * The GIC encodes the source CPU in GICC_IAR,
                         * leading to the deactivation to fail if not
                         * written back as is to GICC_EOI.  Stash the INTID
                         * away for gic_eoi_irq() to write back.  This only
                         * works because we don't nest SGIs...
                         */
                        // 存下整個GICC_IAR的直，deactivation時寫回EOI
                        this_cpu_write(sgi_intid, irqstat);
                }

                generic_handle_domain_irq(gic->domain, irqnr);
        } while (1);
}

/**
 * generic_handle_domain_irq - Invoke the handler for a HW irq belonging
 *                             to a domain.
 * @domain:     The domain where to perform the lookup
 * @hwirq:      The HW irq number to convert to a logical one
 *
 * Returns:     0 on success, or -EINVAL if conversion has failed
 *
 *              This function must be called from an IRQ context with irq regs
 *              initialized.
 */
int generic_handle_domain_irq(struct irq_domain *domain, unsigned int hwirq)
{
        // irq_resolve_mapping使用domain->revmap把hardware IRQ number轉換成
        // Linux IRQ number
        return handle_irq_desc(irq_resolve_mapping(domain, hwirq));
}

int handle_irq_desc(struct irq_desc *desc)
{
        [...]
        generic_handle_irq_desc(desc);
        return 0;
}

static inline void generic_handle_irq_desc(struct irq_desc *desc)  
{  
        desc->handle_irq(desc);
}
```

連結到上一篇，`desc→handle_irq` 指向`handle_percpu_devid_irq` 

```c
/**
 * handle_percpu_devid_irq - Per CPU local irq handler with per cpu dev ids
 * @desc:       the interrupt description structure for this irq
 *
 * Per CPU interrupts on SMP machines without locking requirements. Same as
 * handle_percpu_irq() above but with the following extras:
 *
 * action->percpu_dev_id is a pointer to percpu variables which
 * contain the real device id for the cpu on which this handler is
 * called
 */
void handle_percpu_devid_irq(struct irq_desc *desc)
{
        struct irq_chip *chip = irq_desc_get_chip(desc);
        struct irqaction *action = desc->action;
        unsigned int irq = irq_desc_get_irq(desc);
        irqreturn_t res;

        /*
         * PER CPU interrupts are not serialized. Do not touch
         * desc->tot_count.
         */
        __kstat_incr_irqs_this_cpu(desc);

        // GIC沒有irq_ack (讀取hwirq時就算ack該interrupt了)
        if (chip->irq_ack)
                chip->irq_ack(&desc->irq_data);

        if (likely(action)) {
                trace_irq_handler_entry(irq, action);
                // 呼叫上次用request_percpu_irq註冊的handler (ipi_handler)
                res = action->handler(irq, raw_cpu_ptr(action->percpu_dev_id));
                trace_irq_handler_exit(irq, action, res);
        } else {
                unsigned int cpu = smp_processor_id();
                bool enabled = cpumask_test_cpu(cpu, desc->percpu_enabled);

                if (enabled)
                        irq_percpu_disable(desc, cpu);

                pr_err_once("Spurious%s percpu IRQ%u on CPU%u\n",
                            enabled ? " and unmasked" : "", irq, cpu);
        }
        // 處理完畢，向GIC通知EOI (把hwirq入EOI暫存器)
        if (chip->irq_eoi)
                chip->irq_eoi(&desc->irq_data);
}
```

就是這樣啦\~ EZPZ, right?
