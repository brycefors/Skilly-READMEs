**Zelda 64: Recompiled** is a high-performance, native PC port of *The Legend of Zelda: Majora's Mask* (with *Ocarina of Time* support in development). Unlike traditional emulation, which mimics original hardware in real-time, this project uses static recompilation to transform the original N64 machine code into native C code that runs directly on modern processors.

This guide details the optimal setup process, from installation to configuration.

> **Note:** This guide is based on the **v1.2.2** release, on Windows and using an Xbox One Controller.

# Initial Setup
1. Find or dump a supported **Legend of Zelda: Majora's Mask (MM)** ROM.
1. Download the latest release of **Zelda 64: Recompiled** for your OS from [GitHub](https://github.com/Zelda64Recomp/Zelda64Recomp/releases/latest).
1. Extract the downloaded ZIP file to a folder.
1. Launch **Zelda64Recompiled.exe**.
1. Click **Select ROM**, locate your MM ROM file, and the game will launch.

# Optimal Controller Setup
Zelda 64: Recompiled supports most controllers (Xbox, PlayStation, Switch) out of the box. If you do not have a dual analog controller, skip these steps.

The configuration below binds the buttons to mimic the physical layout of **Ocarina of Time on the GameCube**, optimized for modern dual-stick gameplay.

To configure controls:
1.  Launch **Zelda64Recompiled.exe** (or restart it if currently running).
1.  Click **Setup controls** in the launcher window.

### Button Mapping
Ensure your controller is detected. The recommended mapping for an Xbox controller is:

| N64 Button | Xbox Input |
| :--- | :--- |
| **A** (Action) | A |
| **B** (Attack/Cancel) | X |
| **Start** | Menu |
| **L** (Toggle Map) | LB |
| **R** (Shield) | RT |
| **Z** (Target) | LT |
| **C-Up** (Look/Fairy) | RS (Right Stick Click) |
| **C-Left** (Item 1) | Y |
| **C-Down** (Item 2) | RB |
| **C-Right** (Item 3) | B |

### Camera & Gameplay Settings
Navigate to the **General** tab within the controls menu to enable modern camera controls:

1.  **Targeting Mode:** Set to **Hold**.
1.  **Aiming Camera Mode:** Set to **None**.
1.  **Analog Camera:** Set to **On** (Enables Free Look).
1.  **Analog Camera Mode:** Set to **None**.

Once finished, close the controls window and click **Start Game**.

# Recommended Mods
Zelda 64: Recompiled supports a variety of mods that enhance visuals, fix bugs, and improve quality of life. The following list covers essential mods to give the game a remastered feel.

### How to Install Mods
1.  Launch **Zelda64Recompiled.exe**.
1.  On the splash screen, click **Mods** (or press **Esc** in-game and navigate to **Mods**).
1.  Click **Install Mod** in the bottom-left corner.
1.  Browse to and select the downloaded ZIP file for the mod.
1.  The mod will appear in the list. Ensure the checkbox is enabled.
1.  Restart the game to apply changes.

### Dependencies
Some mods require these core libraries to function. Install these first:
* [ObjDepLoader](https://thunderstore.io/c/zelda-64-recompiled/p/ProxyMM/ObjDepLoader/)
* [EZ Text Replacer API](https://thunderstore.io/c/zelda-64-recompiled/p/LT_Schmiddy/EZ_Text_Replacer_API/)
* [Message Hooks](https://thunderstore.io/c/zelda-64-recompiled/p/danielryb/Message_Hooks/)

### Graphics & Visuals
* [MMN64HD](https://thunderstore.io/c/zelda-64-recompiled/p/Nerrel/MMN64HD/) - HD Textures for Majora's Mask Recompiled.
* [Disable Draw Distance](https://thunderstore.io/c/zelda-64-recompiled/p/tomtee/Disable_Draw_Distance/) - Removes fog and draw distance limits.
* [Seam Fixer](https://thunderstore.io/c/zelda-64-recompiled/p/Reonu/Seam_Fixer/) - Fixes visible seams in world geometry.
* [3DItems](https://thunderstore.io/c/zelda-64-recompiled/p/ProxyMM/3DItems/) - Replaces 2D item sprites with 3D models.

### Quality of Life & Fixes
* [Camera Fixes](https://thunderstore.io/c/zelda-64-recompiled/p/danielryb/Camera_Fixes/)
* [Epona Control Overhaul](https://thunderstore.io/c/zelda-64-recompiled/p/LT_Schmiddy/Epona_Control_Overhaul/)
* [Bow Aiming Reticle](https://thunderstore.io/c/zelda-64-recompiled/p/LT_Schmiddy/Bow_Aiming_Reticle/)
* [Easier Masks](https://thunderstore.io/c/zelda-64-recompiled/p/Hyped/Easier_Masks/)
* [Owls Never Quit](https://thunderstore.io/c/zelda-64-recompiled/p/RecompRando/Owls_Never_Quit/)
* [Move While Aiming](https://thunderstore.io/c/zelda-64-recompiled/p/LT_Schmiddy/Move_While_Aiming/)
* [Permanent Razor Sword](https://thunderstore.io/c/zelda-64-recompiled/p/LT_Schmiddy/Permanent_Razor_Sword/)
* [Song Utilities](https://thunderstore.io/c/zelda-64-recompiled/p/danielryb/Song_Utilities/)
* [Forms Use More Items](https://thunderstore.io/c/zelda-64-recompiled/p/LT_Schmiddy/Forms_Use_More_Items/)
* [Fast Climb](https://thunderstore.io/c/zelda-64-recompiled/p/Keanine/Fast_Climb/)
* [Fast Pushing](https://thunderstore.io/c/zelda-64-recompiled/p/Keanine/Fast_Pushing/)
* [Quick Putaway](https://thunderstore.io/c/zelda-64-recompiled/p/ItsHeckinPat/Quick_Putaway/)

# Conclusion
With these settings and mods, you are ready to experience **Majora's Mask** with modern performance and conveniences.

Enjoy saving Termina!