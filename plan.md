# FPGA Tile-Based Game — Project Plan

## 1. Pre-RTL Setup Checklist

### Hardware

- [ ] Digilent Nexys A7-100T (Artix-7 XC7A100T)
- [ ] VGA cable (DE-15 connector)
- [ ] VGA monitor (must support 640×480 @ 60 Hz — virtually all do)
- [ ] Micro-USB cable for JTAG programming (included with board)

### Software and Toolchain

- [ ] **Vivado ML Edition** (free, supports Artix-7). Download from AMD/Xilinx. Install with "Vivado" and "Artix-7" device support only — the full install is 100+ GB, the minimal one is around 30 GB.
- [ ] Verify Vivado can see the board over JTAG: plug in the board, open Vivado Hardware Manager, "Auto Connect", confirm the FPGA shows up.
- [ ] **Digilent board files**: Download from Digilent's GitHub (`vivado-boards`) and install into `<Vivado install>/data/boards/board_files/`. This gives you the Nexys A7 preset in Vivado's project wizard.
- [ ] **Simulation**: Vivado's built-in simulator (xsim) works fine for initial bring-up. For cocotb testbenches later, install cocotb (`pip install cocotb`) and Icarus Verilog or use Verilator. You can also use Vivado xsim as the cocotb backend.
- [ ] **GTKWave** (optional): For viewing VCD waveforms from cocotb sims outside Vivado.
- [ ] **Python 3.9+** for cocotb and any image-to-ROM conversion scripts.
- [ ] **ImageMagick or Pillow**: For converting sprite/tile PNGs to `.mem` or `.coe` files for BRAM initialization.

### Tutorials to Complete Before Writing Game RTL

Work through these in order. Each one gives you a working bitstream you can test on hardware:

- [ ] **Nexys A7 bringboard test**: Load the factory demo bitstream. Verify all buttons, switches, LEDs, seven-segment display, and VGA output work. This confirms your hardware and JTAG are good.
- [ ] **Project F — "Exploring FPGA Graphics"** (projectf.io):
  - [ ] Part 1: Display signals (VGA timing generator, test pattern on screen)
  - [ ] Part 2: Shapes and simple drawing
  - [ ] Part 3: Sprites and block RAM
  - [ ] Part 4+: Framebuffers, scrolling (skim these for architecture ideas)
- [ ] **Write one module from scratch and simulate it in Vivado**: Pick something simple like a counter or a BRAM wrapper. Go through the full flow: write RTL → write a testbench → simulate → view waveforms → synthesize → check utilization. This ensures you're comfortable with the toolchain before the project gets complex.

### Repository Structure

```
fpga-dungeon/
├── README.md
├── constraints/
│   └── nexys_a7.xdc          # Pin assignments and timing constraints
├── rtl/
│   ├── top.sv                 # Top-level wiring only
│   ├── clk_gen.sv             # MMCM wrapper
│   ├── vga_timing.sv          # Sync generation + pixel coordinates
│   ├── tile_renderer.sv       # Tile map lookup + tile ROM read
│   ├── sprite_engine.sv       # Sprite position compare + ROM read
│   ├── pixel_compositor.sv    # Priority mux + blanking
│   ├── game_logic.sv          # FSM for game state updates
│   ├── input_debounce.sv      # Button debouncer with edge detection
│   └── sound_gen.sv           # (stretch) PWM audio
├── mem/
│   ├── tile_map.mem           # Initial tile map data
│   ├── tiles.mem              # Tile pixel data (16×16 tiles)
│   └── sprites.mem            # Sprite pixel data
├── sim/
│   ├── tb_vga_timing.sv       # Vivado testbench for VGA timing
│   ├── tb_tile_renderer.sv
│   ├── tb_sprite_engine.sv
│   ├── tb_game_logic.sv
│   └── cocotb/                # cocotb testbenches (optional)
│       ├── test_vga_timing.py
│       └── test_game_logic.py
├── tools/
│   ├── png_to_mem.py          # Convert tile/sprite PNGs to .mem files
│   └── map_editor.py          # (optional) Simple tile map editor
└── docs/
    ├── block_diagram.png
    └── interface_specs.md     # Signal-level interface definitions
```

### Constraints File Setup

Before writing any RTL, set up `nexys_a7.xdc` with pin assignments for:

- 100 MHz clock input (pin E3)
- VGA output: `vga_r[3:0]`, `vga_g[3:0]`, `vga_b[3:0]`, `vga_hs`, `vga_vs` (12-bit RGB via resistor DAC)
- Five pushbuttons: BTNU, BTND, BTNL, BTNR, BTNC
- (Optional) Pmod header for audio output

The Digilent master XDC file for the Nexys A7 has all pin assignments commented out. Copy it and uncomment just the pins you need. Don't uncomment everything — Vivado will warn on unconnected ports.


## 2. Implementation Order

Each phase ends with something testable, either in simulation or on hardware. Never go more than a day or two without seeing your design work on the actual board.

### Phase 1: VGA timing + test pattern (Week 1)

**Goal**: Colored bars on screen. Proves your clock, timing, constraints, and VGA output all work.

Modules to implement:
- `clk_gen.sv` — Instantiate the MMCM primitive (or use Vivado's Clocking Wizard IP) to generate 25.175 MHz from the 100 MHz board clock. Vivado won't hit 25.175 exactly; 25.0 MHz works fine for most monitors.
- `vga_timing.sv` — 640×480 @ 60 Hz. Outputs: `hsync`, `vsync`, `pixel_x[9:0]`, `pixel_y[9:0]`, `display_enable`, `vblank` (active for one line at start of vertical blanking).
- `top.sv` — Wire clk_gen → vga_timing. Generate RGB test pattern from pixel_x and pixel_y (color bars, gradient, checkerboard — whatever). Output directly to VGA pins.

Testbench: Simulate `vga_timing` and verify the timing matches the VGA spec:
- H visible: 640, H front porch: 16, H sync: 96, H back porch: 48 = 800 total
- V visible: 480, V front porch: 10, V sync: 2, V back porch: 33 = 525 total
- Verify `display_enable` is high only during the visible region.

**This is the most important milestone.** If you have pixels on screen, everything else is incremental.

### Phase 2: Static tile map (Week 2)

**Goal**: A fixed background of colored tiles filling the screen. Proves BRAM initialization and tile lookup work.

Modules to implement:
- `tile_renderer.sv` — The pipeline is: pixel_x/y → compute tile column/row and pixel offset within tile → read tile_map BRAM for tile index → read tile ROM for pixel color → output tile_rgb.
- Tile map BRAM: 40 columns × 30 rows = 1200 entries. Each entry is a tile index (e.g., 8 bits = up to 256 tile types). Initialize from `tile_map.mem`.
- Tile ROM: Stores pixel data for each tile type. For 16×16 tiles with 12-bit color (4-bit per channel matching the Nexys VGA DAC), that's 16 × 16 = 256 pixels per tile. Start with 4-8 tile types.

Key hardware concept: The tile renderer has a **2-stage pipeline** due to BRAM read latency. On cycle N you present the address, on cycle N+1 you get the data. You'll need to account for this 1-cycle delay. One approach: use the pixel coordinates one or two clocks ahead (from the VGA timing counters before they register) to pre-fetch, so the data arrives exactly when you need it. This is real pipelining, and it's exactly what interviewers want to see you reason about.

Start with hand-written `.mem` files (hex values). Get the tile pipeline working before worrying about nice graphics.

### Phase 3: Input handling (Week 2-3)

**Goal**: Read button presses cleanly.

Module to implement:
- `input_debounce.sv` — For each button: sample the raw input into a shift register (e.g., 8 bits wide), clocked at a reduced rate (use a counter to divide the 25 MHz clock down to ~1 kHz sampling). Output is stable when all bits agree. Also generate a single-cycle `edge` pulse on the rising edge of the debounced signal.

This is a small module but important to get right early. Metastability matters here since the buttons are asynchronous inputs — use a 2-FF synchronizer before the debounce logic.

### Phase 4: Game logic + player movement (Week 3)

**Goal**: A player marker (just a different-colored tile for now) that moves around the tile map in response to button presses.

Module to implement:
- `game_logic.sv` — An FSM triggered by `vblank`:
  1. IDLE: Wait for vblank pulse.
  2. READ_INPUT: Latch debounced button edges.
  3. COMPUTE_NEXT: Calculate candidate next position based on button input.
  4. CHECK_COLLISION: Read tile_map_bram at the candidate position (port A). Wait one cycle for BRAM read. Check if the tile at that position is walkable.
  5. UPDATE: If walkable, update player position register. Return to IDLE.

For now, represent the player by writing a special tile index into the tile map at the player's position (and clearing the old position). This is a temporary hack that lets you see movement before the sprite engine exists.

Collision detection is just a BRAM read: present the candidate tile coordinates as an address on port A, wait one cycle, check if the returned tile index is a wall type. No iteration, no sequential scanning. O(1) lookup.

### Phase 5: Sprite engine (Week 4)

**Goal**: The player is now a proper sprite overlaid on the background, not a tile hack.

Module to implement:
- `sprite_engine.sv` — For each pixel clock, compare `pixel_x/y` against each sprite's bounding box (position register ± sprite width/height). If the current pixel falls within a sprite, read the sprite ROM for that pixel's color and transparency bit. Output `sprite_rgb` and `sprite_active`.
- `pixel_compositor.sv` — Priority mux: if `sprite_active` and not transparent, output `sprite_rgb`. Otherwise output `tile_rgb`. Gate everything with `display_enable`. Output black during blanking.

Start with one sprite (the player). The position registers are written by game_logic on vblank.

For multiple sprites (enemies), the simplest approach is to compare against all sprite bounding boxes in parallel (one comparator per sprite). With 4-8 sprites this is cheap in an Artix-7. If sprites overlap, use a fixed priority order.

### Phase 6: Enemies and game state (Week 5)

**Goal**: Enemies that move on their own using FSMs.

Extend `game_logic.sv`:
- Each enemy gets a simple FSM: patrol (move back and forth), chase (move toward player using Manhattan distance comparison), or random walk.
- Enemy updates also happen during vblank, sequenced after the player update.
- Collision between player and enemy: compare positions directly (no BRAM lookup needed since you have the coordinates as registers).
- Add game state: health, score, level transitions. Store these in registers, display on the seven-segment display or as tile overlays.

### Phase 7: Polish and stretch goals (Week 6+)

- **Sound**: PWM audio through a Pmod. A simple tone generator using a counter driving a square wave. Map game events (collision, pickup, movement) to different frequencies.
- **Multiple levels**: Store multiple tile maps in BRAM or load from a larger ROM. Swap the tile map base address when transitioning levels.
- **Scrolling**: Instead of a fixed 40×30 visible map, store a larger map and offset the tile renderer's coordinate origin by a scroll register. This adds complexity to the renderer but is a great portfolio piece.
- **Raycasted first-person view**: Major stretch goal. Requires a division/multiplication pipeline and a different rendering approach entirely. Save this for after the 2D game is solid.


## 3. Module Interface Specifications

### clk_gen

```
Inputs:
  clk_100m     : input  logic          // 100 MHz board oscillator
  rst_n        : input  logic          // Active-low board reset (optional)

Outputs:
  pixel_clk    : output logic          // 25.175 MHz (or 25.0 MHz)
  clk_locked   : output logic          // MMCM lock indicator
```

### vga_timing

```
Inputs:
  pixel_clk    : input  logic
  rst          : input  logic

Outputs:
  hsync        : output logic
  vsync        : output logic
  pixel_x      : output logic [9:0]    // 0-799 (full line), only valid when de=1
  pixel_y      : output logic [9:0]    // 0-524 (full frame), only valid when de=1
  de           : output logic           // Display enable (high in visible region)
  vblank       : output logic           // Single-cycle pulse at start of vertical blank
```

### tile_renderer

```
Inputs:
  pixel_clk    : input  logic
  pixel_x      : input  logic [9:0]
  pixel_y      : input  logic [9:0]
  de           : input  logic

Outputs:
  tile_rgb     : output logic [11:0]   // 4 bits per channel
  tile_valid   : output logic           // High when output is valid (accounts for pipeline delay)

BRAM interfaces (directly instantiated inside this module):
  tile_map_bram port B: read-only, addressed by tile column/row
  tile_rom: read-only, addressed by {tile_index, pixel_row, pixel_col}
```

### sprite_engine

```
Inputs:
  pixel_clk    : input  logic
  pixel_x      : input  logic [9:0]
  pixel_y      : input  logic [9:0]
  sprite_x     : input  logic [9:0] [N_SPRITES]   // X positions from game_logic
  sprite_y     : input  logic [9:0] [N_SPRITES]   // Y positions from game_logic
  sprite_type  : input  logic [3:0] [N_SPRITES]   // Which sprite graphic to use

Outputs:
  sprite_rgb   : output logic [11:0]
  sprite_active: output logic           // High if any non-transparent sprite pixel at this position
```

### pixel_compositor

```
Inputs:
  pixel_clk    : input  logic
  de           : input  logic
  hsync_in     : input  logic           // Passed through (possibly delayed to match pipeline)
  vsync_in     : input  logic
  tile_rgb     : input  logic [11:0]
  sprite_rgb   : input  logic [11:0]
  sprite_active: input  logic

Outputs:
  vga_r        : output logic [3:0]
  vga_g        : output logic [3:0]
  vga_b        : output logic [3:0]
  vga_hs       : output logic
  vga_vs       : output logic
```

### game_logic

```
Inputs:
  pixel_clk    : input  logic
  rst          : input  logic
  vblank       : input  logic           // Triggers one update cycle per frame
  btn_up       : input  logic           // Edge-detected button pulses
  btn_down     : input  logic
  btn_left     : input  logic
  btn_right    : input  logic
  btn_center   : input  logic
  collision_tile: input logic [7:0]     // Tile index read back from BRAM port A

Outputs:
  player_x     : output logic [9:0]
  player_y     : output logic [9:0]
  enemy_x      : output logic [9:0] [N_ENEMIES]
  enemy_y      : output logic [9:0] [N_ENEMIES]
  map_rd_addr  : output logic [10:0]    // Address for collision check on BRAM port A
  map_rd_en    : output logic
  map_wr_addr  : output logic [10:0]    // For item pickups (clear tile)
  map_wr_data  : output logic [7:0]
  map_wr_en    : output logic
```

### input_debounce (per button, instantiated N times)

```
Inputs:
  clk          : input  logic
  btn_raw      : input  logic           // Asynchronous button input

Outputs:
  btn_stable   : output logic           // Debounced level
  btn_edge     : output logic           // Single-cycle pulse on rising edge
```


## 4. Key Design Decisions

**Why 12-bit color?** The Nexys A7 VGA output uses a resistor-ladder DAC with 4 bits per channel. You physically cannot output more than 12-bit color on this board. This actually simplifies your memory math: each pixel in a tile is 12 bits wide.

**Why 16×16 tiles?** Clean power-of-two sizing means your tile column and row calculations are just bit shifts. `tile_col = pixel_x[9:4]` and `tile_row = pixel_y[9:4]`. No division needed.

**Why dual-port BRAM for the tile map?** The renderer must read the tile map every pixel clock to keep up with the VGA scan. The game logic also needs to read (collision checks) and occasionally write (item pickups) the tile map. Dual-port BRAM lets both happen without arbitration. Port B is dedicated to the renderer (read-only, every cycle). Port A is for game logic (read/write, only during vblank). No bus contention, no stalls.

**Why vblank-synced game updates?** Updating game state during the visible scan would cause tearing — the renderer could read a half-updated position and draw the sprite in two places for one frame. By restricting all game state changes to the vblank interval (about 1.3 ms at 60 Hz), you guarantee the renderer always sees a consistent snapshot. This is the same approach used by classic arcade hardware.

**Why parallel sprite comparison, not sequential?** With 4-8 sprites, you can afford one comparator per sprite running simultaneously. Each comparator checks if the current pixel_x/y falls within that sprite's bounding box. This completes in one cycle, keeping the pixel pipeline happy. Sequential scanning would require multiple cycles per pixel and would be a bottleneck.

**Pipeline alignment**: The tile renderer introduces a 2-cycle delay (address calculation + BRAM read). The sprite engine also has a 1-2 cycle delay. The compositor must receive tile_rgb and sprite_rgb aligned to the same pixel. Either pipeline both paths to the same depth, or delay the shorter path with a shift register. Also delay hsync/vsync through the compositor to match. Tracking pipeline delays through the datapath is a core hardware skill.


## 5. Resources

- **Project F tutorials**: https://projectf.io/tutorials/fpga-graphics/ — Best free resource for FPGA graphics fundamentals
- **Nexys A7 reference manual**: https://digilent.com/reference/programmable-logic/nexys-a7/reference-manual
- **Nexys A7 XDC file**: https://github.com/Digilent/digilent-xdc
- **Vivado design suite download**: https://www.xilinx.com/support/download.html
- **VGA timing reference**: http://www.tinyvga.com/vga-timing/640x480@60Hz
- **Nandland FPGA tutorials**: https://nandland.com — Good supplementary material for Verilog basics
