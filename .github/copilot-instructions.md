# GitHub Copilot Instructions for GenericPLT

## Project Overview

GenericPLT (Generic Pins, Levels and Timings) is a standardized pin naming convention for test programs. It defines generic pin names based on **HVM (High Volume Manufacturing) testing function** rather than RTL (Register Transfer Level) design names or physical package pin names. This approach enables test program portability, maintainability, and reusability across different silicon implementations and package types.

### Key Terminology

- **PLT**: Pins, Levels and Timings in an ATE (Automated Test Equipment) Test Program
- **PLTA**: PLT Architect - the role responsible for defining and maintaining PLT
- **GenericPLT Process**: The PLTA defines a possible list of Generic PinNames for a given dielet or family of dielets, which can work across testing operations like Sort, Class, and BurnIn
  - The PLTA starts by assessing possible RTL pin names for all the dielets involved in a given product
  - From this RTL assessment, the PLTA creates generic pin names based on HVM testing function rather than RTL-specific naming
  - As of today, GenericPLT defines two major buckets of generic pin names: **I/O testing** and **Digital testing**

### Pin Categories

#### Digital Testing Block
Contains all the pins required to perform digital testing of a given dielet through DFX methods. Pins grouped into the digital block are utilized for:
- Reset execution
- Internal registers configurations
- Dielet fabric test port operations
- Reset/power good pins
- Clocks for HVM testing
- Microbreakpoint pins
- Internal signals observability
- Test Access Port (TAP)
- FA/FA debug trigger pins

The digital block is subdivided into smaller groups for easy handling and mapping:
- **CLOCKS**: HVM testing clocks (further broken down into 4 subblocks: CLK_0, CLK_1, CLK_2, and CLK_3)
  - Each subblock contains 5 pins segregated into different ATE resource types:
    - **HPCDIF**: High Purity Clock Differential (HPC in differential mode)
    - **HPC**: High Purity Clock - can support both single-ended and differential pair mode
    - **EC**: Enable Clock - can support both single-ended and differential pair mode
    - **DP_0**: Data Pin index 0
    - **DP_1**: Data Pin index 1
  - When EC and HPC are in single-ended mode, they allow parametric measurements with no issues, so HPC and EC can perform both parametric measurements and digital testing. No need to create a different pin to change the mode. When in differential mode (HPCDIF), parametric measurements are not allowed on both pins, so pins need to be reconfigured as single-ended using a different pin, normally as Data Pin mode (DP_0/DP_1).
  - **Important Note**: There was a misunderstanding in the past where people believed all EC and HPC regardless of single-ended or differential needed a Data Pin counterpart for parametric measurements. This complicated test program pin definitions because extra pins needed to be created for Opens & Shorts. **Clarification**: If the resource is single-ended (regardless of Data Pin, EC, or HPC mode), no single-ended counterpart is required. Only if the resource is HPC/EC configured as differential, then single-ended versions (DP_0/DP_1) are needed.
  - **Why Differential Resources Need Single-Ended Counterparts**: On the HDMT platform, all differential mode resources (DifferentialDPin, DifferentialEnabledClock, DifferentialHighPurityClock) lack PMU behavior - they only support digital testing. PMU (Parametric Measurement Unit) behavior is required for parametric measurements such as voltage/current measurements, which are essential for Opens/Shorts testing. Therefore, resources without PMU behavior require a counterpart resource with PMU behavior (like single-ended DPin mode) in order to perform parametric measurements.
  - **Key Benefit**: It is not needed to create single-ended resources for resources that are already PMU capable, reducing the amount of unnecessary pins in the test program definition. This understanding prevents over-complicating pin definitions and keeps the PLT manageable.
  - **GenericPLT CLOCKS Block Rule**: If in GenericPLT CLOCKS block a pin is defined as differential, then it needs to ensure that DPin is enabled under DP_0 or DP_1. This ensures parametric measurement capability is always available for differential clock resources. If EC or HPC are single-ended, then DP_0 and DP_1 are not required since single-ended resources already have PMU capability.
  - DP_0 and DP_1 are primarily intended for parametric measurements when tester channels are set to differential mode. Differential mode cannot perform parametric measurements, therefore channels need to be reconfigured to Data Pin mode to do so. Current HDMT tester platform allows mapping the same channel to two or more pins with different resource modes. HDMT does not allow channel sharing of pins under the same resource type.
  - **HDMT Limitation**: HDMT cannot change the configuration of a pin once the test program is loaded, so the approach is to create two pins with different names that share the same channel. For a given content, one pin under a specific resource is used, and for different content the other pin is used.
    - **Example**: pin_EC under channel 0.000 is defined as Enable Clock and pin_DP on the same channel 0.000 is configured as Data Pin mode. Depending on the content, pin_EC will be used or pin_DP will be used. Most likely for digital testing pin_EC will be in patterns, but when performing Opens/Shorts tests, pin_DP will be used.
- **TAP**: Test Access Port pins for TAP/JTAG operations
  - Contains **5 pins total**:
    - **TRST**: Test Reset - resets the TAP controller (HDMT DPin resource)
    - **TMS**: Test Mode Select - controls TAP state machine (HDMT DPin resource)
    - **TDI**: Test Data In - serial data input (HDMT DPin resource)
    - **TDO**: Test Data Out - serial data output (HDMT DPin resource)
    - **TCK**: Test Clock - clock for TAP operations (HDMT EC resource)
  - **GenericPLT TAP Pin Nomenclature**:
    - Single dielet: `CPU_TRST`, `CPU_TCK`, `CPU_TMS`, `CPU_TDI`, `CPU_TDO`
    - Multiple dielets: `CPU1_TRST`, `CPU1_TCK`, `CPU1_TMS`, `CPU1_TDI`, `CPU1_TDO` and `CPU2_TRST`, `CPU2_TCK`, `CPU2_TMS`, `CPU2_TDI`, `CPU2_TDO`
- **RESET (RST)**: Reset execution, STRAPs configurations, power good assertion, and reset assertions
  - Contains **20 pins total**: indexed from 00 to 19
  - Pin group identification: **RST**
  - **GenericPLT RESET Pin Nomenclature**:
    - Single dielet: `CPU_RST_00` through `CPU_RST_19`
    - Multiple dielets: `CPU1_RST_00` through `CPU1_RST_19` and `CPU2_RST_00` through `CPU2_RST_19`
- **MBP**: Microbreakpoint pins
  - Pin group identification: **MBP**
  - Contains **9 pins total** subdivided into 3 smaller groups:
    - **IN**: 3 pins indexed from 0 to 2
    - **OUT**: 3 pins indexed from 0 to 2
    - **INOUT**: 3 pins indexed from 0 to 2
  - **GenericPLT MBP Pin Nomenclature**:
    - Single dielet: `CPU_MBP_IN_0` through `CPU_MBP_IN_2`, `CPU_MBP_OUT_0` through `CPU_MBP_OUT_2`, `CPU_MBP_INOUT_0` through `CPU_MBP_INOUT_2`
    - Multiple dielets: `CPU1_MBP_IN_0` through `CPU1_MBP_IN_2`, `CPU1_MBP_OUT_0` through `CPU1_MBP_OUT_2`, `CPU1_MBP_INOUT_0` through `CPU1_MBP_INOUT_2` (and CPU2_*)
- **TESTPORT (TSTPRT)**: Dielet fabric test port operations
  - Pin group identification: **TSTPRT**
  - Used for dielet fabric operations like SSN or STF protocols
  - Parallel ports for faster data retrieval/input into internal registers and DFX (faster than JTAG)
  - Highly used for Scan testing and other DFX operations
  - Contains **36 pins total** subdivided into input and output:
    - **Input (I)**: 16 data pins + 2 input clocks (1 DPin + 1 EC) = 18 pins
    - **Output (O)**: 16 data pins + 2 output clocks (1 DPin + 1 EC) = 18 pins
  - **GenericPLT TESTPORT Pin Nomenclature**:
    - Single dielet Input: `CPU_TSTPRT_I_CK_EC` (EC clock), `CPU_TSTPRT_I_CK` (DPin clock), `CPU_TSTPRT_I_00` through `CPU_TSTPRT_I_15` (data)
    - Single dielet Output: `CPU_TSTPRT_O_CK_EC` (EC clock), `CPU_TSTPRT_O_CK` (DPin clock), `CPU_TSTPRT_O_00` through `CPU_TSTPRT_O_15` (data)
    - Multiple dielets: Same pattern with `CPU1_TSTPRT_*` and `CPU2_TSTPRT_*` prefixes
  - **Important Note**: If `CK_EC` and `CK` for input or output point to the same channel, they cannot be active at the same time. If `CK` (DPin) is activated, then `CK_EC` needs to be deactivated, and vice versa. This follows the same channel sharing principle as the CLOCKS block.
    - **HDMT Implementation**: Channel activation/deactivation happens in the HDMT socket file (.soc) for a given test program. When a channel is deactivated, it is set to Undefined status, commonly referred to as "U'ing the pin out" or "U'ing out the pin". For example, to use the EC resource, you "U out" the DPin version of that pin in the socket file.
- **OBSERVABILITY (OBS)**: Internal signals observability
  - Contains **20 pins total**: indexed from 00 to 19
  - Pin group identification: **OBS**
  - **GenericPLT OBSERVABILITY Pin Nomenclature**:
    - Single dielet: `CPU_OBS_00` through `CPU_OBS_19`
    - Multiple dielets: `CPU1_OBS_00` through `CPU1_OBS_19` and `CPU2_OBS_00` through `CPU2_OBS_19`
- **PROBES**: FA/FA debug trigger pins
  - Contains **2 pins total**: TRIGGER type
  - Used for FIFA or component debug to hook probing tools and simple scope waveform capturing for pattern debug
  - **GenericPLT PROBES Pin Nomenclature**:
    - Single dielet: `CPU_PROBE_TRIG0`, `CPU_PROBE_TRIG1`
    - Multiple dielets: `CPU1_PROBE_TRIG0`, `CPU1_PROBE_TRIG1` and `CPU2_PROBE_TRIG0`, `CPU2_PROBE_TRIG1`

**Note**: All dielets will require a digital testing block. GenericPLT digital testing block uses the same generic pin names across all dielet types (CPU, GCD, HUB, PCD). While the actual test patterns cannot be directly reused across different dielet types due to different RTL behaviors, the infrastructure to generate patterns for ATE (pattern headers, pin definitions, tooling) can be the same, enabling significant reuse of test program infrastructure and development processes.

#### Pattern Domains

GenericPLT pin names are used to generate test content patterns for ATE (Automated Test Equipment), specifically for the HDMT (High Density Modular Test) platform. Pins included in a pattern are organized by **Domain** - a pin grouping definition.

**Domain Purpose**: Domains are constructed for ATE test content pattern generation. A Domain contains the pins used in a given test pattern, organizing which pins are included when generating patterns for the ATE platform.

**Important Note**: Not all GenericPLT pin names are used in test patterns. Some pins are used for analog measurements, monitoring purposes, or debug operations not intended for test patterns. However, the **majority of GenericPLT pin names are consumed by patterns**.

**Examples of Non-Pattern Pins:**
- **Compensation/Reference**: `HUB_CNV_RCOMP` (resistance compensation)
- **Identification**: `CPU_ID`, `CPU_ID_2`, `CPU_ID_3` (device identification)
- **EDM (Edge Die Monitor)**: `BASE_EDM`, `CPU_EDM`, `CPU1_EDM`, `GCD_EDM`, `GND_EDM`, `HUB_EDM`, `PCD_EDM` (test structures around die periphery to check for damages due to dielet singulation)
- **Mechanical/Detection**: `EKEY`, `SKTOCC_B` (socket presence/keying)
- **DLVROUT (Delivery Output)**: `CPU_ATOM0_DLVROUT` through `CPU_ATOM3_DLVROUT`, `CPU_CORE0_DLVROUT` through `CPU_CORE3_DLVROUT`, `CPU_CCF_DLVROUT` (and CPU1_* variants) (analog voltage delivery monitoring)
- **TDAU (Temperature Diode Analog Unit)**: `CPU_TDAU0`, `CPU1_TDAU0`, `GCD_TDAU0`, `PCD_TDAU0`, `HUB_TDAU0` (temperature sensing)

For the **NVL family**, there is **one Domain per dielet**, containing all pins (digital + I/O) for that dielet:

**NVL Domain Definitions:**
- **CPU_ALL** - All CPU pins (digital + I/O) for single CPU or first CPU instance
- **CPU_ALL_1** - All CPU pins for second CPU instance (used in dual-CPU configurations like Sk i9 w BLLC 52C)
- **GCD_ALL** - All GCD pins (digital + I/O)
- **HUB_ALL** - All HUB pins (digital + I/O: 112 digital + 524 I/O = 636 pins total)
- **PCD_ALL** - All PCD pins (digital + I/O: 112 digital + 601 I/O = 713 pins total)

**Domain Usage Examples:**
- **NVL S i3 (8C)**: Uses `CPU_ALL`, `GCD_ALL`, `HUB_ALL`, `PCD_ALL`
- **Sk i9 w BLLC (52C)**: Uses `CPU_ALL`, `CPU_ALL_1`, `GCD_ALL`, `HUB_ALL`, `PCD_ALL`

Patterns reference pins from their corresponding Domain, enabling clean separation and reusability across different product configurations.

#### HDMT Test Program Architectures

HDMT supports two test program architecture types that affect how GenericPLT pin names are referenced:

##### 1. Monolithic Architecture
- Traditional single test program structure
- Pins referenced **directly without prefix**
- Example: `CPU_CLK_0_HPCDIF`, `HUB_DDR_00_CLK_N`

##### 2. IntraDUT (IDUT) Architecture
- Modular test program structure with scope-based organization
- Uses two types of scopes:

**IP Scopes** (multiple per test program):
- One scope per IP/dielet: `IPC::` (CPU), `IPG::` (GCD), `IPH::` (HUB), `IPP::` (PCD)
- Each IP scope can have **one or more domains**, or **no domains at all**
- **Within IP scope**: Pins referenced directly without prefix
  - Example: `CPU_CLK_0_HPCDIF` (when operating in IPC scope)
- **From PKG scope**: Pins require **IPfication** (adding IP scope prefix)
  - Example: `IPC::CPU_CLK_0_HPCDIF` (when referencing from package level)

**PKG Scope** (package - one per test program):
- Package-level operations and cross-IP coordination
- PKG scope can have **one or more domains**, or **no domains at all**
- To access IP pins from PKG scope: **IPfication required**
- Package-specific pins: No prefix needed

**IPfication**: The process of adding IP scope prefix to pin names when referencing them from PKG scope to IP scopes. This enables cross-scope references in IDUT architecture.

**Key Concept**: GenericPLT pin names remain architecture-agnostic. The referencing method (with or without IP scope prefix) depends on the test program architecture and context, but the underlying generic names stay consistent.

#### I/O Testing Block
Defines pins involved in High Speed and Low Speed I/Os for a given dielet. Normally those pins are related to specific IPs like:
- PCIe (PCI Express)
- Thunderbolt
- CSI (Camera Serial Interface)
- DMI (Direct Media Interface)
- DP (DisplayPort)
- And other interconnections and peripherals on the SoC

The I/O block can also be defined as pins used in the platform or system. Note that some dielets have platform and system I/Os while others don't.

**Note**: Not all dielets require an I/O block.

##### PCD I/O Testing Block

The PCD (Platform Controller Die) I/O Testing Block contains the following IP groups:

1. **INJREF** - Injection Reference pins
   - Contains **10 pins**: `PCD_INJREF_00` through `PCD_INJREF_09`

2. **CLKOUT** - Clock Output pins
   - **DMI Clock**: `PCD_CLKOUT_DMI_N`, `PCD_CLKOUT_DMI_P` (differential pair)
   - **SRC Clocks**: 9 differential pairs (SRC0-SRC8)
     - `PCD_CLKOUT_SRC0_N`, `PCD_CLKOUT_SRC0_P` through `PCD_CLKOUT_SRC8_N`, `PCD_CLKOUT_SRC8_P`
   - **Total**: 20 pins (10 differential pairs)

3. **CNV** - Converged Interface
   - **HSIF**: `PCD_CNV_HSIF_RXN`, `PCD_CNV_HSIF_RXP`, `PCD_CNV_HSIF_TXN`, `PCD_CNV_HSIF_TXP`
   - **STEP**: `PCD_CNV_STEP_REXT`, `PCD_CNV_STEP_MON`
   - **Total**: 6 pins

4. **CSI** - Camera Serial Interface
   - **CSIA**: 2 data lanes + clock (6 pins)
     - `PCD_CSIA_CKN`, `PCD_CSIA_CKP`, `PCD_CSIA_D0N`, `PCD_CSIA_D0P`, `PCD_CSIA_D1N`, `PCD_CSIA_D1P`
   - **CSIB**: 2 data lanes + clock (6 pins)
     - `PCD_CSIB_CKN`, `PCD_CSIB_CKP`, `PCD_CSIB_D0N`, `PCD_CSIB_D0P`, `PCD_CSIB_D1N`, `PCD_CSIB_D1P`
   - **CSIC**: 2 data lanes + clock (6 pins)
     - `PCD_CSIC_CKN`, `PCD_CSIC_CKP`, `PCD_CSIC_D0N`, `PCD_CSIC_D0P`, `PCD_CSIC_D1N`, `PCD_CSIC_D1P`
   - **Total**: 18 pins (3 CSI ports)

5. **DMI** - Direct Media Interface
   - **4 lanes** (00-03), each with RX/TX differential pairs
     - `PCD_DMI_0_RXN`, `PCD_DMI_0_RXP`, `PCD_DMI_0_TXN`, `PCD_DMI_0_TXP` through
     - `PCD_DMI_3_RXN`, `PCD_DMI_3_RXP`, `PCD_DMI_3_TXN`, `PCD_DMI_3_TXP`
   - **Total**: 16 pins (4 lanes × 4 pins)

6. **GPP** - General Purpose Pins
   - **GPP_A**: 29 pins (`PCD_GPP_A_00` through `PCD_GPP_A_28`)
   - **GPP_B**: 26 pins (`PCD_GPP_B_00` through `PCD_GPP_B_25`)
   - **GPP_C**: 24 pins (`PCD_GPP_C_00` through `PCD_GPP_C_23`)
   - **GPP_D**: 26 pins (`PCD_GPP_D_00` through `PCD_GPP_D_25`)
   - **GPP_E**: 23 pins (`PCD_GPP_E_00` through `PCD_GPP_E_22`)
   - **GPP_F**: 24 pins (`PCD_GPP_F_00` through `PCD_GPP_F_23`)
   - **GPP_H**: 25 pins (`PCD_GPP_H_00` through `PCD_GPP_H_24`)
   - **GPP_S**: 8 pins (`PCD_GPP_S_00` through `PCD_GPP_S_07`)
   - **GPP_V**: 18 pins (`PCD_GPP_V_00` through `PCD_GPP_V_17`)
   - **Total**: 203 pins

7. **PCIE4** - PCIe Gen4 Interface
   - **8 lanes** (00-07), each with RX/TX differential pairs
     - `PCD_PCIE4_00_RXN`, `PCD_PCIE4_00_RXP`, `PCD_PCIE4_00_TXN`, `PCD_PCIE4_00_TXP` through
     - `PCD_PCIE4_07_RXN`, `PCD_PCIE4_07_RXP`, `PCD_PCIE4_07_TXN`, `PCD_PCIE4_07_TXP`
   - **Total**: 32 pins (8 lanes × 4 pins)

8. **PCIE5** - PCIe Gen5 Interface
   - **24 lanes** (00-23), each with RX/TX differential pairs
     - `PCD_PCIE5_00_RXN`, `PCD_PCIE5_00_RXP`, `PCD_PCIE5_00_TXN`, `PCD_PCIE5_00_TXP` through
     - `PCD_PCIE5_23_RXN`, `PCD_PCIE5_23_RXP`, `PCD_PCIE5_23_TXN`, `PCD_PCIE5_23_TXP`
   - **Total**: 96 pins (24 lanes × 4 pins)

9. **SPI0** - SPI Interface
   - Contains **8 pins**: `PCD_SPI0_00` through `PCD_SPI0_07`

10. **TCP** - Thunderbolt/USB-C Interface
    - **TCP0**: 2 lanes + AUX (10 pins)
      - `PCD_TCP0_0_TXN`, `PCD_TCP0_0_TXP`, `PCD_TCP0_0_TXRXN`, `PCD_TCP0_0_TXRXP`
      - `PCD_TCP0_1_TXN`, `PCD_TCP0_1_TXP`, `PCD_TCP0_1_TXRXN`, `PCD_TCP0_1_TXRXP`
      - `PCD_TCP0_AUXN`, `PCD_TCP0_AUXP`
    - **TCP1**: 2 lanes + AUX (10 pins)
    - **TCP2**: 2 lanes + AUX (10 pins)
    - **TCP3**: 2 lanes + AUX (10 pins)
    - **TCP4**: 2 lanes + AUX (10 pins)
    - **Total**: 50 pins (5 TCP ports)

11. **UFS** - Universal Flash Storage Interface
    - **2 lanes** (00-01), each with RX/TX differential pairs
      - `PCD_UFS_00_RXN`, `PCD_UFS_00_RXP`, `PCD_UFS_00_TXN`, `PCD_UFS_00_TXP`
      - `PCD_UFS_01_RXN`, `PCD_UFS_01_RXP`, `PCD_UFS_01_TXN`, `PCD_UFS_01_TXP`
    - **Total**: 8 pins (2 lanes × 4 pins)

12. **USB3** - USB 3.x Interface
    - **2 ports** (01-02), each with RX/TX differential pairs
      - `PCD_USB3_01_RXN`, `PCD_USB3_01_RXP`, `PCD_USB3_01_TXN`, `PCD_USB3_01_TXP`
      - `PCD_USB3_02_RXN`, `PCD_USB3_02_RXP`, `PCD_USB3_02_TXN`, `PCD_USB3_02_TXP`
    - **Total**: 8 pins (2 ports × 4 pins)

13. **EUSB2** - Enhanced USB 2.0 Interface
    - **8 ports**: `PCD_EUSB2_01N`, `PCD_EUSB2_01P` through `PCD_EUSB2_08N`, `PCD_EUSB2_08P`
    - **EUSB2V2**: `PCD_EUSB2V2_N`, `PCD_EUSB2V2_P`
    - **Total**: 18 pins (8 differential pairs + 1 V2 pair)

14. **Display/Panel** - Display and Panel Control
    - **Backlight Control**: `PCD_L_BKLCTL`, `PCD_L_BKLTEN`, `PCD_L_VDDEND`
    - **Display Port**: `PCD_DDSP_HPDALV`
    - **Total**: 4 pins

15. **MLK** - MLK Interface
    - Contains **3 pins**: `PCD_MLK_CLK`, `PCD_MLK_DATA`, `PCD_MLK_RST`

16. **TMUX** - Test Multiplexer
    - Contains **51 pins**: `PCD_TMUX_00` through `PCD_TMUX_50`

17. **LDO** - Low Dropout Regulators
    - Contains **5 pins**: `PCD_DDI_LDO`, `PCD_DMI_LDO`, `PCD_PCIE_D_LDO`, `PCD_PCIE_E_LDO`, `PCD_PCIE_LDO`

**PCD I/O Testing Block Total: 601 pins**

##### HUB I/O Testing Block

The HUB (Hub Die) I/O Testing Block contains DDR memory interface pins:

1. **DDR** - DDR Memory Interface
   - **xDQ (Data)**: 32 byte lanes, each with 8 data pins (00-07)
     - `HUB_DDR_00_xDQ_00` through `HUB_DDR_00_xDQ_07`
     - `HUB_DDR_01_xDQ_00` through `HUB_DDR_01_xDQ_07`
     - ... continues through ...
     - `HUB_DDR_31_xDQ_00` through `HUB_DDR_31_xDQ_07`
     - **Total**: 256 pins (32 byte lanes × 8 bits)
   
   - **DQS (Data Strobe)**: 32 differential pairs
     - `HUB_DDR_00_DQS_N`, `HUB_DDR_00_DQS_P` through `HUB_DDR_31_DQS_N`, `HUB_DDR_31_DQS_P`
     - **Total**: 64 pins (32 differential pairs)
   
   - **CLK (Clock)**: 16 differential pairs
     - `HUB_DDR_00_CLK_N`, `HUB_DDR_00_CLK_P` through `HUB_DDR_15_CLK_N`, `HUB_DDR_15_CLK_P`
     - **Total**: 32 pins (16 differential pairs)
   
   - **WCK (Write Clock)**: 32 differential pairs
     - `HUB_DDR_00_WCK_N`, `HUB_DDR_00_WCK_P` through `HUB_DDR_31_WCK_N`, `HUB_DDR_31_WCK_P`
     - **Total**: 64 pins (32 differential pairs)
   
   - **CCC (Command/Control/Clock)**: 8 groups, each with 9 pins (00-08)
     - `HUB_DDR_00_CCC_00` through `HUB_DDR_00_CCC_08`
     - `HUB_DDR_01_CCC_00` through `HUB_DDR_01_CCC_08`
     - ... continues through ...
     - `HUB_DDR_07_CCC_00` through `HUB_DDR_07_CCC_08`
     - **Total**: 72 pins (8 groups × 9 pins)
   
   - **DBI (Data Bus Inversion)**: 32 pins
     - `HUB_DDR_00_DBI_00` through `HUB_DDR_31_DBI_00`
     - **Total**: 32 pins
   
   - **ALERT**: 2 pins
     - `HUB_DDR_ALERT_0`, `HUB_DDR_ALERT_1`
     - **Total**: 2 pins

2. **CNV** - Converged Interface
   - **RCOMP**: `HUB_CNV_RCOMP`
   - **Total**: 1 pin

3. **PECI** - Platform Environment Control Interface
   - **Single pin**: `HUB_PECI`
   - **Total**: 1 pin

**HUB I/O Testing Block Total: 524 pins**



### Implementation

The first implementation of GenericPLT is happening on the product family called **Nova Lake**, with product code **NVL**. NVL is implementing 5 functional dielets into a product package.

**Dielets**:
- **CPU**: Contains all the compute cores with big cores and small cores. The dielet containing the CPU IPs is also known as CDIE. CDIEs will come in different flavors and they are identified by the number of small cores and big cores in a given configuration. Nomenclature is big core count + small core count, therefore a CDIE with 4 big cores and 8 small cores is referred to as CPU 4+8. NVL CDIEs will come in the following configurations: 2+0, 4+0, 4+8, 8+16, and 8+16 with bLLC.
- **GCD (Graphics Compute Die)**: Contains all the IPs related to Graphics processing. GCD is also known as GT. GCD dielets are commonly grouped firstly by the number of execution units (EU) they have. NVL GCDs will come in the following configurations: 32EU, 64EU, 192EU, 256EU, and 512EU.
- **HDIE (HUB Die)**: Referred to as HUB die. HUB is in charge of managing the traffic between dielets during normal operation mode. All dielets are interconnected to the HUB. For NVL family there are two types of HUB dielets referred to as: HUB and HUB-AX. HUBs for NVL family contain all 4 small cores.
- **PCD (Platform Controller Die)**: On NVL family it comes in two flavors: PCD-S targeted for Desktop product configurations and PCD-H for Mobile product configurations.
- **Base Dielet**: A passive silicon with just metal routings. CDIE, GCD, HUB and PCD dielets are assembled on top of the base die, allowing interconnections dielet to dielet or die to die. Also base die serves as interconnection between dielets and package.

**Important Note on RTL Pin Name Variability**: Dielets of the same type may have equal or similar pin names across them, but this depends entirely on how the RTL pin names are chosen. It may happen that two CDIEs coming from two different RTL modes could have pin name differences. For example, a TDO pin on one RTL can be called `xxTDO` and `xx_TDO_n` in another RTL. This RTL naming inconsistency is one of the core problems that GenericPLT addresses by providing a standardized naming layer independent of RTL-specific conventions.

CPU, GCD, HUB and PCD are assembled on top of the base die using micro solder bumps. It is assembled at Sort factories once the dielets have been singulated from the wafer. Once the dielets are assembled, such assembly is known as stacked die. The process of assembling the dielets at sort is known as Wafer Level Assembly, even though dielets are not anymore on the wafer, it is referred this way just probably because it happens at Sort where most of the processes are done at Wafer Level. The technology used for wafer stacking for NVL family is known as Foveros.

For HVM purposes, products are named by the total compute core count across all dielets. The total core count includes big cores and small cores from the CPU plus the small cores from the HUB (all HUBs contain 4 small cores). This HVM BOM naming convention uses the global processing capability rather than individual dielet breakdowns.

Dielets are recombined into different products for a given market segment. Products are tested using different Test Interface Units (TIUs): DT (Desktop), MB (Mobile), and MBMoP (Mobile with Memory-on-Package).

**Desktop Segment Product Configurations (DT TIU)**:
- **S i3 (8C)**: BOM `CLASS_NVL_S8C` - CPU 4+0, HUB, PCD-S, GCD 32EU
  - Total cores: 4 big + 0 small (CPU) + 4 small (HUB) = 8 cores
- **S i5 (16C)**: BOM `CLASS_NVL_S16C` - CPU 4+8, HUB, PCD-S, GCD 32EU
  - Total cores: 4 big + 8 small (CPU) + 4 small (HUB) = 16 cores
- **S i7 (28C)**: BOM `CLASS_NVL_S28C` - CPU 8+16, HUB, PCD-S, GCD 32EU
  - Total cores: 8 big + 16 small (CPU) + 4 small (HUB) = 28 cores
- **Sk w BLLC (28C)**: BOM `CLASS_NVL_S28CB` - CPU 8+16 with BLLC, HUB, PCD-S, GCD 32EU
  - Total cores: 8 big + 16 small (CPU) + 4 small (HUB) = 28 cores
- **Sk i9 w BLLC (52C)**: BOM `CLASS_NVL_S52C` - CPU 8+16 with BLLC, CPU 8+16 with BLLC, HUB, PCD-S, GCD 32EU
  - Total cores: 8 big + 16 small (CPU1) + 8 big + 16 small (CPU2) + 4 small (HUB) = 52 cores

**Mobile Segment Product Configurations (MB TIU)**:
- **UL 6C**: BOM `CLASS_NVL_UL6C` - CPU 2+0, HUB, PCD-H, GCD 32EU
  - Total cores: 2 big + 0 small (CPU) + 4 small (HUB) = 6 cores
- **U 8C**: BOM `CLASS_NVL_U8C` - CPU 4+0, HUB, PCD-H, GCD 64EU
  - Total cores: 4 big + 0 small (CPU) + 4 small (HUB) = 8 cores
- **H 16C**: BOM `CLASS_NVL_H16C` - CPU 4+8, HUB, PCD-H, GCD 64EU
  - Total cores: 4 big + 8 small (CPU) + 4 small (HUB) = 16 cores
- **P 16C**: BOM `CLASS_NVL_P16C` - CPU 4+8, HUB, PCD-H, GCD 192EU
  - Total cores: 4 big + 8 small (CPU) + 4 small (HUB) = 16 cores
- **HX 28C**: BOM `CLASS_NVL_HX28C` - CPU 8+16, HUB, PCD-H
  - Total cores: 8 big + 16 small (CPU) + 4 small (HUB) = 28 cores

**Mobile with Memory-on-Package Segment Product Configurations (MBMoP TIU)**:
- **AX 28C**: BOM `CLASS_NVL_AX28C` - CPU 8+16, HUB-AX, PCD-H, GCD 256EU or 512EU
  - Total cores: 8 big + 16 small (CPU) + 4 small (HUB-AX) = 28 cores
- **AX 16C**: BOM `CLASS_NVL_AX16C` - CPU 4+8, HUB-AX, PCD-H, GCD 256EU or 512EU
  - Total cores: 4 big + 8 small (CPU) + 4 small (HUB-AX) = 16 cores

### Problem Statement

Pin List (PLT) development and maintenance is a critical bottleneck in test program development:

**PLT Complexity & Maintenance:**
- PLT changes require complex validations
- PLT-induced issues may not be detectable until large volumes
- Different PLTs between packages (even 99% similar) are problematic to synchronize (e.g., MTL/ARL families)
- Auditing PLT or finding deltas between packages is complex and manual

**Inconsistency Across Test Flows:**
- PLTs differ between Sort and Class, blocking module/collateral reuse
- PLTs differ significantly between dielets (CPU PLT vs. GCD PLT)
- Pin names are inconsistent per function across packages (e.g., `xxTDO` vs. `Tdo`)
- Identical test flows (e.g., FUSE) require multiple collateral sets due to pin name differences

**Preliminary Definition Challenges:**
- PinDefinitions, time-domains, and timings depend on preliminary die/package definitions
- SIUs and TIUs are in constant change requiring continuous updates
- Products like NVL have multiple SIUs per dielet and multiple TIUs (3 reduced pin count, 3 full connectivity)

**Level/Timing Fragmentation:**
- Similar test requirements have separate, tailored levels/timings per dielet
- Naming includes specific acronyms (e.g., "STF" → "SSN") requiring complete rework
- 99% similar timings maintained separately (e.g., SSN on CPU vs. GCD)

**Pattern Time Domain Issues:**
- Digital pattern time domains contaminated with package-specific IO pins
- IO changes force validation of digital pin equations
- Digital content cannot be shared between packages due to IO contamination

**Pin Name Proliferation:**
- Pin names exceeding 35 characters impact iTuff sizes
- Long names complicate pattern debugging (expanding/collapsing windows)

**Key Concept**: Use test-function-based names (e.g., `PWR_CORE_VDD`, `MEM_CH0_DATA0`) instead of RTL-specific names (e.g., `VCC_CORE_0P8V`, `DDR5_CH0_DQ0`) or package names (e.g., `BALL_A12`, `PIN_G7`).

**Core Principle**: Name pins by **what you're testing**, not by the underlying hardware implementation.

**Critical Benefit**: Test programs remain **consistent and resilient to hardware changes** without requiring code modifications. When RTL or package designs change, only the mapping file needs updating - test programs stay unchanged.

## Development Guidelines

### Code Style and Standards

- **Consistency**: Follow the established generic naming patterns throughout the project
- **Resilience**: Design naming conventions that insulate test code from hardware changes
- **Conciseness**: Keep pin names short and meaningful (avoid >35 character names)
- **Readability**: Pin names should be self-explanatory and describe their test function
- **Reusability**: Enable collateral sharing across packages, dielets, Sort, and Class
- **Documentation**: Document all naming conventions, mapping rules, and examples
- **Modularity**: Keep pin categories organized (power, clock, data, control, etc.)
- **Stability**: Ensure generic names remain valid across multiple hardware revisions
- **Domain Separation**: Maintain clean separation between digital and IO pin domains
- **Test Flow Agnostic**: Avoid embedding test flow acronyms in pin names (no "STF", "SSN", etc.)

### Pin Naming Principles

#### General Structure Based on HVM Test Function
```
<TEST_FUNCTION>_<INSTANCE>_<SIGNAL>
```

- **TEST_FUNCTION**: HVM test category (PWR, CLK, MEM, PCIE, USB, etc.) - what you're testing
- **INSTANCE**: Specific instance or channel being tested (CH0, PORT1, LANE0, etc.)
- **SIGNAL**: Signal type or bit position under test (VDD, TX, RX, DATA0, etc.)

#### Naming Best Practices

1. **Use uppercase** for all pin names
2. **Use underscores** as separators
3. **Be descriptive of the test function**, not the hardware implementation
4. **Keep names concise** - avoid unnecessarily long names (target <35 characters)
5. **Group related test pins** with consistent prefixes
6. **Use numerical suffixes** for multi-bit signals (DATA0, DATA1, etc.)
7. **Avoid RTL-specific and package-specific terminology**
8. **Avoid test flow acronyms** (STF, SSN, etc.) that change between products
9. **Focus on testability** - what can be measured/validated in HVM
10. **Enable cross-package reuse** - same name should work for Sort, Class, all packages
11. **Separate digital and IO domains** - don't mix in pattern time domains

#### Common Prefixes by HVM Test Function

- **PWR_**: Power supply testing (PWR_CORE_VDD, PWR_IO_VDDQ) - voltage/current measurement
- **CLK_**: Clock signal testing (CLK_SYS_MAIN, CLK_REF_100M) - frequency/jitter validation
- **RST_**: Reset functionality testing (RST_SYS_N, RST_CORE_N) - reset behavior validation
- **MEM_**: Memory interface testing (MEM_CH0_DATA0, MEM_CH0_ADDR0) - memory I/O validation
- **PCIE_**: PCIe interface testing (PCIE_LANE0_TX, PCIE_LANE0_RX) - SerDes/link testing
- **USB_**: USB interface testing (USB_PORT0_DP, USB_PORT0_DM) - USB compliance testing
- **GPIO_**: General purpose I/O testing (GPIO_00, GPIO_01) - digital I/O validation
- **I2C_**: I2C bus testing (I2C_BUS0_SDA, I2C_BUS0_SCL) - bus protocol validation
- **SPI_**: SPI interface testing (SPI_CS0, SPI_MOSI, SPI_MISO) - SPI protocol validation

### Documentation Standards

- Document the rationale behind naming conventions based on HVM test requirements
- Provide clear mapping examples between generic test function names and RTL/package names
- Include interface-specific naming patterns for common test scenarios
- Maintain a glossary of common abbreviations and test function categories
- Add comments explaining the test function being validated
- Document test coverage expectations for each generic pin category

### Hardware Change Resilience Model

GenericPLT provides a critical isolation layer between test programs and hardware implementation:

```
Test Program Code (Generic Names)
        ↓
    Mapping File
        ↓
RTL Names ← → Package Pins
```

**Impact of Hardware Changes:**

| Change Type | Without GenericPLT | With GenericPLT |
|-------------|-------------------|-----------------|
| RTL rename | Modify all test code | Update mapping file only |
| Package change | Remap all pin refs | Update mapping file only |
| Silicon respin | Update test programs | Update mapping file only |
| New package variant | Port entire test suite | Add new mapping file |
| Sort to Class | Different PLTs block reuse | Same generic PLT enables reuse |
| Dielet changes | Separate PLTs per dielet | Unified generic PLT |
| SIU/TIU updates | Update all collaterals | Update mapping files only |
| Test flow rename | Update all level/timing names | No changes needed |

**Key Benefit**: Test programs achieve **hardware independence** through the generic naming abstraction layer.

### Collateral Reuse Model

GenericPLT enables maximum collateral reuse across the entire product ecosystem:

```
                    Generic PLT (Single Definition)
                              ↓
    ┌─────────────┬───────────┴───────────┬─────────────┐
    ↓             ↓                       ↓             ↓
  Sort          Class                   TIU1          TIU2
    ↓             ↓                       ↓             ↓
  CPU PLT       GCD PLT              Package A      Package B
```

**Reuse Benefits:**
- **Cross-Package**: Same collaterals work for all package variants
- **Cross-Dielet**: CPU, GCD, and all dielets share collaterals
- **Cross-Flow**: Sort and Class share modules without modification
- **Cross-TIU**: Multiple SIUs/TIUs supported via single generic definition
- **Level/Timing**: 99% similar definitions become 100% identical

## GenericPLT Implementation Strategy

### Initiative 1: Function-Based Generic Pin Names
**Objective**: Define generic pin names at TP level based on function, not RTL names, with shorter names for heavily-used content pins.

**Guidelines**:
- Pin names describe test function, not RTL implementation
- Keep names concise for heavily-used pins (target <35 characters)
- Enable content sharing across dielets and between Sort/Class
- Design for timings and levels sharing (full or partial)
- Create smaller, more manageable sets of levels/timings

**Expected Outcomes**:
- Major controllability improvements
- Intrinsic goodness through shared collaterals
- Reduced maintenance overhead

### Initiative 2: Superset Pin Definition (NVL Family Example)
**Objective**: Define a generic pin definition that is a superset of all pins used across the product family for each dielet, covering all packages at Class and Sort for content (patterns).

**Guidelines**:
- Create superset that covers all package variants
- Include all pins needed for Sort and Class testing
- Focus on pins used in patterns/content
- Independent of preliminary T-Specs

**Expected Outcomes**:
- Faster TVPV enablement and content generation
- No need to wait for T-Specs to begin development
- Single definition works across entire product family

### Initiative 3: Unified Digital Content Pattern Headers
**Objective**: Digital content pattern headers use the same superset across all dielets for Sort and Class.

**Guidelines**:
- Separate digital and IO pin domains completely
- Digital patterns should not include package-specific IO pins
- Use consistent digital pin definitions across all dielets
- Enable pattern sharing without IO contamination concerns

**Expected Outcomes**:
- Digital testing decoupled from IO testing
- Faster sharing without worrying about IO changes
- Clean separation of concerns
- Pattern reuse without validation issues

### Initiative 4: Superset IO Content Pattern Headers
**Objective**: IO content pattern headers use a superset for a given dielet (e.g., PCD-H and PCD-S superset).

**Guidelines**:
- Define IO supersets per dielet type
- Cover all package variants for that dielet
- Keep IO pins separate from digital content
- Enable SuperSet module approach

**Expected Outcomes**:
- SuperSet modules enable broader reuse
- IO-specific changes don't impact digital content
- Simplified IO testing across package variants

### Initiative 5: Future Product Dielet Reuse
**Objective**: Enable future products to reuse dielets with complete content/collateral portability.

**Guidelines**:
- Design generic names to be product-family agnostic
- Use same GenericPLT convention across product generations
- Maintain POBJ compatibility where possible
- Document reuse assumptions and dependencies

**Expected Outcomes**:
- If future product reuses a dielet with same GenericPLT: plug-and-play content/collaterals/modules
- If POBJ is the same: direct reuse on new product families
- TVPV/TP partial infrastructure reuse even for modified dielets
- Dramatic reduction in new product bring-up time

### File Organization

```
GenericPLT/
├── conventions/          # Pin naming convention documentation
│   ├── power.md         # Power pin naming rules
│   ├── clocks.md        # Clock pin naming rules
│   ├── memory.md        # Memory interface naming rules
│   └── io.md            # I/O interface naming rules
├── examples/            # Example pin mappings
│   ├── ddr5_mapping.csv # DDR5 interface example
│   ├── pcie_mapping.csv # PCIe interface example
│   └── usb_mapping.csv  # USB interface example
├── tools/               # Conversion and validation utilities
│   ├── converters/      # RTL to generic name converters
│   └── validators/      # Pin list validation tools
├── templates/           # Pin list templates
└── docs/                # Comprehensive documentation
```

### Mapping File Standards

#### CSV Format
```csv
GenericName,RTLName,PackagePin,TestFunction,Description,TestSpecs
PWR_CORE_VDD,VCC_CORE_0P8V,BALL_A12,Power Supply Test,Core power supply,0.8V ±5%
CLK_SYS_MAIN,PLL_OUT_100MHZ,PIN_G7,Clock Frequency Test,Main system clock,100MHz ±100ppm
MEM_CH0_DATA0,DDR5_CH0_DQ0,BALL_C15,Memory IO Test,Memory data bit 0,JEDEC DDR5
```

#### Required Fields
- **GenericName**: The standardized test-function-based name
- **RTLName**: The silicon-specific RTL name
- **PackagePin**: The physical package pin/ball name
- **TestFunction**: The HVM test being performed
- **Description**: Human-readable description
- **TestSpecs**: Test specifications and acceptance criteria

### Validation Rules

When creating or reviewing pin mappings:
- [ ] Generic names follow the established test-function-based convention
- [ ] Generic names describe the HVM test function, not RTL or package naming
- [ ] Generic names are stable and won't need to change with hardware revisions
- [ ] Pin names are concise and under 35 characters where possible
- [ ] Pin names avoid test flow acronyms (STF, SSN, etc.)
- [ ] All RTL pins and package pins are mapped to generic names
- [ ] No duplicate generic names exist
- [ ] Test functions are correctly categorized
- [ ] Test specifications and acceptance criteria are documented
- [ ] Descriptions are clear and test-focused
- [ ] Multi-bit signals use consistent numbering (0-based)
- [ ] Differential pairs are clearly identified (_P/_N suffixes)
- [ ] Mapping is independent of silicon revision and package type
- [ ] Changes to RTL or package only require mapping file updates, not naming changes
- [ ] Digital and IO pin domains are cleanly separated
- [ ] Same generic names work across Sort, Class, and all packages
- [ ] Collateral reuse across dielets is enabled

### Common Patterns and Examples (HVM Test Function Focus)

#### Power Test Pins
```
Generic Name (Test)   RTL Name            Package Pin    HVM Test Function
PWR_CORE_VDD     →    VCC_CORE_0P8V      BALL_A12       Core voltage measurement
PWR_IO_VDDQ      →    VDDQ_1P2V_IO       BALL_B15       I/O voltage measurement
PWR_PLL_AVDD     →    AVDD_PLL_1P0V      PIN_C3         PLL analog supply test
PWR_GND          →    VSS_GND            BALL_D5        Ground reference validation
```

#### Memory Interface Testing (DDR5 Example)
```
Generic Name (Test)   RTL Name            Package Pin    HVM Test Function
MEM_CH0_CLK_P    →    DDR5_CH0_CK_P      BALL_E12       Channel 0 clock test
MEM_CH0_CLK_N    →    DDR5_CH0_CK_N      BALL_E13       Channel 0 clock test
MEM_CH0_DATA0    →    DDR5_CH0_DQ0       PIN_F10        Channel 0 data I/O test
MEM_CH0_ADDR0    →    DDR5_CH0_A0        BALL_G8        Channel 0 address test
MEM_CH0_CS_N     →    DDR5_CH0_CS0_N     PIN_H5         Channel 0 chip select test
```

#### PCIe Interface Testing
```
Generic Name (Test)   RTL Name            Package Pin    HVM Test Function
PCIE_LANE0_TX_P  →    PCIE_GEN5_TX0_P    BALL_J12       Lane 0 TX SerDes test
PCIE_LANE0_TX_N  →    PCIE_GEN5_TX0_N    BALL_J13       Lane 0 TX SerDes test
PCIE_LANE0_RX_P  →    PCIE_GEN5_RX0_P    PIN_K10        Lane 0 RX SerDes test
PCIE_LANE0_RX_N  →    PCIE_GEN5_RX0_N    PIN_K11        Lane 0 RX SerDes test
PCIE_REFCLK_P    →    PCIE_REFCLK_100M_P BALL_L8        Reference clock test
```

#### USB Interface Testing
```
Generic Name (Test)   RTL Name            Package Pin    HVM Test Function
USB_PORT0_DP     →    USB3_P0_SSTX_P     PIN_M5         Port 0 USB3 compliance
USB_PORT0_DM     →    USB3_P0_SSTX_N     PIN_M6         Port 0 USB3 compliance
USB_PORT0_VBUS   →    USB_P0_VBUS_5V     BALL_N3        Port 0 VBUS power test
```

### Version Control

- Write clear commit messages describing naming convention changes
- Document the rationale for new naming patterns
- Track mapping file changes carefully
- Use feature branches for new interface conventions
- Tag releases when major conventions are finalized

### AI Assistance Best Practices

When using GitHub Copilot for this project:

1. **Consistency**: Always follow the established naming patterns
2. **Resilience**: Ensure suggested names won't break when hardware changes
3. **Validation**: Verify suggested pin names match the convention structure
4. **Context**: Consider the interface type when generating names
5. **Documentation**: Ensure all mappings include clear descriptions
6. **Standards**: Cross-reference with existing examples before creating new patterns
7. **Stability**: Design names that remain valid across hardware revisions

### Naming Convention Checklist

Before adding or modifying pin names:
- [ ] Follows `<TEST_FUNCTION>_<INSTANCE>_<SIGNAL>` structure
- [ ] Uses uppercase letters only
- [ ] Uses underscores as separators
- [ ] Clearly describes the HVM test function being performed
- [ ] Pin name is concise (ideally <35 characters) - especially for heavily-used content pins
- [ ] Avoids RTL-specific and package-specific terminology
- [ ] Avoids test flow acronyms (STF, SSN, etc.)
- [ ] Focuses on testability and measurement, not implementation
- [ ] Remains valid across hardware changes (resilient naming)
- [ ] Consistent with similar test pin names in the same category
- [ ] Multi-bit signals use zero-based numbering
- [ ] Differential pairs use _P/_N suffixes
- [ ] Test specifications and acceptance criteria are documented
- [ ] Works across different silicon revisions and package types
- [ ] Works across Sort, Class, and all TIUs
- [ ] Works across all dielets (CPU, GCD, etc.)
- [ ] Part of superset definition for product family (Initiative 2)
- [ ] Properly categorized as digital or IO domain (Initiatives 3 & 4)
- [ ] Documented in the appropriate convention file
- [ ] Enables test program stability when hardware evolves
- [ ] Enables collateral reuse across product family
- [ ] Maintains clean digital/IO domain separation
- [ ] Designed for future product portability (Initiative 5)

### Priority Features for Development

Based on the five GenericPLT initiatives, prioritize:

1. **HVM Test Function Documentation**: Complete guides for all major HVM test categories (Initiative 1)
2. **Superset Pin Definition Templates**: Product family superset definitions (Initiative 2)
3. **Digital/IO Domain Separation**: Guidelines and templates for clean separation (Initiatives 3 & 4)
4. **Cross-Package Templates**: Templates that work across Sort, Class, and all package variants (Initiative 2)
5. **Dielet Unification Examples**: Show how CPU, GCD, and other dielets share generic names (Initiative 1)
6. **Future Product Portability Guide**: Documentation for POBJ reuse and plug-and-play content (Initiative 5)
7. **Mapping Examples**: Comprehensive examples showing generic → RTL → package mappings
8. **Conversion Tools**: Utilities to translate between generic, RTL, and package pin names
9. **Validation Tools**: Scripts to verify pin list compliance with test-function-based conventions
10. **Integration Guides**: Documentation for using generic names with ATE test platforms
11. **Test Coverage Matrix**: Documentation of test functions mapped to generic pin categories
12. **Pin Name Length Optimization**: Guidelines for keeping names concise yet descriptive (Initiative 1)
13. **Multi-SIU/TIU Support**: Documentation for handling complex product configurations

### Helpful Resources

- Project Repository: https://github.com/jonathan-urtecho-intel/GenericPLT
- Issue Tracker: https://github.com/jonathan-urtecho-intel/GenericPLT/issues
- Discussions: https://github.com/jonathan-urtecho-intel/GenericPLT/discussions

---

**Note**: These instructions should be updated as the project evolves. Maintainers should review and refine these guidelines based on project needs and lessons learned.
