# QEMU kernel gdb for GNU Mach

Debugging a wedged or misbehaving gnumach guest with gdb through the QEMU
stub. A wedged guest is evidence, not garbage. Keep it running, attach,
read the crime scene, then decide between destroy and retry.

## Enabling the stub

Two ways.

The clean way is a gdb flag at machine start. With plain QEMU:

```sh
qemu-system-i386 [...] -gdb tcp::1234
```

Under libvirt the same flag goes into the domain XML:

```xml
<qemu:commandline>
  <qemu:arg value='-gdb'/>
  <qemu:arg value='tcp::1234'/>
</qemu:commandline>
```

The runtime way is the monitor command, `gdbserver tcp::1234` from the
HMP. Do not use it when the flag is already present, the second chardev
collides with the first and fails with `duplicate yank instance`.

Only one gdb connection at a time.

## The kernel ELF

You need the exact ELF that is running, with symbols. The kernel is built
with debug info, so globals and line information resolve. If the binary
does not match the running kernel, every answer the symbol table gives
you is a lie. Verify the hash before trusting anything.

Pull it straight from the build tree that produced the installed kernel.
Sources in the tree give you full source context:

```
set substitute-path /usr/src/gnumach /path/to/your/tree
```

## Attaching

Batch mode works well and keeps the session short:

```sh
gdb -q ./gnumach -batch \
  -ex "set architecture i386" \
  -ex "target remote localhost:1234" \
  -ex "info threads" \
  -ex "info registers eip esp" \
  -ex "info symbol \$eip" \
  -ex "detach"
```

Attaching halts every vCPU. Detach resumes them and frees the stub.

## Reading the machine

`info threads` lists one thread per vCPU, marked halted or running.
On a single CPU kernel the extra vCPUs sit parked in firmware at low
addresses, that is normal and is itself a useful signal.

`info symbol $eip` names the function the CPU is in. `x/8i $eip`
disassembles ahead. Globals print directly, for example `p tickless_active`
or `p hpclock_read_ns` on a tree that carries them.

Backtraces are a hint, not a promise. Kernel stacks are small and asm
trampolines break unwinding. Walk the stack by hand when it matters,
`x/32xw $esp`, and resolve the interesting addresses with
`info symbol ADDR`.

The QEMU monitor gives a second opinion from physical memory:

```sh
virsh -c qemu:///system qemu-monitor-command DOMAIN --hmp "xp /4wx 0xADDR"
```

Kernel virtual addresses map to physical with the direct map, phys equals
virt minus 0xc0000000 on i386.

## Signatures worth knowing

A healthy idle kernel halts inside its idle loop, `machine_idle` on a
plain build, `tickless_machine_idle` with `tickless_active` set on a
tickless one. Seeing that means the machine is working, not wedged.

A boot that prints `ext2fs: part:2:device:hd0: No such device or address`
and then goes quiet has stalled in the boot translators. Read where the
CPUs and the bootstrap threads sit before destroying it, that is the
evidence an upstream report needs.

## Clean exit

`detach`, then `quit`. A healthy guest keeps running. Confirm with ssh
or your serial console before reusing the domain.

Never destroy a wedged guest before reading it. Record findings with
register values and symbol names, serial output alone is not enough.
