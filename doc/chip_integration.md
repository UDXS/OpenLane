**DISCLAIMER: THIS PAGE IS STILL UNDER DEVELOPMENT.**
**THE INFORMATION HERE MIGHT BE INCORRECT OR OUTDATED.**

**FINALIZATION PENDING IO RELEASE.**

# Chip Integration

Using openlane, you can produce a GDSII from a chip RTL.


## The current Methodology

The current methodology views the chip using the following hierarchy:
- Chip Core
    - The hard macros
    - The rest of the design
- Chip IO
    - IO Pads
    - Power Pads
    - Corener Pads

The current methodology goes as follows:
1. Hardening the hard macros.
2. Hardening the core with the hard macros inside it.
3. Hardening the full chip with the padframe. 


## Hardening The Macros

The macros are hardened normally through the flow by running `./flow.tcl`. 

Make sure to properly configure your macro to get the expected outcome, if it requires special configurations. [Here][0] you can find a list of all the available OpenLANE configuartions.

If your macro requires special steps or skipping/repeating some steps. Then you should use an [interactive script][2].

For power routing purposes your macros will require a special `pdn.tcl` each and will have to meet certain area constraints. Check this [section](#power-routing) for more details.

## Hardening The Core

The chip core would usually require an [interactive script][2] to harden. You could take this [chip core][4] as an example.

You need to set the following environment variables in your `config.tcl` file for the chip core:
- `::env(VERILOG_FILES)` To point at the used verilog files; those that were not previously hardened.
- `::env(VERILOG_FILES_BLACKBOX)` To point at the blackboxes (the hardened macros).
- `::env(EXTRA_LEFS)` To point at the LEF files of the hardened macros.
- `::env(EXTRA_GDS_FILES)` To point at the GDS files of the hardened macros.

Therefore, the verilog files shouldn't have any includes in any of your verilog files. But use `::env(VERILOG_FILES)` and `::env(VERILOG_FILES_BLACKBOX)` for that purpose.

Add `set ::env(SYNTH_READ_BLACKBOX_LIB) 1`, if you have `::env(VERILOG_FILES_BLACKBOX)` in your configuration file.

[Here][0] you can find a list of all the available OpenLANE configuartions.

For power routing purposes your core will require a special `pdn.tcl` each and will have to meet certain constraints. Check this [section](#power-routing) for more details.

## Hardening The Full Chip


The full chip requires an [interactive script][2] to harden. You could take this [full chip][5] as an example.

You need to set the following environment variables in your `config.tcl` file for the chip:
- `::env(VERILOG_FILES)` To point at the used verilog files; those that were not previously hardened. Ideally, this should be only one file.
- `::env(VERILOG_FILES_BLACKBOX)` To point at the blackboxes (the hardened macros). Ideally, this should include all the other verilog files.
- `::env(EXTRA_LEFS)` To point at the LEF files of the hardened core macro.
- `::env(EXTRA_GDS_FILES)` To point at the GDS files of the hardened core macro.

Therefore, the verilog files shouldn't have any includes in any of your verilog files. But use `::env(VERILOG_FILES)` and `::env(VERILOG_FILES_BLACKBOX)` for that purpose.

Add `set ::env(SYNTH_READ_BLACKBOX_LIB) 1`, if you have `::env(VERILOG_FILES_BLACKBOX)` in your configuration file.

Add `set ::env(SYNTH_FLAT_TOP) 1` to your `config.tcl`. To flatten the padframe, if it's presented in a `chip_io` module.

The following inputs are provided to produce the final GDSII:

1. Padframe cfg file (provided by the user or generated by padring). [Here][6] is an example.
2. Hardened lef for the core module, generated [here](#hardening-the-core)
3. Top level netlist instantiating pads and core module (Could be provided by the user or generated by [topModuleGen][7])

**Note:** The chip Verilog file should contain a guard `ifdef PFG` under which all the verilog file names should be included, without specifying a directory. This includes the verilog for the pads as well.

Given these inputs the following [interactive script][5] script. Mainly, it does the following steps:
-  set `padframe_root $::env(TMP_DIR)/padframe/<chip design name>/`.
-  Copy all the lef files (of all the macros and power/corner/io pads) to `$padframe/mag`, and all the verilog files to `$padframe/verilog`.
-  Copy the padframe cfg file to `$padframe/mag`.
-  Combine the pads lef, macros lefs and tech lef in one lef file.
-  Use padring to generate the padframe.def and the core.def. 
-  Parse the padframe cfg file to get the diearea and input it to the floorplanner.
-  Run the top level netlist through yosys.
-  Running `chip_floorplan` which first generates a floorplan using `verilog2def` followed by removing PINS section and cleaning NETS section in the generated def file.
-  Replace the components of the floorplan def file with placed components from padframe def file and core def file.
-  Perform manual placement if desired.
-  legalize the placement.
-  Perform `power_routing`.
-  Route the design.
-  Perform power routing.
-  Generate a GDSII file of the routed design.
-  Run DRC and LVS checks.

## Power_routing

### Macros:
Each macro in your design should have a special `pdn.tcl` and point to it by setting `::env(PDN_CFG)`. You could also use `$PDK_ROOT/sky130A/libs.tech/openlane/common_pdn.tcl` as a reference.

- All `pdn.tcl` files should contain this special stdcell section, instead of the one in the `common_pdn.tcl`. The purpose of this is to prohibit the use of metal 5 in the power grid of the macros and use it exclusively for the core and top levels.

```tcl
pdngen::specify_grid stdcell {
    name grid
    rails {
	    met1 {width $::env(FP_PDN_RAIL_WIDTH) pitch $::env(PLACE_SITE_HEIGHT) offset $::env(FP_PDN_RAIL_OFFSET)}
    }
    straps {
	    met4 {width $::env(FP_PDN_VWIDTH) pitch $::env(FP_PDN_VPITCH) offset $::env(FP_PDN_VOFFSET)}
    }
    connect {{met1 met4}}
}
```

- If your macro contains other macros inside it. Then make sure to add a `macro` section for each or one for all of them depending on their special configs. The following example is using special `connect` section and different `rails` and `straps`:
```tcl
pdngen::specify_grid macro {
    orient {R0 R180 MX MY R90 R270 MXR90 MYR90}
    power_pins "vpwr vpb"
    ground_pins "vgnd vnb"
    straps {
	    met1 {width 0.74 pitch 3.56 offset 0}
	    met4 {width $::env(FP_PDN_VWIDTH) pitch $::env(FP_PDN_VPITCH) offset $::env(FP_PDN_VOFFSET)}
    }
    connect {{met1_PIN_hor met4}}
}
```

**WARNING:** only use met1 and met4 for and rails straps.

Refer to [this][3] for more details about the syntax.

- The height of each macro must be greater than or eaqual to the value of `$::env(FP_PDN_HPITCH)` to allow at least two metal 5 straps on the core level to cross it and all the dropping of a via from met5 to met4 connecting the vertical straps of the macro to the horizontal straps of the core and so connect the power grid of the macro to the outer core ring.

### Core:

The core as well should have a special `pdn.tcl` pointed to by setting `::env(PDN_CFG)`. You could also use `$PDK_ROOT/sky130A/libs.tech/openlane/common_pdn.tcl` as a reference.

It should have an `stdcell` section that includes a `core_ring` on met4 and met5. It should use met5 and met4 for the straps, and met1 for the rails.

```tcl
pdngen::specify_grid stdcell {
    name grid
    core_ring {
	    met4 {width 20 spacing 5 core_offset 20}
	    met5 {width 20 spacing 5 core_offset 20}
    }
    rails {
	    met1 {width $::env(FP_PDN_RAIL_WIDTH) pitch $::env(PLACE_SITE_HEIGHT) offset $::env(FP_PDN_RAIL_OFFSET)}
    }
    straps {
	    met4 {width $::env(FP_PDN_VWIDTH) pitch $::env(FP_PDN_VPITCH) offset $::env(FP_PDN_VOFFSET)}
	    met5 {width $::env(FP_PDN_HWIDTH) pitch $::env(FP_PDN_HPITCH) offset $::env(FP_PDN_HOFFSET)}
    }
    connect {{met1 met4} {met4 met5}}
}
```

Then for each macro it should have a `macro` section to specify that metal4 pins should be hooked up to the metal5 straps. The follwing is an example for one macro section, specified for a macro called `pll`:
```
pdngen::specify_grid macro {
    instance "pll"
    power_pins "vpwr"
    ground_pins "vgnd"
    blockages "li1 met1 met2 met3 met4"
    straps { 
    } 
    connect {{met4_PIN_ver met5}}
}
```

Refer to [this][3] for more details about the syntax.

When you use the `power_routing` command in the chip interactive script, the power pads will be connected to the core ring, and thus the whole chip would be powered.

[0]: ./../configuration/README.md
[1]: ./OpenLANE_commands.md
[2]: ./advanced_readme.md
[3]: https://github.com/The-OpenROAD-Project/OpenROAD/blob/openroad/src/pdngen/doc/PDN.md
[4]: Reference_for_interactive_script_core
[5]: Reference_for_interactive_script_chip
[6]: Example_padframe_cfg_file
[7]: ./../scripts/topModuleGen/README.md