# HPSDR-Orion-GiGE-16RX
UPDATE: release area contains both the Orion and Angelia firmware. Angelia is 12 max rcvrs.

Modified Old Protocol Orion v5.0 FPGA firmware adding Gige and 16 Receiver functionality

(N1GP) Added 16-rcvr operations, changed to use gigabit ethernet, closed timing per K5SO's
TimingClosureFieldGuide

Converted megafunctions and compiled with Quartus Lite 17.1.0

My approach was to use the existing old protocol firmware and merge in aspects from
the new Gige from both Hermes-Lite and the New protocol Ethernet..

What I have tested:

IP address change (Static/DHCP) works

Firmware upgrade works (NOTE below about recycling power)

Transmit works

Receive up to 16 receivers at 192k work, 16 @ 384k is too much for my PC to handle.

tested with cudaSDR, CWSL_Tee/SkimSrv/HermesIntf on my github site
 https://github.com/n1gp
 
And also Alans(M0NNB) SparkSDR

*NOTE: The firmware upgrade process works except you have to recycle the power
to get the FPGA to reload. I tested previous versions from 4.7 - 5.2 and they 
do the same thing. It may be my rig or ...?

I have not test Puresignal, but I have reports from others trying this firmware that is does work.

Some Fixes to note, I found that the Alex Filter algorithm had a minor bug
where an index was incorrect (~line 2610): C122_frequency_HZ[6]

if (C122_frequency_HZ[6] > C122_frequency_HZ[0] && C122_frequency_HZ[4] >= C122_frequency_HZ[1] &&
C122_frequency_HZ[6] >= C122_frequency_HZ[2] && C122_frequency_HZ[4] >= C122_frequency_HZ[3] &&
C122_frequency_HZ[6] >= C122_frequency_HZ[4] && C122_frequency_HZ[6] >= C122_frequency_HZ[5])
C122_freq_max <= C122_frequency_HZ[6];


and also that if one was to operate the firmware at the full 7 receivers
using auto for the filters, then switch say down to 3 or 4 receivers the
firmware would remember the other 5,6,7 and still use it in the Alex filter
selection algorithm. I noticed that by watching the bandscope that if I selected
say rx 7 for 10 meters and then went down to 4 rx's the filters would still
be on for the 10 meter band.

I recently added a Deadman watchdog timer settable from 1-7 seconds (0 is disabled) using the
upper (previously unused) 3 MSbits in C3, see below for protocol updates that I hope I can get approved.

In order to use up to 16 receivers I made the following modifications to the 'old' protocol below.
Basically, remove the 'Time stamp – 1PPS' and change the C0 arrangement to accomodate the new
receivers. Note this scheme is used by Hermes-Lite and Red-Pitaya to get past the 7 receiver
limit.

Also to be able to control the ADC selection of 16 receivers I chose to use the unused C4
but each bit selects the ADC1 or ADC2 since that's what current ANAN offers.

(Below was worked out with Hermes-Lite group to allow more rcvrs, decided to use it here)
     
        C4
        0 0 0 0 0 0 0 0
        | | | | | | | |
        | | | | | | + + ----------- Alex Tx relay (00 = Tx1, 01= Tx2, 10 = Tx3) 
        | | | | | + --------------- Duplex (0 = off, 1 = on)
        | + + + +------------------ Number of Receivers (0000 = 1, 1111 = 16)
        +-------------------------- Common Mercury Frequency (0 = independent frequencies to Mercury
                                    Boards, 1 = same frequency to all Mercury boards)

(Removed bit 6 above for extra rcvrs) | +------------------------ Time stamp – 1PPS on LSB of Mic data (0 = off, 1 = on)

        C0 = 0 0 0 0 0 1 0 x     C1, C2, C3, C4   NCO Frequency in Hz for Receiver _1
        C0 = 0 0 0 0 0 1 1 x     C1, C2, C3, C4   NCO Frequency in Hz for Receiver _2
        C0 = 0 0 0 0 1 0 0 x     C1, C2, C3, C4   NCO Frequency in Hz for Receiver _3
        C0 = 0 0 0 0 1 0 1 x     C1, C2, C3, C4   NCO Frequency in Hz for Receiver _4
        C0 = 0 0 0 0 1 1 0 x     C1, C2, C3, C4   NCO Frequency in Hz for Receiver _5
        C0 = 0 0 0 0 1 1 1 x     C1, C2, C3, C4   NCO Frequency in Hz for Receiver _6
        C0 = 0 0 0 1 0 0 0 x     C1, C2, C3, C4   NCO Frequency in Hz for Receiver _7
        C0 = 0 0 1 0 0 1 0 x     C1, C2, C3, C4   NCO Frequency in Hz for Receiver _8   (Was 0 0 0 1 0 0 1 x) 
        ...
        C0 = 0 0 1 1 0 1 0 x     C1, C2, C3, C4   NCO Frequency in Hz for Receiver _16

        Added ADC control for rcvrs 8-16
        C2
        0 0 0 0 0 0 0 0
        | | | | | | | |
        | | | | | | +-+------------ ADC assignment for RX5 (00 = ADC1, 01 = ADC2, 10 = ADC3)
        | | | | +-+---------------- ADC assignment for RX6 (00 = ADC1, 01 = ADC2, 10 = ADC3)
        | | +-+-------------------- ADC assignment for RX7 (00 = ADC1, 01 = ADC2, 10 = ADC3)
        +-+------------------------ ADC assignment for RX8 (00 = ADC1, 01 = ADC2, 10 = ADC3) *Was unused

        C4 – ADC 9-16 assignment (Was not presently used)
        0 0 0 0 0 0 0 0
        | | | | | | | |
        | | | | | | | +------------ ADC assignment for RX9  (0 = ADC1, 1 = ADC2)
        | | | | | | +-------------- ADC assignment for RX10 (0 = ADC1, 1 = ADC2)
        | | | | | +---------------- ADC assignment for RX11 (0 = ADC1, 1 = ADC2)
        | | | | +------------------ ADC assignment for RX12 (0 = ADC1, 1 = ADC2)
        | | | +-------------------- ADC assignment for RX13 (0 = ADC1, 1 = ADC2)
        | | +---------------------- ADC assignment for RX14 (0 = ADC1, 1 = ADC2)
        | +------------------------ ADC assignment for RX15 (0 = ADC1, 1 = ADC2)
        +-------------------------- ADC assignment for RX16 (0 = ADC1, 1 = ADC2)
        
(Adding use of upper 3 MSBits of C3 for a 1-7 second Deadman watchdog timeout)

        C0
        0 0 0 1 0 1 0 x   


        C3
        0 0 0 0 0 0 0 0
        |   | | | | | |
        |   | | | | | +------------ Metis DB9 pin 1 Open Drain Output (0=OFF, 1= ON)
        |   | | | | +-------------- Metis DB9 pin 2 Open Drain Output (0=OFF, 1= ON)
        |   | | | +---------------- Metis DB9 pin 3 3.3v TTL Output (0=OFF, 1= ON)
        |   | | +------------------ Metis DB9 pin 4 3.3v TTL Output (0=OFF, 1= ON)
        |   | +-------------------- 20dB Attenuator on Mercury when Tx (0 = disable, 1 = enable)
        +---+---------------------- 1-7 second Deadman watchdog (0 disabled)
