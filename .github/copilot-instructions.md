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
