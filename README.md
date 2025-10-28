# GenericPLT

[![GitHub](https://img.sh- **Eliminates Synchronization Issues**: One generic PLT for entire product family

## Features

- 🎯 HVM test-function-based pin naming convention
- 🛡️ Test programs remain unchanged when hardware evolves
- 🔄 Cross-package, cross-dielet collateral reuse
- 📋 Standardized pin list templates for product families
- 🔧 Mapping utilities between generic, RTL, and package names
- 📝 Comprehensive naming guidelines and examples
- 🔒 Hardware changes isolated to mapping files only
- 🧹 Clean separation of digital and IO pin domains
- 📏 Concise pin names for better debugging and iTuff efficiency
- 🔗 Unified PLT across Sort, Class, and all test flows
- 🎭 Multi-SIU/TIU support through abstraction layer

## Getting Starteddge/GitHub-Repository-blue)](https://github.com/jonathan-urtecho-intel/GenericPLT)

A standardized pin naming convention for test programs based on HVM testing function rather than RTL or package pin names.

## Overview

GenericPLT (Generic Pins, Levels and Timings) defines a consistent naming convention for test program pins based on their **HVM (High Volume Manufacturing) testing function** rather than their RTL (Register Transfer Level) design names or physical package pin names.

### Key Terminology

- **PLT**: Pins, Levels and Timings in an ATE (Automated Test Equipment) Test Program
- **PLTA**: PLT Architect - the role responsible for defining and maintaining PLT
- **GenericPLT Process**: Starts with the PLTA defining a possible list of Generic PinNames that will be used for a given dielet or family of dielets, which can work across testing operations like Sort, Class, and BurnIn
  - The PLTA begins by assessing possible RTL pin names for all the dielets involved in a given product
  - From this assessment, the PLTA creates generic pin names based on HVM testing function
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
- **TAP**: Test Access Port pins
- **RESET**: Reset execution and power good pins
- **MBP**: Microbreakpoint pins
- **TESTPORT**: Dielet fabric test port operations
- **OBSERVABILITY**: Internal signals observability
- **PROBE_TRIGGER**: FA/FA debug trigger pins

**Note**: All dielets will require a digital testing block. GenericPLT digital testing block uses the same generic pin names across all dielet types (CPU, GCD, HUB, PCD). While the actual test patterns cannot be directly reused across different dielet types due to different RTL behaviors, the infrastructure to generate patterns for ATE (pattern headers, pin definitions, tooling) can be the same, enabling significant reuse of test program infrastructure and development processes.

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

### The Problem

Pin List (PLT) development and maintenance creates significant bottlenecks in test program development:

- **Complex Validation**: PLT changes require extensive validation; issues may only surface at high volumes
- **Package Variations**: Different PLTs between packages (even 99% similar) make synchronization problematic
- **Sort vs. Class Differences**: Different PLTs block module and collateral reuse between test flows
- **Manual Auditing**: Finding PLT deltas between packages is complex and manual
- **Dielet Inconsistency**: PLTs differ significantly across dielets (e.g., CPU vs. GCD)
- **Inconsistent Pin Names**: Same functional pin has different names across packages (e.g., `xxTDO` vs. `Tdo`)
- **Collateral Duplication**: Identical test flows require separate collaterals due to pin name differences
- **Level/Timing Fragmentation**: Similar test requirements have tailored levels/timings per dielet
- **Pattern Time Domain Contamination**: Digital patterns mixed with package-specific IOs
- **Long Pin Names**: Names exceeding 35 characters impact iTuff sizes and pattern debugging
- **Constant Updates**: Preliminary die/package definitions and SIUs/TIUs in constant flux
- **Multi-SIU/TIU Complexity**: Products like NVL have multiple SIUs per dielet and multiple TIUs

### The Solution

GenericPLT addresses these challenges by providing a stable, function-based naming convention that:

- **Eliminates PLT-Induced Bottlenecks**: Reduces validation cycles and PLT maintenance overhead
- **Enables Cross-Package Reuse**: Same generic names work across all packages (Sort, Class, different TIUs)
- **Unifies Dielet Testing**: Consistent pin names across CPU, GCD, and all dielets
- **Simplifies Module Sharing**: Identical test flows share collaterals without modification
- **Removes Pattern Contamination**: Clean separation between digital and IO pin domains
- **Shorter, Clearer Names**: Function-based names are concise and debugging-friendly
- **Absorbs SIU/TIU Changes**: Preliminary definition changes only update mapping files
- **Test Program Stability**: Test programs remain consistent across hardware revisions
- **Hardware Change Resilience**: RTL or package changes don't require test program modifications
- **Zero Test Code Impact**: Hardware evolution is absorbed by mapping files, not test code
- **Eliminates Synchronization Issues**: One generic PLT for entire product family

## Features

- 🎯 Function-based pin naming convention
- � Platform-agnostic test program development
- � Standardized pin list templates
- � Mapping utilities between generic and RTL names
- 📝 Comprehensive naming guidelines and examples

## Getting Started

### Prerequisites

- [Git](https://git-scm.com/)
- Understanding of test program development
- Familiarity with pin list structures

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/jonathan-urtecho-intel/GenericPLT.git
   cd GenericPLT
   ```

2. Review the naming convention guidelines
3. Integrate the generic pin naming standards into your test program development workflow

### Quick Start

```text
Example: Converting RTL/Package names to Generic Pin Names based on HVM Test Function

RTL/Package Name:   Generic Name (Test Function):
DDR5_CH0_DQ0    →   MEM_CHAN0_DATA0      (Memory channel test)
PCIE_GEN5_TX0   →   PCIE_LANE0_TX        (PCIe lane test)
USB_DP_PORT1    →   USB_PORT1_DP         (USB port test)
BALL_A12        →   PWR_CORE_VDD         (Core power test)
```

## Usage

### Basic Pin Naming Convention

The generic pin naming follows this structure based on **HVM test function**:

```
<TEST_FUNCTION>_<INSTANCE>_<SIGNAL>
```

**Key Principle**: Name pins by what you're testing, not by RTL or package naming.

**Examples:**
- `PWR_CORE_VDD` - Testing core voltage (regardless of RTL name like VCC_CORE or package ball name)
- `CLK_SYS_MAIN` - Testing main system clock functionality
- `MEM_CH0_DATA0` - Testing memory channel 0, data bit 0
- `PCIE_LANE0_TX` - Testing PCIe lane 0 transmit functionality

### Mapping Files

Create mapping files to translate between generic test function names and RTL/package names:

```csv
GenericName,RTLName,PackagePin,TestFunction,Description
PWR_CORE_VDD,VCC_CORE_0P8V,BALL_A12,Power Supply Test,Core power supply
CLK_SYS_MAIN,PLL_OUT_100MHZ,PIN_G7,Clock Frequency Test,Main system clock
MEM_CH0_DATA0,DDR5_CH0_DQ0,BALL_C15,Memory IO Test,Memory data bit 0
```

### Integration with Test Programs

1. Define your pin list using generic names based on **HVM test function**
2. Create a mapping file for your specific silicon (RTL to generic) and package (ball/pin to generic)
3. Use translation utilities to generate platform-specific test programs
4. Maintain test code using generic names for portability across different silicon revisions and package types
5. **Hardware changes only require updating the mapping file** - test programs remain unchanged
6. Test functions remain consistent even when RTL or package implementations change

### Benefits in Practice

**Without GenericPLT:**
- Pin name inconsistency: `xxTDO` in one package, `Tdo` in another → Cannot share collaterals
- Level/Timing changes: `STF` renamed to `SSN` → Modify all levels/timings manually
- Package variant: MTL to ARL → Manually synchronize 99% similar PLTs
- Dielet differences: CPU vs. GCD → Maintain duplicate collaterals for identical flows
- Pattern contamination: Digital patterns mixed with IO pins → Complex equations, limited reuse
- SIU/TIU updates: Preliminary changes → Update all test programs and collaterals
- Result: Extensive test program modifications, validation, regression testing, and maintenance overhead

**With GenericPLT:**
- Pin name consistency: `JTAG_TDO` across all packages → Share collaterals everywhere
- Level/Timing stability: Generic names don't include test flow acronyms → No changes needed
- Package variants: Same generic PLT → Automatic synchronization via mapping files
- Dielet unification: Same generic names (e.g., `FUSE_*`) → Single set of collaterals
- Clean patterns: Separate digital and IO domains → Simple equations, maximum reuse
- SIU/TIU updates: Mapping file changes only → Test programs unchanged
- Result: **Test programs require zero changes**, maximum collateral reuse, minimal maintenance

### Real-World Impact

| Scenario | Traditional Approach | GenericPLT Approach |
|----------|---------------------|---------------------|
| **Cross-Package Testing** | Duplicate collaterals for each package | Single collateral set for all packages |
| **Sort to Class Transition** | Different PLTs block module reuse | Same generic PLT enables seamless reuse |
| **Dielet Variations** | Separate PLTs for CPU, GCD, etc. | Unified generic PLT across all dielets |
| **SIU/TIU Changes** | Update test programs + collaterals | Update mapping files only |
| **Pin Name Synchronization** | Manual auditing and delta tracking | Automatic via generic naming |
| **Pattern Time Domain Cleanup** | Digital patterns contaminated with IOs | Clean separation by function |
| **Level/Timing Reuse** | 99% similar but maintained separately | Single generic definition |
| **TVPV Enablement** | Wait for T-Specs and preliminary definitions | Start immediately with generic superset |
| **Future Product Reuse** | Port and validate all content | Plug-and-play if same dielet/POBJ |

## Project Structure

```
GenericPLT/
├── .github/              # GitHub-specific files
│   └── copilot-instructions.md
├── conventions/          # Naming convention guidelines (to be added)
├── examples/             # Example pin mappings (to be added)
├── tools/                # Conversion utilities (to be added)
└── README.md             # This file
```

## Documentation

- [GitHub Copilot Instructions](.github/copilot-instructions.md) - Development guidelines for AI assistance
- Naming Convention Guide (coming soon)
- Pin Mapping Best Practices (coming soon)
- Integration Examples (coming soon)

## Contributing

Contributions are welcome! To contribute to this project:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

Please ensure your code follows the project's coding standards and includes appropriate tests.

### Development Guidelines

- Write clear, descriptive commit messages
- Add tests for new features
- Update documentation as needed
- Follow existing code style and conventions

## Support

If you encounter any issues or have questions:

- **Issues**: [GitHub Issues](https://github.com/jonathan-urtecho-intel/GenericPLT/issues)
- **Discussions**: [GitHub Discussions](https://github.com/jonathan-urtecho-intel/GenericPLT/discussions)

## Roadmap

### GenericPLT Initiatives

GenericPLT implementation follows five key initiatives:

#### 1. Function-Based Generic Pin Names
**Objective**: Define generic pin names at TP level based on function, not RTL names, with shorter names for heavily-used content pins.

**Benefits**:
- Potential content sharing across dielets, between Sort and Class
- Timings and levels sharing (full or partial) among all dielets
- Smaller, more manageable sets of levels/timings
- Major controllability improvements
- Intrinsic goodness through shared collaterals

#### 2. Superset Pin Definition for NVL Family
**Objective**: Define a generic pin definition that is a superset of all pins used across the NVL family for each dielet, covering all packages at Class and Sort for content (patterns).

**Benefits**:
- Faster TVPV enablement and content generation
- No need to wait for T-Specs to begin development
- Single definition works across entire product family
- Eliminates package-specific pin list variations

#### 3. Unified Digital Content Pattern Headers
**Objective**: Digital content pattern headers use the same superset across all dielets for Sort and Class.

**Benefits**:
- Digital testing decoupled from IO testing
- Faster sharing without worrying about IO changes
- Clean separation of concerns
- Pattern reuse without contamination issues

#### 4. Superset IO Content Pattern Headers
**Objective**: IO content pattern headers use a superset for a given dielet (e.g., PCD-H and PCD-S superset).

**Benefits**:
- SuperSet modules enable broader reuse
- IO-specific changes don't impact digital content
- Simplified IO testing across package variants

#### 5. Future Product Dielet Reuse
**Objective**: Enable future products to reuse dielets with complete content/collateral portability.

**Benefits**:
- If a future product reuses a dielet, all content, collaterals, and modules are plug-and-play (with same GenericPLT)
- If POBJ is the same, direct reuse on new product families
- TVPV/TP partial infrastructure reuse even for modified dielets
- Dramatic reduction in new product bring-up time

---

### Future Enhancements

Future enhancements planned for GenericPLT:
- [ ] Complete naming convention documentation
- [ ] Pin category definitions (power, clock, data, control, etc.)
- [ ] Example pin mapping files for common interfaces
- [ ] Automated conversion tools between generic and RTL names
- [ ] Validation utilities for pin list compliance
- [ ] Integration examples with popular test platforms

## Maintainers

- **Jonathan Urtecho** - [@jonathan-urtecho-intel](https://github.com/jonathan-urtecho-intel)

## License

This project's license will be specified as development progresses. Please check back for updates.

## Acknowledgments

- Built with care for the testing community
- Contributions and feedback are appreciated

---

**Note**: This project is in early development. Features and documentation will be expanded as the project evolves. Star ⭐ the repository to stay updated!
