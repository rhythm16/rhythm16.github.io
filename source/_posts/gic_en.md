---
title: "Notes on GIC - Hardware Handling of Virtual Interrupt Direct Injection"
date: 2022-12-17 21:20:15
summary:
tags:
  - GIC
  - ARMv8
categories: technical
---

## Introduction

I have been reading ARM’s ”GICv3 and GICv4 Software Overview” for the past few weeks, it’s a nice introduction to GICv3, v4. ARM emitted some “special” use cases e.g. hardware with only one security state, and using the legacy mode to avoid this introductory book becoming too long. Reading this first makes it a lot easier to dive in to the official spec “Arm® Generic Interrupt Controller Architecture Specification GIC architecture version 3 and version 4” later on.

It’s way too easy to get tunnel-visioned with all the small details in these kinds of documents, you might feel like you understood all the details but after a while you realize you’re not getting the big picture. This is one of the motivations for this article, I would like to explain parts of the content in my own words to make sure I at least have a feel for the context.Therefore much assumptions (e.g. using the typical configurations) are made, and also much basics are skipped.

I created this visualized note with Heptabase to aid my learning, you are welcome to check it out:

<https://app.heptabase.com/w/d4dfaf701a143f308454cb962ba9963472a27873bc9a2b2779e89d9430a972ac>

## Disclaimer

The following will be about virtual interrupts, so Collection Table, Collection ID are skipped.

Furthermore, the steps which hypervisor should take are necessary but not sufficient, I’m just listing out what must be done architecurally.

## Steps Involved in Direct Injection of Virtual Interrupts

Start from a peripheral sending an LPI (Local Peripheral Interrupt):

1. The peripheral knows the address of the ITS (Interrupt Translation Service) on the bus, then writes the Device ID and Event ID to the `GITS_TRANSLALTOR` register to send an interrupt.

2. After sensing the LPI, ITS reads one of `GITS_BASER[0..7]` whose `Type` field is Device Table to get the address of the Device Table. The Device Table is a table in memory which records the translation of Device IDs to Interrupt Translation Tables’ base addresses.

3. The hardware reads the address of the Interrupt Translation Table corresponding to the Device ID written in.

4. The hardware reads the Interrupt Translation Table to look for the entry corresponding to the Event ID written in, and reads either (assume it is the second case going forward):

   1. a collection ID and an INTID

   2. a vINTID and a vPE ID, and optionally a doorbell pINTID

5. The hardware reads one of `GITS_BASER[0..7]` whose `Type` field is vPE Table to get the address of the vPE Table.

6. The hardware looks into the vPE Table for the entry corresponding to the vPE ID (from step 4) to get a target Redistributor and a virtual LPI Pending Table Address.

7. The hardware reads the `GICR_VPENDBASER` register of the target Redistributor and compares it with the virtual LPI Pending Table Address read from step 6, there are two possibilities (assume it is the first case going forward):

   1. it’s a match, go on to inject the virtual interrupt

   2. it’s not a match, inject physical interrupt with the pINTID if it is supplied in the 4’th step

8. The running vPE is directly interrupted with the virtual interrupt without exiting the VM, and starts running its interrupt service routine, it communicates directly with the Virtual CPU Interface, and ends the interrupt by writing to the `EOI` register directly.

Hypervisor software must initialize all kinds of structures before using virtual interrupt direct injection, including:

### Device Table

This is a global table, software must allocate memory for it and write the base address into `GITS_BASER[0..7]` whose `Type` is devicie table, then fill each Device ID’s entry with the corresponding Interrupt Translation Table base address.

### Interrupt Translation Table

Each Device ID has an Interrupt Translation Table which software must allocate, and the base address must be filled in to the Device Table. The ITTs contain Event ID → vINTID:vPE ID (optionally a doorbell pINTID as well) mappings, for each Device ID.

### vPE Table

Also a table software must allocate space for, and write the address into `GITS_BASER[0..7]` whose `Type` is vPE Table. It should be filled with vPE ID → target Redistributor:vLPI Pending Table mappings.

### Summary

The three tables above essentially translates Device ID:Event ID into:

* vINTID (virtual interrupt number)

* target redistributor (which PE)

* vLPI Pending Table Address (is the PE running the specified vPE?)

These information enables the hardware to directly inject virtual interrupts.

## Filling in the Tables

To fill the three tables

* Device Table

* Interrupt Translation Table

* vPE Table

You don’t actually write to the memory area allocated for them, but instead send commands to the ITS to instruct the hardware to write the correct entries. An in-memory command queue is used to send command to the ITS, software writes to the command queue to send commands. The command queue must also be allocated by software and the base address, read head, and write head should be written in to `GITS_BASERn` , `GITS_CREADR` , and `GITS_CWRITER` registers respectively.

## Setting up Direct Injection of Virtual Interrupts

Let’s say VM `x` ‘s vPE `y` is running on PE `z` , the hypervisor wishes to forward Device ID `d` ‘s Event ID `e` to it as an virtual interrupt with the number `f`, it must:

1. write vINTID \= `f` , vPE ID \= `y` in the Interrupt Translation Table entry that is translated using the combination `d:e`

2. set the entry of the vPE Table corresponding to vPE ID \= `y` ‘s target redistributor to `z` , and the address of the vPE’s vLPI Pending Table

## Migrating vPE

If the hypervisor is trying to move vPE `y` from PE `x1` to `x2` , it must:

1. clear `x1` redistributor’s `GICR_VPENDBASER` , and move the value into `x2` redistributor’s `GICR_VPENDBASER`

2. change vPE ID `y` ‘s vPE Table entry’s target redistributor from `x1` to `x2`

## Summary of the Acronyms

* PE: Processing Element

  * hardware

* CPU Interface

  * hardware, controlled using `mrs` , `msr` instructions

* Redistributor

  * hardware, controlled using MMIO

  * one for each PE

* Distributor

  * hardware, controlled using MMIO

  * one for each GIC

* Interrupt Translation Service

  * hardware, controlled using MMIO

  * can exist multiple instances on a system, but this article considers the case with one instance

* Device Table

  * software table, must allocate memory for it

  * write to the command queue to fill it

* Interrupt Translation Table

  * software table, must allocate memory for it

  * write to the command queue to fill it

* vPE Table

  * software table, must allocate memory for it

  * write to the command queue to fill it

* Command Queue

  * software table, must allocate memory for it

  * write to it to control:

    * Device Table

    * Interrupt Translation Table

    * vPE Table

* LPI Configuration Table

  * software table, must allocate memory for it

  * write to it directly

* LPI Pending Table

  * software table, must allocate memory for it

  * write to it directly

* vLPI Configuration Table

  * software table, must allocate memory for it

  * write to it directly

* vLPI Pending Table

  * software table, must allocate memory for it

  * write to it directly
