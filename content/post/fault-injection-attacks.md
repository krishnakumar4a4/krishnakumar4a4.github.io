+++
authors = [
    "Krishna Kumar Thokala",
]
title = "Fault injection attacks on secure boot"
description = "Fault injection attacks on secure boot"
date = 2019-03-16T09:43:12+05:30
tags = [
    "Security",
    "Secure Boot",
    "Fault Injection Attacks",
    "Embedded Systems",
    "NullConf",
]
categories = [
    "security",
]
series = ["security"]
aliases = ["security"]
images = [
    "/images/encrypted_secure_boot_impl.png",
]
+++

Nullconf is one of the largest conferences in security started in 2010 (as I keep hearing that from everyone). This is the first time I have ever attended a security conference and was little over excited too. I had been there for two days and took a load of quite interesting aspects of security domain as a whole. Lot of less known and discussed topics like hardware hacking in real-time, security in the post quantum era, AI in threat detection and mitigation, vulnerability of existing telecom networking infrastructure etc are few highlights of the event.

One of the talks by Niek Timmers and Albert on compromising secure boot with various fault injection techniques was quite interesting to me. I wanna brief that here.

A regular boot sequence includes running a program from ROM which loads the first stage bootloader into SRAM from a flash storage. This code initializes DDR and loads second stage bootloader into DDR followed by kernel initialization etc.
External Flash storage is vulnerable to be tampered either by physical access or a software access. Hence the entire system control can be compromised if not taken proper care at the hardware storage layer.

A secure boot sequence ensures that an authentication mechanism in place by having a signature of the code that is about to be loaded into RAM and will be verified against at each and every stage of the boot process.

#### Secure boot flow:
Hardware → ROM → Bootloader → TEE bootloader → TEE OS → REE bootloader → REE OS → Apps.

> Along with authentication, hardware root of trust, privilege escalation control, encryption of all code and proper hardware and software exploitation defenses ensures a better secure boot mechanism.

###### How does the fault injection actually works?
CPU always expects its supply voltage to be stable, if not, glitches can occur. This is exactly what can lead to a fault injection attack. Let’s say we have got a hardware that has capability to control supply voltage to the target processor and inject glitches at a high degree of precision. (Such hardwares do exist, I have seen them live). We can actually modify instructions that are currently getting executed. Isn’t cool??

#### Instruction skipping:
{{< figure src="/images/encrypted_secure_boot_design.png">}}

As we know external flash storage is vulnerable to attacks if the attacker gets physical access to it. In an encrypted secure boot design. Assume that the first stage bootloader is residing on the internal flash memory of the controller and whereas BL2 can still be on external flash. Now, it would be extremely hard for an attacker to modify BL1 without tampering the controller. BL1 has code to decrypt and verify the BL2 on SRAM before actually loading to DDR. Hence ensures BL2 to be valid.

{{< figure src="/images/encrypted_secure_boot_impl.png">}}

How does the above setup be attacked? Can BL2 still be compromised? Yes.
If a glitch is introduced after BL2 is copied to SRAM before validation. It can potentially bypass the entire control flow until the verification step and directly jump to next set of instructions. Provided, the glitch is carefully timed and its duration is carefully chosen. This is called instruction skipping, where all the instructions in the control flow between load and verification will be corrupted and will have no impact on the state of execution.

How does an attacker determine the glitch timing?

Power analysis. True. An attacker can record the power consumption of the CPU and can easily determine various points of interest from the surges. Cryptographic operations typically consume more power while the execution is happening which will be very evident given the state of attacker’s familiarity on similar boards.

{{< figure src="/images/power_consumption_analysis.png">}}

#### Instruction corruption:
###### Attack on ARM load instructions LDR and LDMIA
> LDR — To copy a word from one memory address to a register.
> LDMIA — To copy multiple words from a memory address to registers.

Load instructions are particularly an interesting target for an attacker primarily because of the instruction encoding scheme.

###### Controlling PC during the secure boot:
A PC is special register which holds the memory address of the next instruction to be executed. ARM instructions which can accept PC as an argument are prone to fault injection attack. Assume the attacker has physical access to the setup and was able to load malicious code and pointers into an external flash memory. Whenever a load operation is about to happen for a copy operation from a memory location to register, the instruction can be corrupted using a glitch tricking the processor to load the value into a program counter(PC) instead of register.

How is this possible?

Attacked instruction: LDR R3, [R0] → HEX equivalent of the given instruction is E5903000.

Instruction to be: LDR PC, [R0] → HEX equivalent of the given instruction is E590F000.

If we closely observe the two HEX values are differ only at the 3rd byte which is “30” and “F0” values through which R3 and PC are represented in the instruction. The corresponding binary representations for “30” and “F0” are “00110000” and “11110000” respectively, and a glitch introduced with precision can actually flip the first two bits of R3 to become PC. By this glitch, an attacker would be able make the CPU load the content of R0 into a Program counter(PC) instead of an intermediate register R3. This attack has almost no impact, unless the attacker has some kind of access to the storage and was able to inject some malicious code in it from which R0 is read.

The attack surface of LDMIA instruction seems to be higher than LDR based on experimental results, primarily because of the instruction encoding scheme

##### Attack on TEE(Trusted Execution Environment) after the boot:
More secure sensitive embedded devices have something called TEEs which separates themselves from Rich Execution Environment(REE). REE will have access to dedicated APIs to make calls to TEE for running some operations in a secure environment. TEE APIs usually copies the payload from REE and executes them internally. Assuming the attacker has full control over REE and was able to make calls to TEE with malicious payload. An attacker can again introduce a fault while the malicious code pointers are being copied into the internal registers. Through a successful instruction corruption can make them copy to program counter(PC) and gain access to the TEE.

###### How exactly a randomly corrupted instruction being helpful for attacker?
This answer is quite interesting and is more about learning the behavior/patterns of corruption for a given glitch amplitude, duration and timing.

###### How does an attacker determine the exact time, duration and amplitude of the glitch that can corrupt the instruction?
Without doing a practical experiment it is impossible to determine these values. A sample setup for the experiment would look like below

{{< figure src="/images/fault_injection_setup.png">}}

Along with the the above setup, a sample program is used which runs in a loop continuously executing the load instructions.

And with glitch VCC ranging between -1.4V to -1.0V, glitch length between 700ns to 1000ns and glitch delay between 30us to 35us and for a total of 10,000 iterations. Results of the experiments are as follows.

{{< figure src="/images/LDR_inst.png" title="LDR instruction">}}
{{< figure src="/images/LDMIA_inst.png" title="LDMIA instruction">}}

In the graphs, green dots(Expected) represents no affect on instructions. Yellow dots(Mute) represents the resets and red dots(Success) represents a successful glitch.

It is quite evident from the graphs, that the LDR instruction is quite less affected by the glitch compared to LDMIA instruction and hence the attack surface is larger for LDMIA instructions.

It is observed that power glitching is more difficult to be successful if it requires to have complex bit flips.

#### Countermeasures:
###### At hardware level:
- Having a variable clock speeds for the CPU would make the code execution flow less deterministic to the attacker.
- Sensors monitoring the environmental conditions and alert on unusual activity.
###### At software level:
- Introducing random delays inside code can also increase complexity of attack.
- Double checking conditions before branching and verify at regular intervals whether expected program executed.
- Redesign of instruction encoding that would maximize the hamming distance between valid to harmful instructions.

#### Conclusion:
Having all of the above countermeasures wouldn’t really make a system completely secure but reasonably harder enough for an attacker to counter them. All the attacks described are observed on ARM 32 bit architecture, but the principles described can be applied to many other architectures as well.

> I would like to thank Niek Timmers @ riscure for reviewing this.

###### References:
- https://www.slideshare.net/tieknimmers/pew-pew-pew-designing-secure-boot-securely-134091830
- https://www.riscure.com/uploads/2017/09/Controlling-PC-on-ARM-using-Fault-Injection.pdf

