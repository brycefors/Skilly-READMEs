**Ship of Harkinian (SoH)** is a community-developed, reverse-engineered PC port of *The Legend of Zelda: Ocarina of Time*. Unlike emulation, SoH runs natively on modern hardware, enabling widescreen support, high frame rates, modding, and other enhancements.

This guide details the optimal setup process, from installation to configuration.

> **Note:** This guide is based on the **Copper Bravo (9.1.1)** release, on Windows and using an Xbox One Controller.


# Initial Setup
1. Find or dump a supported **Legend of Zelda: Ocarina of Time (OoT)** ROM.
1. Download the latest release of **Ship of Harkinian (SoH)** for your OS from [GitHub](https://github.com/HarbourMasters/Shipwright/releases/latest). 
1. Extract the downloaded ZIP file to a folder.
1. Move the OoT ROM into the extracted SoH folder.
1. Start the `soh` binary (executable).
1. When prompted to generate OTR files, click **Yes**.
1. Select your detected ROM and click **Yes**.
1. Wait for the generation process to complete. If asked to add another ROM, click **No**.
1. Ship of Harkinian will start running Ocarina of Time on your machine.

# In-Game Optimization
1. On the file select screen, navigate to **Options**.
1. Set **Sound** to **Surround**.
1. Change **Z Targeting** to **Hold** to mimic modern Zelda controls.

# Optimal Controller Setup
Ship of Harkinian supports most controllers (Xbox, PlayStation, Switch) out of the box. If you do not have a dual analog controller, skip these steps.

The configuration below binds the buttons to mimic the physical layout of **Ocarina of Time on the GameCube**. Specifically, mapping **N64 B** to **Xbox X** replicates the angle of the GameCube's B button relative to A. It also moves C-buttons to the remaining face buttons to free up the Right Stick for modern camera control.

To configure and remap buttons:

1. Press **Esc** to open the on-screen menu bar.
1. Click on **Settings** in the top bar, then select **Controls**.
1. Ensure your controller is selected in the dropdown menu.
1. **Remap Buttons:** SoH auto-maps controllers, but the following layout is optimized for modern dual-stick gameplay (using an Xbox controller). Use the **X** icon to remove bindings and the **+** icon to add new ones.

    | N64 Button | Xbox Input |
    | :--- | :--- |
    | **A** | A |
    | **B** | X |
    | **Start** | Menu |
    | **L** | LB |
    | **R** | RT |
    | **Z** | LT |
    | **C-Up** | RS (Right Stick Click) |
    | **C-Down** | RB |
    | **C-Left** | Y |
    | **C-Right** | B |

1. **Additional Control Settings:** Scroll down in the Controls window to configure the following:
    *   **Ocarina Controls:** Enable **Dpad Ocarina Playback**.
    *   **Camera Controls:** Enable **Right Stick Aiming**.
    *   **Third-Person Camera:** Enable **Free Look** (allows right-stick camera control).
    *   **Inversion:** (Optional) Disable **Invert Aiming Y Axis** and **Invert Z-Weapon Aiming Y Axis**.
1. **Ready to Play:** Press **Esc** to close the menu and start the game. If you wish to apply further enhancements, proceed to the next section.

# Optimal Graphics
To achieve the best visual experience in Ocarina of Time, configure the following settings:

1. Press **Esc** to open the menu, then navigate to **Settings** > **Graphics**.
1. **High Frame Rate:** Check **Match Refresh Rate**. This unlocks the frame rate from the original 20 FPS to match your monitor's refresh rate.
1. Navigate to the **Enhancements** tab at the top, then select **Graphics**.
1. **Models & Textures:** Enable the following options to improve asset quality:
    *   **Disable LOD** (Level of Detail)
    *   **Enable 3D Dropped Items/Projectiles**
    *   **Animated Link in Pause Menu**
    *   **Show Gauntlets in First-Person**
1. **Draw Distance:** Increase **Actor Draw Distance** to **5x** (Max) and enable **Widescreen Actor Culling**.
1. **Fix Vanishing Paths:** To prevent textures from disappearing at certain camera angles:
    *   Go to **Enhancements** > **Fixes**.
    *   Change **Fix Vanishing Paths** from *Disabled* to **No Vanish**.

# Quality of Life and Convenience
While Ship of Harkinian offers extensive QoL options, some can alter the game's original feel. The recommendations below aim to reduce frustration for new players while preserving the core experience.

1. Press **Esc** to open the menu.
1. Navigate to **Enhancements** > **Quality of Life**.
    *   **Saving:** Enable **Remember Save Location**.
    *   **Item Count Messages:** Enable **Gold Skulltula Tokens**, **Pieces of Heart**, and **Heart Containers**.
    *   **Misc:** Enable **Better Owl**.
    *   **Convenience:** Enable **Instant Putaway**.
1. Navigate to **Enhancements** > **Skips & Speed-ups**.
    *   Enable **Skip Text**, **Empty Bottles Faster** and **Faster Pause Menu**.
    *   Set **Vine/Ladder Climb** to **+1**, **Block Pushing Speed** to **+2**, and **Crawl Speed** to **2x**.
1. Navigate to **Enhancements** > **Items**.
    *   **Equipment:** Enable **Equip Items on Dpad**, **Assignable Tunics and Boots**, and **Ask to Equip New Items**.
    *   **Ocarina:** Enable **Prevent Dropped Ocarina Inputs** and **Fast Ocarina Playback**.
    *   **Masks:** Set **Bunny Hood Effect** to **Faster + Longer Jump**. Enable **Mask Equippable as Adult** and **Persistent Masks**.
    *   **Bow / Slingshot:** Enable **Equip Multiple Arrows at Once**, **Skip Magic Arrow Equip Animation**, and **Aiming Reticle**.
    *   **Hookshot:** Enable **Targetable Hookshot Reticle**.
    *   **Magic Spells:** Enable **Better Farore's Wind** and **Faster Farore's Wind**.
    *   **Bottles:** Enable **Rebottle Blue Fire**.
1. Navigate to **Enhancements** > **Fixes**.
    *   **Camera Fixes:** Enable **Fix Camera Drift**, **Fix Camera Swing**, and **Fix Hanging Ledge Swing Rate**.

# Texture Packs
This section covers recommended texture pack mods to enhance the visual fidelity of Ocarina of Time. Select one or the other depending on your preference as having both could cause graphical glitches.


### OoT Reloaded
**OoT Reloaded** is a comprehensive high-definition texture pack created by GhostlyDark. It aims to modernize the game's visuals while remaining faithful to the original N64 aesthetic.

1.  Visit the [OoT Reloaded website](https://evilgames.eu/texture-packs/oot-reloaded.htm) and download the **OTR** version.
1.  Extract the downloaded `.7z` file.
1.  Move the `.o2r` file into the `mods` folder located inside your Ship of Harkinian directory.
1.  Restart Ship of Harkinian to apply the new textures.

### 3DS Aesthetic
To replicate the visual style of *Ocarina of Time 3D*, follow these steps to install the necessary character models and texture packs.

1.  **Download Character Models:**
    *   Visit [Playas 3DS Adult and Young Link](https://gamebanana.com/mods/download/475743).
    *   Download `3ds_link_-_adult_child.zip` and `ai-upscaled-a_y-link_textures.zip`.
1.  **Download Texture Pack:**
    *   Visit [Djipi's 3DS Experience](https://gamebanana.com/mods/download/477979).
    *   Download `djipi_s_3ds_experience.zip`
1.  **Install:**
    *   Extract the `.otr` files from the downloaded ZIP archives.
    *   Move them into the `mods` folder within your Ship of Harkinian directory.
    *   Delete the following files from the `mods` folder: 
        *  `Djipi's 3DE - 02 Link's Textures (Delete if using a custom player model).otr`
1.  **Restart:** Restart Ship of Harkinian to apply the new textures.
1.  **Grotto and 2D Pre-Rendered Fix:** Press **Esc**, navigate to **Enhancements**, and enable **Disable Grotto Fixed Rotation** and **Disable 2D Pre-Rendered Scenes**.

**Troubleshooting Visual Glitches:**
If you encounter graphical issues with the 3DS setup, follow these steps to ensure correct texture layering:

1.  **Update Texture Pack:** Download [djipi_s_3ds_experience_christmas_2024.zip](https://gamebanana.com/mods/download/477979) and extract it over your existing files.
1.  **Install Base Textures:** Download **OoT Reloaded** (as described in the previous section) and place it in the `mods` folder.
1.  **Verify Mod Load Order:** Launch SoH, open the menu, and navigate to the **Modding** menu. Ensure the load order matches the list below (from top to bottom):
    *   `M3DS*_Textures.otr`
    *   `3ds_adult_link.otr` and `3ds_young_link.otr`
    *   `Djipi's 3DE - *.otr` (Non-numbered files, e.g., HUD Mod)
    *   `Djipi's 3DE - ## *.otr` (Numbered files: Highest numbers at the top, lowest at the bottom)
    *   `OoT_Reloaded_v*.o2r`

# Conclusion
With these settings, you are ready to experience **Ocarina of Time** at its best. Ship of Harkinian breathes new life into this classic with high frame rates, modern controls, and enhanced visuals.

Enjoy your journey through Hyrule!