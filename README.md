SNES Enhancement Cartridge Technical Design Overview


1. Introduction


This document describes the architecture, operation, and design rationale of a Super Nintendo Entertainment System enhancement cartridge built around a modern microcontroller. The system is designed to provide deterministic interaction with the SNES bus while enabling extended capabilities such as large-scale asset streaming, high-quality audio playback, and controlled persistent storage. The design prioritizes timing correctness, electrical safety, and flexibility through software-managed resources.


2. SNES Address Space and Cartridge Memory Model


The SNES CPU provides a 24-bit address space organized into banks, with each bank containing 64 kilobytes of addressable memory. Although the full address space spans 16 megabytes, this cartridge design intentionally services only the lower 16 address lines, exposing a 64 kilobyte window to the SNES at any given time. The upper address bits are not directly decoded in hardware. This decision is driven by strict timing constraints, as SNES memory accesses must be serviced within tens of nanoseconds. By limiting address decoding to a single 16-bit read from a GPIO port, the system minimizes latency and ensures deterministic behavior within the required timing window.


The cartridge is therefore built around a windowed memory model in which a fixed 64 kilobyte region is dynamically mapped by the microcontroller to a much larger virtual address space. This virtual space includes data sourced from internal SRAM, external pseudo-static RAM connected via a quad SPI interface, and files stored on an SD card. The microcontroller is responsible for managing this mapping and updating the visible contents of the window as requested by the SNES. From the perspective of the SNES CPU, this memory behaves like conventional ROM, but in practice it is backed by a software-defined memory system.


3. Cartridge Select Signal Interpretation


The signal traditionally labeled as ROMSEL on the SNES is used in this design as a general cartridge select signal. Although historically associated with read-only memory, the signal simply indicates that the CPU is accessing the cartridge address space. In this implementation, that space is treated as a bidirectional interface. During read cycles, the cartridge provides data through the memory window, while during write cycles, the SNES is able to send commands and data to the microcontroller. This effectively turns the cartridge address space into a mailbox-driven communication channel.


4. Data Bus Handling and Bidirectional Operation


The data bus is handled in a bidirectional manner using a combination of fixed-direction microcontroller inputs and a level-shifted output path. The SNES data lines operate at five volts, and the microcontroller reads them through five-volt tolerant input pins. When the SNES performs a write operation, the cartridge output drivers are disabled and the microcontroller samples the incoming data directly. When the SNES performs a read operation, the microcontroller continuously drives the output side of the level shifter, but the level shifter itself is only enabled during valid bus cycles. This approach eliminates the need to dynamically switch GPIO direction, avoiding latency and preventing contention.


5. Output Enable Gating and Bus Safety


Control of the data bus output enable is implemented entirely in discrete logic. The output enable signal for the level shifter is derived from a combination of the cartridge select signal and the SNES bus timing signal. The output is only enabled when the cartridge is selected and the bus is in the appropriate phase for a read operation. When the cartridge is not selected, or when the bus is outside a valid read window, the outputs are placed into a high impedance state. This matches the expected behavior of a standard SNES cartridge, where the data bus must not be driven when the device is not actively addressed. By implementing this gating in hardware, the system guarantees correct timing regardless of firmware execution state.


6. Frame Release and Cooperative Timing
A key feature of the system is the frame release mechanism, which allows the microcontroller to temporarily suspend real-time bus servicing in order to perform background tasks. During this period, the cartridge deliberately returns a fixed, recognizable data pattern regardless of the address being read. A routine running on the SNES, typically from work RAM, polls the cartridge until valid data resumes. This mechanism provides a cooperative synchronization point between the SNES CPU and the microcontroller. While the cartridge is in the not-ready state, the microcontroller can perform operations such as audio mixing, SD card transfers, and memory remapping. The SNES may either wait or execute code from internal RAM to avoid visible stalls.


7. Audio Streaming and Synchronization


Audio playback is implemented using a streaming architecture. Data is read from the SD card using a multi-bit SDIO interface and buffered in memory. The microcontroller processes this data and outputs a continuous PCM stream through its serial audio interface to an external digital-to-analog converter, specifically the PCM5102A. This component provides high-quality stereo output at line level and requires minimal external circuitry. The use of DMA for audio output ensures uninterrupted playback while the processor handles other tasks.
To maintain synchronization between audio and video, the system references the SNES master clock, which operates at approximately 21.477 MHz in NTSC systems. This clock is used to derive the effective frame timing of the console. Rather than relying on a fixed number of audio samples per frame, the microcontroller tracks timing drift and adjusts playback using fractional stepping. This involves slightly modifying the rate at which samples are consumed from the buffer while maintaining a fixed output rate to the DAC. The result is long-term synchronization between audio and visual events without introducing audible artifacts.


8. External Memory Architecture


External memory is provided in the form of an eight megabyte pseudo-static RAM device connected over a quad SPI interface. This memory serves as a high-capacity cache for assets, audio data, and other resources that are too large to fit in internal memory. It allows the system to preload data from the SD card and avoid latency during time-critical operations. While slower than internal SRAM, it is significantly faster and more predictable than direct SD card access.


9. USB High-Speed Interface


The system includes a USB high-speed interface implemented using a ULPI-connected physical layer device. This interface allows the cartridge to appear as a communication and storage device when connected to a host computer. It supports terminal-style debugging through a virtual serial interface and enables direct access to the SD card for rapid file transfer and development. This eliminates the need to remove the SD card during iteration and provides a modern workflow for updating assets and inspecting system behavior.


10. Power Supervision and Safe Shutdown


Power integrity and safe shutdown behavior are addressed through the use of two voltage supervisors. A supervisor monitoring the 3.3 volt rail provides early detection of power loss and allows the microcontroller to halt SD card operations before voltage drops below safe levels. This reduces the risk of file system corruption. A second supervisor monitors the 5 volt rail supplied by the SNES and directly controls the output enable logic of the level shifter. If the supply becomes unstable, the data bus is immediately placed into a high impedance state, ensuring that the cartridge does not drive invalid signals onto the system bus regardless of firmware state.


11. Reset Control and System Recovery
Reset handling is implemented using a bidirectional control scheme. The SNES reset line is monitored by the microcontroller and can also be asserted by it using an open-drain output. This allows the cartridge to detect when the user presses the reset button and to coordinate internal state accordingly. It also allows the microcontroller to initiate a system reset when necessary, such as after a fault condition or during controlled reinitialization. A minimal supervisory firmware component ensures that reset handling remains reliable even if higher-level firmware encounters an error.


12. CIC Lockout Compatibility


To satisfy the SNES lockout mechanism, a small secondary microcontroller is used to emulate the cartridge side of the CIC authentication system. This device operates independently of the main processor and handles the required challenge-response communication at startup, allowing the console to proceed with normal operation.


13. Persistent Storage and RTC


Persistent storage is provided through a combination of internal battery-backed SRAM within the microcontroller and external SD card storage. The internal SRAM resides in the backup domain and retains its contents when main power is removed. Because it is not directly connected to the SNES bus, all access is mediated by the microcontroller. The SNES interacts with this storage through a mailbox-style command interface, sending requests that the microcontroller interprets and executes. The real-time clock is also part of this domain and is accessed in the same manner. This design allows for controlled, transactional access to persistent data rather than direct memory mapping.
The SD card provides larger-scale storage for game data and save files. While it is possible to write save data directly to the SD card, such operations are not atomic and must be protected against interruption. The system may present a warning during save operations, similar to historical console behavior. To improve reliability, internal SRAM can be used as a staging area before committing data to the SD card, reducing the likelihood of corruption.


14. Conclusion


This cartridge architecture transforms the traditional SNES memory interface into a controlled, software-defined system. By combining deterministic bus handling, hardware-assisted safety mechanisms, and flexible memory management, the design enables capabilities far beyond those of a standard cartridge while maintaining compatibility with the original hardware constraints.