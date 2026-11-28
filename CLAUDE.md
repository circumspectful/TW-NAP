# instructions and description for Claude

**Summary**
This is a mod for Napoleon: Total War focusing on adding interesting units, 
editing some existing mechanics, and overall improving the experience of the game.

**Things to keep in mind**
NTW is an old game with many quirks. For example: 
1. pack files should not be compressed since this will cause crashes.
2. pack files must be registered in user.script.txt to load. Load order matters.
3. edits often must be made across multiple DBs; there is very little done automatically.
4. RPFM MCP: `set_game napoleon`, then `generate_dependencies_cache` once (vanilla isn't loaded until then).
   Vanilla tables are single files named after the table, e.g. `db/units_tables/units`. Don't use "data__".
5. The repo's `Napoleon Total War/` folder is a reference copy of the install and is gitignored. Never commit game files.
6. Real install: C:\Program Files (x86)\Steam\steamapps\common\Napoleon Total War
7. RPFM session can silently fall back to warhammer_3. Check `session_status` shows napoleon right before `new_pack`/`save_pack`.
8. Unit appearance = `uniforms.filename` -> `variantmodels/units/<filename>.{soldier,officer,musician,standard_bearer}.unit_variant`
   (kitbash of existing parts: mesh + texture_folder per category). Cards: `ui/units/icons/<filename>_icon.tga`, `ui/units/info/<filename>_info.tga`.
9. New tables go in NEW, differently named files (e.g. `db/units_tables/tw_nap_units`); they load additively. See FunctionalModExample (AUM-NAP) for proof.
10. user.script.txt must be saved as UTF-16 ("Unicode"), not UTF-8.
11. LOC: loc in a Mod-type pack does NOT load (names blank). Ship a separate Movie-type pack (e.g. mod/tw_nap_loc.pack, copied to data, no script entry)
    containing a FULL text/localisation.loc (vanilla + our rows). Make it by editing the UPC_English.pack (Movie) and saving to a new path.
12. TEXTURES: kitbash parts may only use texture folders listed in the faction's atlas (variantmodels/units/atlas/<faction>_diffuse.atlas, UTF-16 paths).
    New texture folders render nothing. Mesh+texture swaps work only if UV layouts match (e.g. Base_Plume mesh takes French_Plume_2 / White_Plume_1 / French_Plume_4 textures).
    france_spa atlas is smaller than france; check it before reusing a variant for spa_france.
13. Install = copy mod/tw_nap_mod.pack AND mod/tw_nap_loc.pack to the game's data folder; only tw_nap_mod.pack needs a `mod tw_nap_mod.pack;` line in user.script.txt.
14. In-game stats show melee defence = DB value + 3 (DB 11 -> shown 14). Compare units by gaps, not raw numbers.
15. RPFM MCP arg shapes: `source` is a bare string "game_files" or {"pack": key}; table_rows/read_file take `file:{source,path}`; list_files uses `path_prefix`;
    copy_files takes `pack` (destination); add_files_from_disk supports `pack_paths`; new_pack can't set pack type (reuse UPC_English.pack as a Movie base).
    Game files list with lowercase paths. Pillow (pip) can read/write DDS (DXT5) and TGA (use compression=None); rebuild mips if replacing a mipmapped DDS.
16. Unit variants are binary-searchable: extract variantmodels/units from game_files and grep UTF-16 strings (e.g. to find which units use a part).
17. Verified working (2026-11-26): Fusilier-Grenadiers unit (kitbash, stats, costs, loc, recruitment). Unit cards are still placeholder art; real screenshots planned.
18. NEVER commit or push anything in the `temp/` folder. It is for development use only: all references (FunctionalModExample, ImageReferences, the game copy) live there.
    Only commit LICENSE, CLAUDE.md and the packs in mod/ unless told otherwise. (Reference paths in these notes now live under temp/.)
19. NEW TEXTURES (done for Fusilier-Chasseurs): override France's atlas in Movie pack tw_nap_atlas.pack (4096² DXT5; free space at y>=3840).
    Splice 4x4 DXT5 blocks into france_diffuse.dds per mip (levels 0-5 for 128x256), append entries to the .atlas (binary: u32 ver, u32 unk, u32 count, then
    entries of 2x512-byte UTF-16 strings + 6 float32: u,v,w,h normalised plus width,height px; gloss/colour_mask s1 end with `_gloss`/`_colour`).
    THE ENGINE READS EVERY CHANNEL (normal/gloss/mask) AT THE DIFFUSE RECT, so entries in the other .atlas files must mirror the diffuse rect, and france_normal.dds MUST be spliced
    at the same rect (zero normals = dark blotches; this caused the apron splotches). Gloss/colour_mask free area is zero = matte/untinted, fine. Scripts lived in temp/work (atlaslib.py, atlasfile.py, build_atlas.py) - temp is not committed.
    Mesh files: 4 LODs, 64-byte vertices (pos float3 first), see tw_nap_tall_plume. Plume UV: lower ~40% of height uses the bottom texture panel.
20. Atlas free space (france_diffuse, 4096²): only the strip y>=3840 (256 tall), now COMPLETELY FULL (x 0..4096, laid out by temp/work/build_atlas13.py; rects get repacked on each rebuild,
    mips need x%128==0 (128/384-wide) or x%256==0 (256/512-wide)). To add more art, downscale/share something or retire an entry. Colour masks cannot be added (splicing the uncompressed mask DDS = +45MB);
    new rects read a ZERO mask, so BAKE colours into the diffuse (colour x luminance x ~1.5) for new textures.
    Unit variant strings are fixed-width UTF-16 fields: patch them in place by byte replace (see temp/work/patchvar.py), then add_files_from_disk with include_base_folder=true
    (paths must include the `variantmodels/` folder). Deleting wrongly-added paths: delete_files with the stray folder. Clear AppData army_setups/battle_preferences if custom battle crashes after adding units.
21. SLOT TRICK (from NTW3 sappers): any category slot can hold any mesh. The Plumes slot with Torsos/German_Militia_Coat/mesh + a texture that is fully transparent except
    UV rect (0,256)-(136,512) of 512 shows only the coat's front skirt panel = APRON (tw_nap_sapper_apron, stored 256x256 in the atlas; gloss/colour_mask entries point at the black free area x=3072,y=3840).
    Used by Old Guard Sappers. Atlas row y=3840 is filled to x=2048 now. Sapper abilities: chevaux_de_frise, earthworks, wooden_stakes, fougasse_basic (+ stats can_place_stakes=true).
22. MESH FORMAT (variant_part_mesh, skinned parts have header word2==1): header 36 bytes (words: magic, 0, skinned, 0, 4 LODs, total verts, total idx, 9|0, 2|0), then per LOD
    [n, m] + n*40-byte vertices + m u16 indices; trailing bytes after LOD3 must be kept. Vertex: half3 position at +0, half u at +6, half v at +14, (+8..13 second position for blended verts),
    bone indices at +27 and +31, normal-frame bytes at +24..26/+32..39 (128-centred). Non-skinned parts (plumes) use 64-byte vertices.
    Hide geometry by making triangles degenerate (index a,a,a); merge parts from another mesh by appending verts/idx per LOD, fixing both totals (see temp/work/build_meshes11.py, meshlib.py).
    Mirroring a part: negate y halves (+2,+10), flip winding, swap L/R bone indices (14<->17 etc.), flip y of the frame bytes (25,33,37). Axes: x up, z front, +y = wearer's LEFT.
    New mesh files in the Mod pack load fine (tw_nap_tall_plume, tw_nap_gendarme_hat, tw_nap_gendarme_coat). Old Guard Gendarmerie coat = Guard coat + the general's aiguillettes copied UNMIRRORED from G_F_Coat (silver recolour).
    Its texture is an editable 512x512 (temp/coat_edit/gendarmerie_coat_EDIT_512.png) that temp/work/pack_coat12.py repacks into the 512x256 atlas slot: lapel panel 1:1, the rest halved, aiguillettes 1:1; run it, then build_atlas13.py (uses temp/work/dxt5enc.py, our own DXT5 encoder; Pillow's DDS DXT5 encoder is visibly blocky on small details like buttons - always encode with dxt5enc), then re-add both DDS and the coat mesh to the packs.
24. CRASH BISECT RESULT (Napoleon escort soldier variant overriding france_gen_late_napoleon.soldier): the custom buff GLOVE texture (hands/Gloves_2/tw_nap_gendarme_gloves) crashed battle load on the
    general unit although it works on the foot Gendarmes; vanilla Gloves_2/tex works. Cause unknown. Everything else (coat, hat, belt, breeches, plume, silver epaulette) loads. Bisect by adding ONE part per build.
    Old Guard Grenadier visor = the black PEAK island (UV box 160-280 x 80-146 of 384) in the Frontplate mesh, NOT the brass plate: hidden via LOD tris (temp/work/visor13.py -> hats/tw_nap_oldguard_hat/mesh in the Mod pack).
23. NAPOLEON'S ESCORT + FOOT GENDARMERIE: Gen_Late_Napoleon (only; Gen_Early_Napoleon untouched) soldier+musician variants are overridden in the Movie pack tw_nap_atlas.pack
    (france_gen_late_napoleon.*, officer stays vanilla). His stats row is edited in a FULL COPY of vanilla db/unit_stats_land_tables/unit_stats_land inside tw_nap_mod.pack
    (RPFM flags it TableIsDataCoring, expected). AUM never re-declares vanilla keys, so duplicate-key rows in tw_nap_ tables are UNPROVEN - use full-table copies for vanilla rows.
    men 30, dismount (can_dismount + dismounted spacing 0.8/1.75), musket via tw_nap_cavalry_musket_sword. mount_model is per UNIT, so Napoleon and escort share horse_covered_white.
    Foot Old Guard Gendarmerie: 60 men, total_cap/total_cap_mp 1, campmap_policing_bonus (boolean), recruited at sArmy5_staff_college (building_levels.only_in_capital = true => Paris for France).
    The cav test unit was removed (rows, loc and variants renamed into the foot unit). Clear AppData army_setups/battle_preferences if custom battle crashes.
25. UNIT CARDS + CAPS (playtested): cards are RLE TGAs (icon 66x88 RGBA, info 280x300 RGB, same header as vanilla; Pillow compression='tga_rle'), named by the UNIFORM filename
    (e.g. france_inf_skirm_french_young_guard_voltiguers - check the uniforms table, 'skirm' vs 'elite'). New-unit cards live in tw_nap_mod.pack; replacements of vanilla Old Guard / Young Guard cards
    live in the Movie pack tw_nap_atlas.pack. Caps (campaign/MP): Old Guard Grens 2/1, OG Chasseurs 2/1, Sappers 1/1 (120 men), Fusilier-Gren 2/2, Fusilier-Chasseurs 2/2, Voltiguers 4/4, Tirailleurs 4/4, Gendarmes d'Elite 1/1.
    Vanilla Old Guard / Young Guard caps are edited via a FULL COPY of vanilla db/units_tables/units in tw_nap_mod.pack (same data-coring trick as unit_stats_land).
    Display names: "Middle Guard Fusilier-Grenadiers/-Chasseurs", "Gendarmes d'Elite" (loc pack only).
edit this list as you work to add more reminders for yourself.

**General Outline**
1. New Units - primary
   1a. Unit Stats (balancing, unit size, weaponry)
   1b. Unit Textures (recolors of existing uniforms for the most part)
   1c. Unit Models (recombining existing models, like pairing a particular bearskin hat with a particular backpack)
   1d. Unit Flavor (names, descriptions, unit cards)
2. Experimenting With AI Behavior - secondary
3. Experimenting With The Economy - secondary
4. Experimenting With The Campaign - secondary
5. Experimenting With Nations - tertiary
