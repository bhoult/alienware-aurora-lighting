# Alienware Aurora ACT1250 lighting on Linux

A small GTK 4 app and command-line tool for the case lighting on the Alienware Aurora ACT1250 (2025): the front vent ring, the alien-head power button, the rear fan and the CPU cooler ring. It talks to the AW-ELC lighting controller (USB `187c:0550`) directly over hidraw. It needs no AWCC and no OpenRGB, and after a one-time udev rule it needs no root.

- Solid colors, a rainbow, random colors, and an animated color cycle and color wave. The animations run on the controller itself, so they keep going with the app closed.
- Separate colors for the logo, rear fan and CPU cooler, or let them follow the vent.
- Brightness and cycle-speed sliders, with gamma-corrected colors.
- **Keep After Reboot** stores the lighting in the controller's flash as the power-on pattern.
- Updates take about 2 seconds (about 5 for the wave) and never block the window.

## Requirements

- An Aurora ACT1250 whose AW-ELC reports platform `0x0811` and 70 zones. The tool checks this and refuses to run on anything else, because other Alienware machines use different zone maps.
- Python 3 with PyGObject, GTK 4 (4.12 or later) and libadwaita 1.6 or later. On Ubuntu: `sudo apt install python3-gi gir1.2-gtk-4.0 gir1.2-adw-1`.

## Install

```sh
sudo cp 60-alienware-elc.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
install -m 755 alienware-lighting ~/.local/bin/
cp io.github.bhoult.AlienwareLighting.desktop ~/.local/share/applications/
```

The udev rule lets the logged-in user open the controller. Ubuntu's `openrgb` package ships an equivalent rule, so you can skip that step if it's installed. The desktop entry expects `alienware-lighting` on your `PATH`.

## Usage

Open **Alienware Lighting** from the app menu, or run `alienware-lighting` with no arguments.

From the command line:

```sh
alienware-lighting 8000ff                                   # whole case purple
alienware-lighting cycle --speed 3 --save                   # color cycle, kept after reboot
alienware-lighting rainbow --logo ffffff --rear-fan 8000ff --cpu 00ff00
alienware-lighting wave --brightness 60
```

The vent takes a hex color or `rainbow`, `random`, `cycle` or `wave`. `--interior` sets the rear fan and CPU cooler together.

## Zone map

| Zones | Light |
|---|---|
| 0–59 | Front vent ring, ten blocks of six alternating between the two sides |
| 63–65 | Alien head logo on the power button: three LEDs behind one diffuser, so it shows their blend |
| 68 | Rear fan |
| 69 | CPU cooler (AIO) ring |
| 60–62, 66, 67 | Nothing visible |

The GPU's light isn't on this controller.

## What we learned about the controller

Firmware 1.1.10, platform `0x0811`. The command set is the one OpenRGB's AlienwareController uses.

- Every reply just echoes the request. The only success signal is the reply to `FINISH_PLAY` or `FINISH_SAVE` (`03 21 00 03 <slot>`): byte 7, counting the hidraw report id as byte 0, is `0x00` when the program is accepted and `0xFF` when it's rejected.
- A program that names any zone id at or past the controller's zone count (70) is rejected whole, and the lights don't change.
- Color `000000` doesn't switch a zone off: the zone shows an older stored animation instead. Send `010101` for off.
- One select packet, `03 23 01 00 NN <ids>`, takes up to 25 zone ids. This firmware accepted 70 select+action pairs in one program, and 13 pairs with 10 morph actions each, so the Area-51's 13-pair limit doesn't apply here. The tool stays within 13 pairs anyway.
- Morph actions (mode `0x02`, three per `03 24` packet) chained after a selection loop on the controller. A duration of 2000 ms with tempo 100 per step fades smoothly.
- The live slot `0xFFFF` keeps its pattern; we saw no timeout.
- The LEDs use linear PWM, so colors are gamma-corrected (γ = 2.0) before sending, as OpenRGB #5822 suggests.

## Why not OpenRGB?

Ubuntu's OpenRGB 0.9 package crashes on this controller: a stack buffer overflow with more than 25 zones, fixed in OpenRGB 1.0 by [MR !3532](https://gitlab.com/CalcProgrammer1/OpenRGB/-/merge_requests/3532). OpenRGB 1.0 detects it as 70 "Unknown" zones, but each change takes about 10 seconds and freezes the window. It also checks the wrong reply byte, so it treats every update as failed and keeps resending it.

## Credits

- Protocol: OpenRGB's [AlienwareController](https://gitlab.com/CalcProgrammer1/OpenRGB/-/tree/master/Controllers/AlienwareController) by Gabriel Marcano, building on kmescher's reverse engineering in [trackmastersteve/alienfx#41](https://github.com/trackmastersteve/alienfx/issues/41#issuecomment-465388185).
- Program limits, the success byte and gamma: [OpenRGB issue #5822](https://gitlab.com/CalcProgrammer1/OpenRGB/-/work_items/5822), from an Area-51 owner.
- Zone map: [alienfx-tools #631](https://github.com/T-Troll/alienfx-tools/issues/631) and the Aurora R16 map in [alienfx-tools #556](https://github.com/T-Troll/alienfx-tools/issues/556).
- Written with [Claude Code](https://claude.com/claude-code).

## License

GPL-2.0-or-later, matching OpenRGB, whose controller code the protocol handling follows.
