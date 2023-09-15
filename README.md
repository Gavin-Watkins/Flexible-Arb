# Flexible Arduino Due based Arbitrary Signal Generator
Often during a project I have needed a complex test signal that is beyond the capability of the signal or function generators available in my lab. Commercial arbitrary signal generators (ARB) consisting of a fast memory coupled to a digital-to-analogue converter (DAC) are available, but often do not offer the right mix of analog and digital outputs in a suitably flexible (and affordable!) form. This has also been a problem during my day job where I have need to drive a high efficiency radio frequency (RF) power amplifiers (PA) or transmitters with two analogue outputs for the in-phase (I) and quadrature (Q) inputs of an RF modulator and several digital outputs for switching parts of the transmitter. [Previous work used an ARB platform developed around an Arduino Mega2560](https://www.researchgate.net/publication/361063526_A_Digital_Power_Amplifier_for_32-QAM). This was limited to a sample rate of 457kS/s. To increase the sample rate a new ARB has been developed based on an Arduino Due offering up to 8.4MS/s.

The Arduino Due is based on the 32-bit ATSAM3X8E allowing multiple 8-bit DACs and digital outputs to be mapped onto a single port. This ensuring all DAC or digital outputs update at the same time and only a single LUT is call for every update. The Arduino Due also runs at a higher clock rate of 84 MHz. Although 32-bit, none of the four ports (A to D) available on the ATSAM3X8E’s pins map to all 32-bits. Of the four ports, C with 24-bits available is the most useful. These 24 bits in this project are connected to two 8-bit DACs and one 8-bit digital port. However, they could for example drive two 12-bit DACs or any such combination.

# Testing the Arduino Due
The first thing anyone looks at with an ARB is the sampling rate. In a software based ARB like this (as oppose to an FPGA implementation) the microprocessor much fetch the output samples from memory and present them to the port with the DAC. This takes ten instruction cycles resulting in 8.4 MS/s as shown below in the minimum viable code:

start:                                // start of loop  

count = count + 1;                    // increment counter, takes one instruction

count = count & 0x000000FF;           // mask to prevent overflow from end of LUT, takes one instruction

PIOC->PIO_ODSR = Portvalues[count];   // access the LUT and send it to PortC, takes six instructions

goto start;                           // and repeat, takes two instructions

The felixbility of this option opens up other possiblities, like adding a second port which reads from its own LUT. PortA offers 23 bits, which would give a total of 47bits of data. This would however slow down the sample rate to 5.25 MS/s and require external latches for synchronising the data.

# DACs
The most important part of any ARB are the DACs. For this I settle on the DAC0801. Although quite old, they are readily available and easy to interface. Their datasheet quotes a settling time of 100ns consisting of the DAC slewing time and that for any ringing at the output to settle. The slewing time is a factor of the DAC output current and its load impedance. This suggests a maximum sample rate of 10 MS/s, which fits well with the 8.4 MS/s of the Due. The DAC0801 has complementary current sink outputs (pins 2 and 4). The datasheet suggests op-amp active current to voltage (I/V) converters as an output stage. This was found to introduce excess ringing, so passive resistive I/V converters were used instead. With a 1kΩ resistive load (RL) the slew rate was 30V/μs equivalent to a rise and fall time of 42ns. 

The DAC (IC1 and IC3) and op-amp (IC2 and IC4) interface is shown in the Eagle schematic "Sch6.sch" and as a png "Sch6.png". The peak output current of the DAC0801 is set at 2.13mA by the 4k7 resistors on pins 14 and 15 (R10, R14, R19 and R20) when a ±10V power supply is used. A 1k1 resistor tied to 0V and two 4k7 resistors in series (9k4) to +10 V positive supply rail add an offset so the output voltage at pin 4 swings positive and negative around 0V. This is a high impedance port so an op-amp with a gain of 2V/V buffers the output to drive 2V peak-to-peak into a 50Ω load. In the prototype a TL071 proved optimal in terms of bandwidth and ringing. Note in the schematic this is listed as a 741, simply because it was the first op-amp listed. The TL071 has a unity gain bandwidth product of 3MHz, reduced to 1.5MHz with a gain of 2V/V. Other op-amps may provide a better combination of bandwidth, slew rate and output current.

# Code
The Arduino Due can be programmed in the usual Arduino IDE, but has a unique set of instructions which allows direct port access. The following commands setup port C of the Due for direct access: PIOC→PIO_PER and PIOC→PIO_OER. The command PIOC→PIO_ODSR is then used to write data to the port. A goto loop sends data from the LUT to Port C runs in the setup, as using the main Void loop incurs various overheads including the I/O system to check the serial port. The result is 10 instructions and 8.4MS/s output rate. The LUT allocations are accessed by the variable count which is a 32-bit integer. This is masked (by ANDing) with FF (256) in the sinewave example, 1FF (512) in the triple test example and 4FFF (16384) for the 4G Long Term Evolution (LTE) signal example. Restricting the LUT to powers of 2 simplifies (and hence speeds up) the code. For other values an IF or FOR loop could be used to reset count at any value. A noInterrupts command to disable interrupts is included as without it jitter was noted on the output waveform suggesting the compiler sets some interrupts by default. If a lower sample frequency is desired though, the delayMicroseconds command can be included. In this case noInterrupts should be commented out.

# Data Generation
The data for all ports and the trigger signal are stored in a single LUT (Portvalues in the code). For Port C, the bits that are available on the external pins are: bits 1-8 for Ch A DAC, bits 12-19 for Ch B DAC, bits 21-26 for a digital interface and bit 28 as a trigger. These are generated individually and then mapped into a 32bit word which forms the values of Portvalues by multiplying the individual values by an offset and adding them together, for example:

Portvalues[X] = 2^28 x Trigger + 2^21 x Digital + 2^12 x DAC2 + 2 x DAC1

# Tool Chain 
The first two examples were generated in an OpenOffice spreadsheet (File "Signal Gen3.ods) as the LUTs were not very big. The third example was in Python ( due to the resulting LUT size. If using a spreadsheet the data will be generated in a single column. This can be copy and pasted directly into the Arduino IDE as a single column array. It is often more practical though to do this as a 2D array. The values from different cells can be combined with commas as a string into a single cell using the “&” function. In the sinewave example where the values of column Y are to be reformatted into a 16 column by 16 row for 256 values the following is used:

=Y2 & " , " & Y3 & ", " & Y4 & ", " & Y5 & ", " & Y6 & ", " & Y7 & ", " & Y8 & ", " & Y9 & ", " & Y10 & ", " & Y11 & ", " & Y12 & ", " & Y13 & ", " & Y14 & ", " & Y15 & ", " & Y16 & ", " & Y17 & ", "

Using the autofill function will not appropriately increment the Y column values, so there is still quite a bit of manual processing to be done, which is why for larger arrays it is appropriate to use Python. The Python code reads in the File "1M4LTE.txt". The sample rate of the LTE data is 7.68 MS/s, this is the original 122.88 MS/s downsampled by 16. Adding a dummy instruction to the Arduino code results in a sample rate of 7.64 MS/s. This is not close enough to 7.68 MS/s for standard compliance, but sufficient for measuring the adjacent channel power ratio (ACPR) of a PA/transmitter output.

# Hardware Implementation
To verify the concept, an Arduino Due “shield” was produced. This was fabricated on standard FR4 substrate using through hole DIL components. The layout is shown in "Sch6.brd" for Eagle and "Sch6_PCB.png" as a png. The Gerber files are in the zip file "Gerbers.zip". DIL components incur stray inductance and capacitance when operating in the MHz region, but allow the design to be easily modified. A surface mount implementation would probably produce better results than those presented here. 

The shield runs off a 5V supply for the digital and +/-10V for the analogue. The former can be extracted from the USB interface, as it is assumed that this project will be permanently connected to a computer for updating the LUT. The +/-10V can be derived from the 5V USB with a DC-DC converter and then smoothing and filtering.

# Example Results
The code includes three examples: a 32.8125kHz quadrature sinewave on DAC1 and DAC2 with a 256 value LUT (8.4MS/s / 256 = 32.8125 kHz), a triple tone test at around 1 MHz and the 4G LTE signal generation. Screen captures of these are shown to demonstrate the capabilities of the ARB. For the triple tone test the three tone frequencies are 984.375 kHz, 1.00078125 MHz and 1.0171875 MHz. These three resulting frequencies (Fr) were calculated with:

Fr = Fs * n / LUTs

Where Fs is the sampling frequency, n a whole number to avoid discontinuities – reducing spurious produces – and LUTs the LUT size. For an Fs was 8.4MS/s, n: 60, 61 and 62, and LUTs 512. Even with all the harmonically related spurious products, they are -38dB relative to the three tones. A higher resolution DAC would offer greater dynamic range, and hence lower spurious products. 

For the LTE signal, a screen capture of the spectrum of the I-channel signal is included. This includes guard channels, resulting in an actual RF bandwidth is 1.08MHz (consisting of six 180kHz resource blocks). The baseband bandwidth is half this, a spectrum of 504kHz as clearly shown in the example. The spectral regrowth is partially due to DAC non-linearities and also spurious products introduced by the system. uffers. The next phase of this research is to combine the LTE signal generator with a low-cost quadrature modulator development board to generate 4G signals for testing transmitter architectures.
