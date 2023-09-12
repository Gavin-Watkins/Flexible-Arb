# Flexible-Arb
Arduino Due based arbitrary signal generator
Often during product development, complex test signals are needed beyond those available from common signal or function generators. To meet this requirement, arbitrary signal generators (ARB) consisting of a fast memory coupled to a digital-to-analogue converter (DAC) were developed. ARBs come in many shapes and sizes dependent on the sample rate, number of channels and digital resolution. For some applications, one or two analogue outputs are sufficient, but occasionally a combination of synchronized analogue and digital outputs are required. One such application is driving contemporary high efficiency radio frequency (RF) power amplifiers (PA) or transmitters with 4G Long Term Evolution (LTE) signals. In this example, two analogue outputs are required to drive the in-phase (I) and quadrature (Q) inputs of an RF modulator and several digital outputs for switching sections of the transmitter. A recent prototype digital PA needed just such a ARB [1]. For that a platform using an Arduino Mega2560 was developed. 

The Arduino Mega2560 is based around the ATmega2560 8-bit 16MHz micro-controller. It has a total of 54 general purpose input/output (GPIO) pins configured into six 8-bit and one 6-bit ports. Data must be called separately from memory for each port, therefore new data appears on each port at different times, so they are unsynchronized. In [1] this was be overcome with external 74HC373 transparent latches between the Arduino and the DACs and digital outputs which were all enabled together after each port had been updated. This resulted in a sample rate of 457kS/s. To increase the sample rate a new ARB has been developed based on an Arduino Due offering up to 8.4MS/s.

Hardware Description
The Arduino Due is based on the 32-bit ATSAM3X8E allowing multiple 8-bit DACs and digital outputs to be mapped onto a single port. This ensuring all outputs are synchronized. Sample rate is dramatically improved as only one look-up-table (LUT) call is made from memory. The Arduino Due also runs at a higher clock rate of 84MHz. Although 32-bit, none of the four ports (A to D) available on the ATSAM3X8E’s pins map to all 32-bits. Of the four ports, C with 24-bits available is the most useful. A simplified schematic is shown in Figure 1. Along with the digital outputs, a trigger output is also included.

DAC Requirements
The ATSAM3X8E includes internal DACs, but writing to them takes many clock cycles resulting in a low sample rate. For this reason, external DAC0801s are used. Although these are quite old, they are still available and easy to interface. Their datasheet quotes a settling time of 100ns consisting of the DAC slewing time and that for any ringing at the output to settle [2]. The slewing time is a factor of the DAC output current and its load impedance. The DAC0801 has complementary current sink outputs (pins 2 and 4).  

The DAC0801 datasheet suggests op-amp active current to voltage (I/V) converters as an output stage. This was found to introduce excess ringing, so passive I/V converters were used instead. With a 1kΩ resistive load (RL) the slew rate was 30V/μs equivalent to a rise and fall time of 42ns. The schematic of the DAC and op-amp interface is shown in Figure 2.

The peak output current of the DAC0801 is set at 2.13mA by the 4k7 resistors on pins 14 and 15 when a ±10V power supply is used. A 1k1 resistor tied to 0V and two 4k7 resistors in series (9k4) to +10 V positive supply rail add an offset so the output voltage at pin 4 swings positive and negative around 0V. This is a high impedance port so an op-amp with a gain of 2V/V buffers the output to drive 2V peak-to-peak into a 50Ω load. In the prototype a TL071 proved optimal in terms of bandwidth and ringing. The TL071 has a unity gain bandwidth product of 3MHz, reduced to 1.5MHz with a gain of 2V/V. Other op-amps may provide a better combination of bandwidth, slew rate and output current.

Programming
Arduinos are very easy to program in C with their free integrated development environment (IDE). The Arduino Due does however use some different commands to those of the more common varieties when directly accessing ports. The following commands setup port C of the Due for direct access: PIOC→PIO_PER and PIOC→PIO_OER. PIOC→PIO_ODSR is then used to write data to the port as shown in the code listing in Figure 3. 

There are a number of “eccentricities” in this code. Firstly, the loop which accesses the LUT and sends it to Port C runs in the setup, not in the main loop. For some reason it ran quicker taking only 10 instructions, resulting in 8.4MS/s output rate. The LUT allocations are accessed by the variable count which is a 32-bit integer. This is masked (by ANDing) with FF (256) in the example shown here. Increasing this to FFF would allow addressing of a 4096 allocation LUT etc. Restricting the LUT to powers of 2 simplifies (and hence speeds up) the code. For other values an IF or FOR loop could be used to reset count at any value. 

The code listed in Figure 3 generates two quadrature sine waves on the two DAC outputs (Ch A and Ch B) at 32.8125kHz (8.4MS/s / 256). count is incremented by 1 on each iteration. Increasing the increment value would increase the sine wave frequency similar to direct digital synthesis [3]. 

Another eccentricity of the code is included the noInterrupts command to disable interrupts even though none were set. If not included, the complier appears to enable an interrupt by default resulting in jitter on the output waveform. If a lower sample frequency is desired though, the delayMicroseconds command can be included. In this case noInterrupts should be commented out.

# Data Generation
The data for all ports and the trigger signal are stored in a single LUT (Portvalues in Figure 3). For Port C, the bits available on the external pins are: bits 1-8 for Ch A, bits 12-19 for Ch B, bits 21-26 for the digital interface and bit 28 as a trigger. These are generated individually and then mapped into a 32bit word by multiplying the individual values by an offset and adding them together, for example:

Word32 = 2^28*Trigger + 2^21*Digital + 2^12*DAC2 + 2*DAC1

# Tool Chain 
The data can be generated in a spreadsheet like Excel or OpenOffice, or programming language like Python or Matlab. If using a spreadsheet, one useful tip is the “&” function to combine the values of individual cells into a single cell of comma separated values which can be copy and pasted into the Arduino code. LUTs are normally displayed as a square array, i.e. 16 columns by 16 rows for 256 values like in Figure 3 instead of a single column. The “&” command can be used for example:

=Y2 & " , " & Y3 & ", " & Y4 & ", " & Y5 & ", " & Y6 & ", " & Y7 & ", " & Y8 & ", " & Y9 & ", " & Y10 & ", " & Y11 & ", " & Y12 & ", " & Y13 & ", " & Y14 & ", " & Y15 & ", " & Y16 & ", " & Y17 & ", "

This groups together the values from cells Y2 to Y17 into a single cell so that when pasted into the Arduino IDE they appear as a single row of 16 columns.

# Hardware Implementation
To verify the concept, an Arduino Due “shield” was produced. This was fabricated on standard FR4 substrate using through hole DIL components. DIL components incur stray inductance and capacitance when operating in the MHz region, but allow the design to be easily modified. A surface mount implementation would probably produce better results than those presented here. 

A screen capture of the two quadrature outputs generated by the code in Figure 3 is shown in Figure 4. There is no output lowpass filter after the DACs other than the frequency response of the TL071. Additional passive filtering would reduce the output switching noise. 

More complex waveforms than sinusoid waves can also be generated, for example the baseband complex data for a 1.4MHz LTE signal consisting of 16384 32-bit words (64kbytes). This signal was processed in Python into an array of 1024 rows and 16 comma separated columns. The 1.4MHz LTE signal has a sample rate of 7.68MS/s. An additional instruction was included in the Arduino loop, resulting 11 instructions and a sample rate of 7.64MS/s. Further, count was ANDed with 4FFF. The frequency domain of the in-phase (I) baseband signal in Figure 5.

The 1.4MHz LTE signal includes guard channels, resulting in an actual RF bandwidth is 1.08MHz (consisting of six 180kHz resource blocks). The baseband bandwidth is half this, 504kHz as clearly shown in Figure 5. There is substantial spectral regrowth in Figure 5 partially due to DAC non-linearities and also spurious products introduced by the system. 

Another ARB application is to directly synthesize modulated RF carriers without any up-conversion. This is often promoted with very high speed (>10 GS/s) ARBs for generating signals with multi GHz bandwidth at GHz carrier frequencies [4]. A simpler example is shown in Figure 6, for a triple tone test. The three tone frequencies are 984.375 kHz, 1.00078125 MHz and 1.0171875 MHz. These three resulting frequencies (Fr) were calculated with:

Fr = Fs * n / LUTs

Where Fs is the sampling frequency, n a whole number to avoid discontinuities – reducing spurious produces – and LUTs the LUT size. Here, Fs was 8.4MS/s, n: 60, 61 and 62, and LUTs 512. Figure 6 shows the spectrum. Even with all the harmonically related spurious products, they are -38dB relative to the three tones. A higher resolution DAC would offer greater dynamic range, and hence lower spurious products [5]. 

Conclusion
A technique using an Arduino Due to generate arbitrary waveforms with multiple analogue and digital outputs is discussed in this article. This is seen as a starting point for discussion of what is possible with low-cost hobbyist level electronic development boards. Promising results are presented for the generation of sinusoid waves, a LTE 4G baseband signals and a three tone tests. 
Alternative platforms running at higher clock frequencies like the Arduino Portenta H7, Raspberry Pi RP2040, Raspberry Pi Zero 2 W or ESP32 could provide higher sampling frequencies and better results. Similarly, so could higher resolution DACs and better op-amp buffers. The next phase of this research is to combine the LTE signal generator with a low-cost quadrature modulator development board to generate 4G signals for testing transmitter architectures.
