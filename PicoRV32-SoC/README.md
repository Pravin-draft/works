# SOC_PnR_Picrorv32
Open Source Digital ASIC Design requires three open-source components:

RTL Designs = github.com, librecores.org, opencores.org
EDA Tools = OpenROAD, OpenLANE,QFlow
PDK = Google + Skywater 130nm Production PDK


![215494429-0618b809-56e7-4206-bb10-d468faebf570](https://github.com/user-attachments/assets/1c9e3143-5ec4-4805-859b-b85937fb7c69)


**ASIC design Flow:**
Sythesis : RTL is converted into a gate level netlist made up of components of standard cell libary (SCL).
Floor Planning/ Power Planning : Objective is to plan silicon area and create robust power distribution network. The power network usually uses the upper metal layer which are thicker than lower layer and thus lower resistance. This lowers the IR drop problem
Placement : There are two steps, first is global placement which is the general optimal positons for cells and might not be legal. Next is detailed placement which is the actual legal placements of the cells from global placement.
Clock tree synthesis : it delivers clock to all clock cells with minimum skew & latency and is usually a tree (H-tree, X-tree ... )
Routing : Use horizontal and vertical wires to connect cells together. The router uses PDK information (thickness, pitch, width,vias) for each metal layer to do the routing. The Sky130 defines 6 routing layers. It do global routing (use coarse grain grids) and detailed routing (use fine grain grids).
Verification before sign-off : Involves physical verification like DRC and LVS and timing verification. Design Rule Checking or DRC ensures final layout honors all design rules and Layout versus Schematic or LVS ensures final layout matches the gate level netlist from synthesis phase. Timing verification ensures timing constraints are met.


PDK (Process Design Kit) = A set of data files and documents which serves as the interface between the designer and the fab. This includes cell libraries, IO libraries, design rules (DRC, LVS, etc.)

![215494588-e33e8ebd-0c3c-4187-b807-b04f41de1946](https://github.com/user-attachments/assets/feeffd1b-194c-4df1-9623-56ec90a220a9)

**What is Openlane?**
OpenLane = An open-source ASIC development flow reference. It consists of multiple open-source tools needed for the whole RTL to GDSII flow. This is tuned epecially for Sky130 PDK. It also works for OSU 130nm. It is recommended to read the OpenLANE documentation before moving forward.


![182759711-6b9352ec-7652-4589-af31-53a409eb2830](https://github.com/user-attachments/assets/1ba77ede-b59b-4690-a101-e3a936e83c31)

The input for the whole flow are the rtl files, sdc file, and PDK files. The output is GDSII/LEF file.

Yosys is used to convert the HDL to gate level netlist using generic components. The ABC script is then used to map the generic components to the standard cell library of the PDK. These ABC scripts is used to make various synthesis strategies (using the Synthesis Exploration) which can optimize the design either with least area or best timing.

The Logic Equivalency Cheking (LEC) is used to compare the resulting netlist after optimization of place and route to the gate level netlist from synthesis phase

Antenna Rules Violation = long wire segments will act as antennna and will accumulate charges, this might damage the connected transistor gates. Solution is to either use bridging or antenna diode insertion to leak away the charges




**Run OpenLANE:**

cd work/tools/openlane_working_dif/openlane = set this directory

`docker` = Open the docker platform inside the openlane/

`% flow.tcl -interactive` = run script for automating the whole RTL to GDSII flow but in step by step -interactive mode

`% package require openlane 0.9 `= retrives all dependencies for running v0.9 of OpenLANE


![Screenshot from 2025-01-22 15-37-37](https://github.com/user-attachments/assets/04b9ed16-17d6-42b0-8d20-3b75fa2ab691)

**Design Setup Stage:**

`% prep -design picorv32a` = Setup the filesystem where the OpenLANE tools can dump the outputs. This also creates a run/ folder inside the specific design directory which contains the command log files, results, and the reports dump by each tools.


![Screenshot from 2025-01-22 15-57-02](https://github.com/user-attachments/assets/856abc3e-6ca6-4f01-8fa2-1935b70d77ff)

**Run synthesis:**

`% run_synthesis` = Run yosys RTL synthesis, ABC scripts (for technology mapping), and OpenSTA.


![Screenshot from 2025-01-22 16-05-34](https://github.com/user-attachments/assets/e021e6c0-2951-4b79-98c2-c9394228f1ed)


![Screenshot from 2025-01-22 16-07-03](https://github.com/user-attachments/assets/1147d72d-9a85-4097-9f66-e4663583f641)

**Floorplan Stage:**
Define height and width of core and die.
Core is where the logic blocks are placed and this seats at the center of the die. The width and height depends on dimensions of each standard cells on the netlist. Utilization factor is (area occupied by netlist)/(total area of the core) In practical scenario, utilization factor is 0.5 to 0.6. This is space occupied by netlist only, the remaining space is for routing and more additional cells. Aspect ratio is (height)/(width) Aspect ratio of core, so only aspect ratio of 1 will produce a square core shape.

Define location of Preplaced Cell.
These are reusable complex logicblocks or modules or IPs or macros that is already implemented (memory, clock-gating cell, mux, comparator...) . The placement on the core is user-defined and must be done before placement and routing (thus preplaced cells). The automated place and route tools will not be able to touch and move these preplaced cells so this must be very well defined

Surround preplaced cells with decoupling capacitors. The complex preplaced logicblock requires a high amount of current from the powersource for current switching. But since there is a distance between the main powersource and the logicblock, there will be voltage drop due to the resistance and inductance of the wire. This might cause the voltage at the logicblock to be not within the noise margin range anymore (logic is unstable). The solution is to use decoupling capacitors near the logic block, this capacitor will send enough current needed by the logicblock to switch within the noise margin range.

![Screenshot from 2025-01-22 19-02-12](https://github.com/user-attachments/assets/59838ef9-f721-47cd-8d5a-e74651816227)
![Screenshot from 2025-01-22 19-26-41](https://github.com/user-attachments/assets/c803a8fd-731e-4418-947a-6f1384adcc71)

Power Planning Decoupling capactor for sourcing logic blocks with enough current is not feasible to be applied all over the chip but only on the critical elements (preplaced complex logicblocks). Large number of elements switching to logic 0 might cause ground bounce due to large amount of current that needs to be sink at the same time, and switcing to logic 1 might cause voltage droop due to not enough current from the powersource to source needed current of all elements. Ground bounce and voltage droop might cause the voltage to not be within the noise margin range. The solution is to have multiple powersource taps (power mesh) where elements can source current from the nearest VDD and sink current to the nearest VSS tap. This is the reason why most chips have multiple powersource pins.

Pin Placement The input and output ports are placed on the space between the core and the die. The placements of the ports depens on where the cells connected to those ports are placed on the core. The clock port is thicker(least resistance path) than data ports since this clock must be capable to drive the whole chip.

Logical Cell Placement Blockage This makes sure that the automated placement and routing tool does not place any cell on the pin locations of the die.


![Screenshot from 2025-01-22 19-36-43](https://github.com/user-attachments/assets/a11b9292-2e57-4c77-944a-434caca8452e)
![Screenshot from 2025-01-22 19-38-02](https://github.com/user-attachments/assets/cbedf4c5-26e5-4c35-ac9e-31abfa11d6a3)
![Screenshot from 2025-01-22 19-38-14](https://github.com/user-attachments/assets/414a18a9-4aa3-49c9-bade-90bbc10b4924)

To run floorplan`run_floorplan`
   
![Screenshot from 2025-01-23 12-56-36](https://github.com/user-attachments/assets/c3e2cf37-2b58-4214-b172-f8baa70017a9)
![Screenshot from 2025-01-23 16-00-23](https://github.com/user-attachments/assets/3f240d94-065d-432b-b0c6-dc9d1f16fdca)

To center the view, press "s" to select whole die then press "v" to center the view. Point the cursor to a cell then press "s" to select it, zoom into it by pressing 'z". Type "what" in tkcon to display information of selected object. These objects might be IO pin, decap cell, or well taps as shown below.

Run placement:
This commmand is a wrapper which does global placement (performed by RePlace tool), Optimization (by Resier tool), and detailed placement (by OpenDP tool). It displays hundreds of iterations displaying HPWL and OVFL. The algorithm is said to be converging if the overflow is decreasing. It also checks the legality.

![Screenshot from 2025-01-23 16-07-49](https://github.com/user-attachments/assets/7e6446c7-c437-4932-a334-583b1ee7d23d)

**Designing a Library Cell:**
SPICE deck = component connectivity (basically a netlist) of the CMOS inverter.
SPICE deck values = value for W/L (0.375u/0.25u means width is 375nm and lengthis 250nm).
PMOS should be wider in width(2x or 3x) than NMOS.
The gate and supply voltages are normally a multiple of length (in the example, gate voltage can be 2.5V)
Add nodes to surround each component and name it. This will be used in SPICE to identify a component.


![Screenshot from 2025-01-23 18-13-27](https://github.com/user-attachments/assets/e98c0691-45f7-451b-8889-86f61887de5c)

**CMOS Fabrication Process (16-Mask CMOS Process):**


![Screenshot from 2025-01-24 12-17-42](https://github.com/user-attachments/assets/21bd86a1-3412-4128-ace6-a964ff280f40)

**Delay Table:**
In order to avoid large skew between endpoints of a clock tree (signal arrives at different point in time):

Buffers on the same level must have same capacitive load to ensure same timing delay or latency on the same level.
Buffers on the same level must also be the same size (different buffer sizes -> different W/L ratio -> different resistance -> different RC constant -> different delay).

Buffers on different level will have different capacitive load and buffer size but as long as they are the same load and size on the same level, the total delay for each clock tree path will be the same thus skew will remain zero. This means different levels will have varying input transition and output capacitive load and thus varying delay.

Delay tables are used to capture the timing model of each cell and is included inside the liberty file. The main factor in delay is the output slew. The output slew in turn depends on capacitive load and input slew. The input slew is a function of previous buffer's output cap load and input slew and it also has its own transition delay table.

![Screenshot from 2025-01-27 11-53-49](https://github.com/user-attachments/assets/90cde1c2-2cb3-4be3-8f66-932ea75e2ce4)

**Setup Timing Analysis :**
Create the pre_sta.conf and save it in the openlane folder.
`set_cmd_units -time ns -capacitance pF -current mA -voltage V -resistance kOhm -distance um
read_liberty -min /home/abhinavprakash1999/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib
read_liberty -max /home/abhinavprakash1999/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib
read_verilog /home/abhinavprakash1999/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/29-01_16-21/results/synthesis/picorv32a.synthesis_cts.v
link_design picorv32a
read_sdc /home/abhinavprakash1999/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/my_base.sdc
report_checks -path_delay min_max -fields {slew trans net cap input_pin}
report_tns
report_wnse`

![Screenshot from 2025-01-31 10-21-13](https://github.com/user-attachments/assets/f6f178ae-59ae-4614-af94-c269c1f25c85)

**Clock Tree Synthesis Stage:**
There are three parameters that we need to consider when building a clock tree:

Clock Skew = In order to have minimum skew between clock endpoints, clock tree is used. This results in equal wirelength (thus equal latency/delay) for every path of the clock.
Clock Slew = Due to wire resistance and capacitance of the clock nets, there will be slew in signal at the clock endpoint where signal is not the same with the original input clock signal anymore. This can be solved by clock buffers. Clock buffer differs in regular cell buffers since clock buffers has equal rise and fall time.
Crosstalk = Clock shielding prevents crosstalk to nearby nets by breaking the coupling capacitance between the victim (clock net) and aggresor (nets near the clock net), the shield might be connected to VDD or ground since those will not switch. Shileding can also be done on critical data nets.


![Screenshot from 2025-01-30 16-07-09](https://github.com/user-attachments/assets/2cead5e0-bb95-4706-a934-b48039e05fab)

**Maze Routing - Lee's Algorithm:**
The shortest path is one that follows a steady increment of one (1-to-9 on the example below). There might be multiple path like this but the best path that the tool will choose is one with less bends. The route should not be diagonal and must not overlap an obstruction such as macros.
This algorithm however has high run time and consume a lot of memory thus more optimized routing algorithm is preferred (but the principles stays the same where route with shortest path and less bends is preferred)


**DRC Cleaning:**
DRC cleaning is the next step after routing. DRC cleaning is done to ensure the routes can be fabricated and printed in silicon faithfully. Most DRC is due to the constraints of the photolitographic machine for chip fabrication where the wavelength of light used is limited. There are thousands of DRC and some DRC are:

Minimum wire width
Minimum wire pitch (center to center spacing)
Minimum wire spacing (edge to edge spacing)
Signal short = this can be solved my moving the route to next layer using vias. This results in more DRC (Via width, Via Spacing, etc.). Higher metal layer must be wider than lower metal layer and this is another DRC.
Routing Stage and TritonRoute:
OpenLane routing stage consists of two stages:

Global Routing - Form routing guides that can route all the nets. The tool used is FastRoute
Detailed Routing - Uses the global routing's guide to actually connect the pins with least amount of wire and bends. The tool used is TritonRoute.
Triton Route

Performs detailed routing and honors the pre-processed route guides (made by global route) and uses MILP based (Mixed Integer Linear Programming algorithm) panel routing scheme(uses panel as the grid guide for routing) with intra-layer parallel routing (routing happens simultaneously in a single layer) and inter-layer sequential layer (routing starts from bottom metal layer to top metal layer sequentially and not simultaneously).
Honors preferred direction of a layer. Metal layer direction is alternating (metal layer direction is specified in the LEF file e.g. met1 Horizontal, met2 Vertical, etc.) to reduce overlapping wires between layer and reduce potential capacitance which can degrade the signal.

![Screenshot from 2025-01-30 21-43-17](https://github.com/user-attachments/assets/4c9ab3fa-176b-4a72-8b1f-fc2f487065a6)

To make power ring and rails `run gen_pdn`

![Screenshot from 2025-02-01 11-31-24](https://github.com/user-attachments/assets/4badd854-23ad-4768-9591-ee553212f178)

**SPEF Extraction and GDSII Streaming:**
To route `run_routing`


![Screenshot from 2025-02-01 15-33-54](https://github.com/user-attachments/assets/ae80605e-fca7-4c97-909a-46f814fd2c60)
