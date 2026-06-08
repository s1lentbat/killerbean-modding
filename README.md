# Killer Bean — Game Code Reference
### Decompiled from `Assembly-CSharp.dll` via MelonLoader IL2CPP Interop + dnSpy

This document describes every major system, class, and field found in the game's decompiled source code. All classes live in the `Il2Cpp` namespace inside `Assembly-CSharp.dll`. This is a reference for mod development — use it to know what exists, what it does, and how to access it from a MelonLoader plugin.

---

## Table of Contents

1. [How the Code Works (IL2CPP Interop)](#1-how-the-code-works-il2cpp-interop)
2. [Player Stats System — `PlayerStats`](#2-player-stats-system--playerstats)
3. [Player Input — `Player_Input_Handler`](#3-player-input--player_input_handler)
4. [Player Shooting — `Player_Shooting_Control`](#4-player-shooting--player_shooting_control)
5. [Player Animation — `KB_Animancer`](#5-player-animation--kb_animancer)
6. [Player Movement — `KB_FirstPerson_Animancer`, `FPController`](#6-player-movement)
7. [Enemy AI — `Enemy_Simple_AI`](#7-enemy-ai--enemy_simple_ai)
8. [Enemy Health & Damage — `Enemy_Damage_Receiver`, `EnemyHealth`, `EnemyStats`](#8-enemy-health--damage)
9. [Specialized Enemy Types](#9-specialized-enemy-types)
10. [Combat & Damage System — `Damage`, `BulletsDamage`, `Explosion_Damage`](#10-combat--damage-system)
11. [Weapon System](#11-weapon-system)
12. [Currency System — `CurrencyManager`](#12-currency-system--currencymanager)
13. [Mission System — `Mission_Manager`, `Mission_MASTER`](#13-mission-system)
14. [Campaign System — `Campaign`, `Campaign_Director`](#14-campaign-system)
15. [World & Environment — `Game_Master_Variables`, `DayNight`](#15-world--environment)
16. [Health Manager — `HealthManager`, `SimpleHealth`](#16-health-manager)
17. [Audio — `AudioManager`, `RandomAudioPlayer`](#17-audio)
18. [UI Systems](#18-ui-systems)
19. [Save System — `KB_Save_Manager`](#19-save-system--kb_save_manager)
20. [Spawning System](#20-spawning-system)
21. [Vehicle System](#21-vehicle-system)
22. [Boss Enemies](#22-boss-enemies)
23. [Special Beans](#23-special-beans)
24. [Interaction System — `KB_Interactor`](#24-interaction-system--kb_interactor)
25. [Skills System — `Player_Skills`, `Skill`](#25-skills-system)
26. [Shop System — `ShopManager`](#26-shop-system--shopmanager)
27. [Wave System — `WaveManager`](#27-wave-system--wavemanager)
28. [Object & Transform Utilities](#28-object--transform-utilities)
29. [Third-Party Middleware Used](#29-third-party-middleware-used)
30. [Quick Mod Reference — Most Useful Fields & Methods](#30-quick-mod-reference)

---

## 1. How the Code Works (IL2CPP Interop)

Killer Bean is compiled with Unity's **IL2CPP** backend. This means the original C# game code is compiled to native machine code (`GameAssembly.dll`). You cannot run the original C# directly.

When MelonLoader launches, it runs a tool called **IL2CppInterop** (formerly Il2CppAssemblyUnhollower) which reads the metadata file at `Killer Bean_Data/il2cpp_data/Metadata/global-metadata.dat` and generates **interop wrapper DLLs** — one for each original assembly — stored in `MelonLoader/Il2CppAssemblies/`.

These wrapper DLLs contain C# class/method stubs that call the native code via raw `IL2CPP.*` pointer operations. This is what you see in every decompiled file:

```csharp
IL2CPP.il2cpp_runtime_invoke(SomeClass.NativeMethodInfoPtr_SomeMethod, ...);
```

When you write mod code that calls `PlayerStats.Instance.Heal(100f)`, the interop wrapper translates that into the correct native function call inside `GameAssembly.dll`.

**Key implication for modders:** You cannot patch game methods by replacing managed DLLs. You must use **Harmony** (included with MelonLoader) to hook native method invocations at runtime, or directly read/write fields through the interop properties.

---

## 2. Player Stats System — `PlayerStats`

**File:** `PlayerStats.cs`  
**Namespace:** `Il2Cpp`  
**Type:** `MonoBehaviour`, singleton

This is the single most important class for gameplay modding. It holds every numerical stat for the player character and is accessible globally via `PlayerStats.Instance`.

### Singleton Access
```csharp
PlayerStats ps = PlayerStats.Instance; // null if not in a gameplay scene
```

### Health Fields
| Field | Type | Description |
|-------|------|-------------|
| `baseMaxHealth` | `float` | The base maximum health value set at game start |
| `maxHealthBonus` | `float` | Additive bonus on top of base max health from upgrades |
| `currentHealth` | `float` | The player's current health. Write directly to set it |
| `currentArmor` | `float` | Current armor value. Armor absorbs damage before health |
| `extraLives` | `int` | Number of revival chances remaining |

### Combat Fields
| Field | Type | Description |
|-------|------|-------------|
| `baseDamageMultiplier` | `float` | Multiplies all outgoing damage. Default ~1.0 |
| `weaponDamageBonus` | `float` | Additive weapon damage bonus from upgrades |
| `criticalChance` | `float` | Probability (0.0–1.0) of landing a critical hit |
| `criticalMultiplier` | `float` | Damage multiplier when a critical hit occurs |
| `damageReduction` | `float` | Fraction of incoming damage absorbed/ignored |

### Weapon / Combat Rate Fields
| Field | Type | Description |
|-------|------|-------------|
| `fireRateMultiplier` | `float` | Multiplies fire rate. 2.0 = twice as fast |
| `magazineSizeMultiplier` | `float` | Multiplies magazine capacity |
| `reloadSpeedMultiplier` | `float` | Multiplies reload speed. Higher = faster reload |

### Movement Fields
| Field | Type | Description |
|-------|------|-------------|
| `baseMoveSpeed` | `float` | Base movement speed value |
| `moveSpeedMultiplier` | `float` | Multiplier applied on top of base speed |

### Unlock Flags
| Field | Type | Description |
|-------|------|-------------|
| `hasDodgeRoll` | `bool` | Whether the player can perform dodge rolls |
| `hasDoubleJump` | `bool` | Whether the player has double jump ability |
| `hasGrapplingHook` | `bool` | Whether the player has grappling hook ability |

### Economy Fields
| Field | Type | Description |
|-------|------|-------------|
| `currencyMultiplier` | `float` | Multiplies all currency pickups |

### Events (subscribe in mod code)
| Event | Signature | Fires when |
|-------|-----------|------------|
| `OnHealthChanged` | `Action<float, float>` | Health changes (args: newHealth, maxHealth) |
| `OnDeath` | `Action` | Player dies |
| `OnRevive` | `Action` | Player revives from an extra life |

### Methods
| Method | Description |
|--------|-------------|
| `GetMaxHealth()` | Returns `baseMaxHealth + maxHealthBonus` |
| `GetCurrentHealth()` | Returns `currentHealth` |
| `Heal(float amount)` | Adds health up to max. Fires `OnHealthChanged` |
| `TakeDamage(float damage)` | Applies damage after reduction/armor calc. Fires `OnHealthChanged`, may trigger death |
| `AddArmor(float amount)` | Adds armor |
| `Die()` | Kills the player immediately |
| `Revive()` | Revives the player (uses an extra life) |
| `GrantExtraLife()` | Adds one extra life |
| `IncreaseMaxHealth(float amount)` | Permanently raises max health |
| `AddDamageMultiplier(float percent)` | Adds to damage multiplier |
| `AddWeaponDamage(float percent)` | Adds to weapon damage bonus |
| `AddCriticalChance(float percent)` | Raises crit chance |
| `AddFireRate(float percent)` | Raises fire rate multiplier |
| `AddMagazineSize(float percent)` | Raises magazine size multiplier |
| `AddReloadSpeed(float percent)` | Raises reload speed |
| `AddMoveSpeedMultiplier(float percent)` | Raises move speed multiplier |
| `GetMoveSpeed()` | Returns computed move speed |
| `CalculateDamage(float baseDamage)` | Returns final damage after all multipliers and crit rolls |
| `RollCritical()` | Returns `true` if a random crit roll succeeds |
| `UnlockDodgeRoll()` | Sets `hasDodgeRoll = true` |
| `UnlockDoubleJump()` | Sets `hasDoubleJump = true` |
| `UnlockGrapplingHook()` | Sets `hasGrapplingHook = true` |
| `ApplyCurrencyMultiplier(int base)` | Returns currency after applying multiplier |
| `GetStatsDebugString()` | Returns a string dump of all stats (useful for logging) |

---

## 3. Player Input — `Player_Input_Handler`

**File:** `Player_Input_Handler.cs`  
**Namespace:** `Il2Cpp`  
**Type:** `MonoBehaviour`

Central hub for all player input state. Almost every other player system reads from this. Uses Unity's new Input System (`UnityEngine.InputSystem`) but also has legacy Rewired support.

### Controller Detection
| Field | Type | Description |
|-------|------|-------------|
| `current_controller` | (enum) | Which input device is active (keyboard, gamepad, etc.) |
| `latest_controller_string` | `string` | Name of the last used controller |
| `current_controller_string` | `string` | Name of the current controller |
| `can_change_controller` | `bool` | Whether the game will accept a new controller swap |

### Core Input Flags
| Field | Type | Description |
|-------|------|-------------|
| `can_read_inputs` | `bool` | Master switch — if false, all inputs are ignored |
| `can_move` | `bool` | Whether player movement is permitted |
| `can_mouse_gamepad_look` | `bool` | Whether camera look input is accepted |
| `can_fire` | `bool` | Whether the player is allowed to fire |
| `fire_button_pressed` | `bool` | True while fire input is held |

### Player State Flags
| Field | Type | Description |
|-------|------|-------------|
| `is_dead` | `bool` | Player death state |
| `is_in_neardeath_mode` | `bool` | Low health "near death" dramatic state |
| `is_in_FPS_Mode` | `bool` | True when camera is in first-person mode |
| `is_currently_ADS` | `bool` | True while aiming down sights |
| `ADS_zoomed_in` | `bool` | True once ADS zoom is fully complete |
| `is_jumping` | `bool` | Currently airborne from a jump |
| `is_sprinting` | `bool` | Sprint input held |
| `is_crouched` | `bool` | Currently crouching |
| `is_Dive` | `bool` | Currently performing a dive / roll |
| `is_Slide` | `bool` | Currently sliding |
| `is_reloading` | `bool` | Reload animation playing |
| `is_knocked_out` | `bool` | Knocked out / temporarily incapacitated |
| `is_special_move` | `bool` | Special slow-motion move active |
| `is_slowmo_on` | `bool` | Time slow is active |
| `is_special_slowmo` | `bool` | Cinematic special slow-mo variant |

### Vehicle State
| Field | Type | Description |
|-------|------|-------------|
| `getting_in_vehicle` | `bool` | Transition animation to enter vehicle |
| `is_driving_vehicle` | `bool` | True while inside a vehicle |
| `unlimited_ammo_while_driving` | `bool` | Ammo consumption disabled in vehicle |
| `can_exit_vehicle` | `bool` | Whether the exit prompt is available |

### Combat Input Queues
| Field | Type | Description |
|-------|------|-------------|
| `queue_melee_light` | `bool` | Light melee attack buffered |
| `queue_melee_heavy` | `bool` | Heavy melee attack buffered |
| `melee_light_button_clicked` | `bool` | Melee button just pressed this frame |
| `is_jump_attack` | `bool` | Aerial attack combo active |
| `can_upward_attack` | `bool` | Upward strike available |

### Water State
| Field | Type | Description |
|-------|------|-------------|
| `is_under_water` | `bool` | Player is submerged |
| `water_depth_height` | `float` | Depth of current water body |
| `water_swim_height` | `float` | Y threshold at which swimming begins |

### Bullet Trick System
A unique Killer Bean mechanic: the player can perform stylish multi-shot "bullet tricks."

| Field | Type | Description |
|-------|------|-------------|
| `is_shoot_bullet_trick` | `bool` | Bullet trick sequence active |
| `shoot_bullet_trick_from_3rdperson` | `bool` | Trick was initiated in third-person mode |
| `bullet_trick_shot_count` | `int` | Number of shots fired in the current trick |
| `bullet_trick_start_timestamp` | `float` | `Time.time` when trick began |
| `bullet_trick_end_timestamp` | `float` | `Time.time` when trick ended |

### Weapon Swap State
| Field | Type | Description |
|-------|------|-------------|
| `is_swapping_weapon` | `bool` | Weapon swap animation playing |
| `holstering_guns_playing` | `bool` | Holster animation playing |
| `all_weapons_holstered` | `bool` | Both weapon slots are holstered |

### Primary Action Priority
| Field | Type | Description |
|-------|------|-------------|
| `current_primary_action` | (enum) | Current queued primary action (shoot, melee) |
| `next_primary_action` | (enum) | Next queued primary action |
| `primary_action_waittime` | `float` | Cooldown between switching primary actions |
| `time_stamp_shoot` | `float` | `Time.time` of last shot |
| `time_stamp_melee` | `float` | `Time.time` of last melee |

---

## 4. Player Shooting — `Player_Shooting_Control`

**File:** `Player_Shooting_Control.cs`  
**Namespace:** `Il2Cpp`  
**Type:** `MonoBehaviour`

Bridges input to the actual weapon firing logic. Connects to the Infima Games weapon pack and the F3D projectile system.

### References Held
| Field | Type | Description |
|-------|------|-------------|
| `_actions` | `InputActionAsset` | The Unity Input System action map asset |
| `health_manager` | `HealthManager` | Reference to player health component |
| `KBAnim` | `KB_Animancer` | Third-person animation controller |
| `KBAnim_FPS` | `KB_FirstPerson_Animancer` | First-person animation controller |
| `KB_Char_ECM2` | (ECM2 character) | Physics-based character controller |
| `PIH` | `Player_Input_Handler` | Input state reference |
| `char_gun` | (InfimaGames weapon) | Currently equipped weapon component |
| `F3D` | `F3DProjectile` | Projectile spawning system |
| `shoot_from_emulator` | (Transform/object) | Origin point for emulated shots |
| `cam_dir` | (Transform) | Camera forward direction for aiming |

### Methods
| Method | Description |
|--------|-------------|
| `IsFireButtonHeld()` | Returns true if fire input is currently held |
| `Fire()` | Triggers a shot — called when fire conditions are met |
| `FireStop()` | Stops automatic fire |
| `Network_FireStop()` | Network-synced fire stop (multiplayer) |
| `OnFire(CallbackContext)` | Input System callback on fire press |
| `Rewired_Fire()` | Legacy Rewired input fire trigger |
| `Rewired_StopFiring()` | Legacy Rewired input fire release |

---

## 5. Player Animation — `KB_Animancer`

**File:** `KB_Animancer.cs`  
**Namespace:** `Il2Cpp`  
**Type:** `MonoBehaviour`

The animation state machine for the player character. Uses the **Animancer** middleware (a Unity animation system). Tracks animation state transitions for all player actions.

### Key State Fields
| Field | Type | Description |
|-------|------|-------------|
| `this_character_mode` | (enum) | Whether character is in third-person, FPS, vehicle, etc. |
| `current_animation` | (AnimationClip/state) | Currently playing animation |
| `prev_animation` | (AnimationClip/state) | Previously played animation |
| `is_dead` | `bool` | Death animation playing |
| `can_interrupt_anim` | `bool` | Whether the current anim can be cut short |

### Jump / Air Animation
| Field | Type | Description |
|-------|------|-------------|
| `jump_stage` | (enum) | Jump phase: takeoff, ascending, peak, descending, landing |
| `jump_stage_previous` | (enum) | Previous jump phase for transition logic |
| `current_jump_count` | `int` | How many jumps used (tracks double jump) |
| `y_velocity` | `float` | Vertical velocity, used to pick fall animations |
| `will_land_timestamp` | `float` | Predicted landing time for anticipation anims |

### Aim Animation
| Field | Type | Description |
|-------|------|-------------|
| `anim_aim_stage` | (enum) | Blend stage of aim-in / aim-out animation |
| `angle_of_aiming_H` | `float` | Horizontal aim angle, drives upper body blend |
| `time_wo_shooting_reset` | `float` | Duration after which aim resets if not firing |
| `time_wo_shooting_timer` | `float` | Timer tracking idle-since-last-shot |

### Movement Direction Vectors
| Field | Type | Description |
|-------|------|-------------|
| `moveinput_direction` | `Vector3` | Raw move input direction |
| `dir_2D_cam_main` | `Vector2` | Camera facing in 2D (XZ plane) |
| `dir_2D_player` | `Vector2` | Player facing in 2D |
| `dir_2D_vehicle` | `Vector2` | Vehicle heading in 2D |

### Vehicle / Special Move Flags
| Field | Type | Description |
|-------|------|-------------|
| `is_driving_motocross` | `bool` | On a motocross bike |
| `is_car_backflip` | `bool` | Car backflip trick active |
| `is_launched` | `bool` | Player was launched by explosion or launcher |
| `launch_speed` | `float` | Speed of launch for physics blend |
| `enemy_lockon_activated` | `bool` | Enemy lock-on aiming mode active |

---

## 6. Player Movement

### `FPController` / `KB_Butt_Ground_Check` / `KB_Wall_Run_Detector`

The player uses ECM2 (Easy Character Motion 2) as the base physics character controller. On top of that:

- **`KB_Wall_Run_Detector`** — Raycasts to detect walls, enables wall-running state when the player runs parallel to a wall.
- **`KB_Wall_Run_Raycaster`** — Helper that fires the actual wall-detection rays.
- **`KB_Butt_Ground_Check`** — Checks if the player's lower body (butt) is touching the ground, used for landing and slide initiation.
- **`KB_Dive_Land_Checker`** — Checks for safe landing when performing a dive.
- **`KB_Obstacle_Detector`** — Detects obstacles in the player's path for vaulting/climbing logic.

---

## 7. Enemy AI — `Enemy_Simple_AI`

**File:** `Enemy_Simple_AI.cs`  
**Namespace:** `Il2Cpp`  
**Type:** `MonoBehaviour`

The standard enemy brain. Uses Unity's NavMesh for pathfinding and Animancer for animations. Enemies with this component follow the player, play search animations when they lose sight, and attack when in range.

### Navigation
| Field | Type | Description |
|-------|------|-------------|
| `agent` | `NavMeshAgent` | Unity pathfinding component |
| `agent_speed` | `float` | Current navmesh movement speed |
| `target_destination` | `Vector3` | Current navmesh destination |
| `dist_to_target` | `float` | Cached distance to current target |
| `target_radius` | `float` | How close to target before stopping |
| `_navMeshHit` | `NavMeshHit` | Result of last navmesh position sample |

### State Flags
| Field | Type | Description |
|-------|------|-------------|
| `can_attack` | `bool` | Attack is off cooldown |
| `is_attacking` | `bool` | Currently playing attack animation |
| `got_hit` | `bool` | Was hit this frame |
| `can_find_new_target_pos` | `bool` | Allowed to pick a new wander point |

### Animation
| Field | Type | Description |
|-------|------|-------------|
| `_Animancer` | `AnimancerComponent` | Body animation controller |
| `_FaceAnimancer` | `AnimancerComponent` | Face/expression animation controller |
| `idle_anims` | `AnimationClip[]` | Pool of idle animations (randomly selected) |
| `run_anims` | `AnimationClip[]` | Pool of run animations |
| `idle_search_anim` | `AnimationClip` | "Looking around" animation when target is lost |
| `attack_anim` | `AnimationClip` | Melee attack animation |
| `stagger_anim` | `AnimationClip` | Hit-stagger animation |
| `stunned_anim` | `AnimationClip` | Stunned/dazed animation |
| `suplexed_anim` | `AnimationClip` | Being suplexed by the player |

### Cooldown Timers
| Field | Type | Description |
|-------|------|-------------|
| `anim_change_cooldown` | `float` | Current timer before next anim swap |
| `anim_change_cooldown_max` | `float` | Max cooldown between anim changes |
| `got_hit_cooldown` | `float` | Timer locking hit reaction anim |
| `target_change_cooldown` | `float` | Timer before picking a new wander position |
| `search_anim_cooldown` | `float` | How long since the search anim last played |

### Methods
| Method | Description |
|--------|-------------|
| `Got_Hit()` | Triggers hit stagger animation |
| `Get_Suplexed()` | Triggers suplex animation |
| `Attack_End()` | Called by animation event when attack finishes |
| `Stagger_End()` | Called by animation event when stagger finishes |

### Related: `Enemy_PandaBT_Ticker`
More complex enemies use **PandaBT** (a Behavior Tree middleware) instead of `Enemy_Simple_AI`. The ticker drives the BT at a configurable update rate, controlled by `Game_Master_Variables.Panda_BT_acitvation_dist` — enemies outside this range have their BT paused for performance.

---

## 8. Enemy Health & Damage

### `Enemy_Damage_Receiver`
**File:** `Enemy_Damage_Receiver.cs`  
The collider-facing component placed on enemy hitboxes. When a bullet or explosion overlaps with this, damage is routed through it to the master reference.

| Field | Type | Description |
|-------|------|-------------|
| `Ref` | `Enemies_Master_Ref` | Points to the enemy's central data/health hub |

### `Enemies_Master_Ref`
Central component on the enemy root GameObject. Links all sub-components (AI, health, animations, weapons, voice) together so they can communicate without expensive `GetComponent` calls at runtime.

### `EnemyHealth`
Tracks hit points for a standard enemy. Has a reference to `EnemyStats` for base values.

### `EnemyStats`
Holds the enemy's stat data: base health, damage output, movement speed, etc. Think of it as the enemy equivalent of `PlayerStats` but without a singleton — each enemy has its own instance.

### `Enemy_Vehicle_Health`
Variant health component for enemy vehicles (cars, helicopters, tanks). Tracks vehicle structural health separately from passenger health.

### `Enemy_Weak_Spot`
Component placed on specific collider zones that receive multiplied damage. Used for headshots, exposed fuel tanks, etc.

---

## 9. Specialized Enemy Types

| Class | Description |
|-------|-------------|
| `Shadow_Bean_AI` / `ShadowBeanAI` | Shadow Bean: fast melee enemy, uses `ShadowBeanMelee` and `ShadowBeanProjectileSystem` |
| `Ninja_AI_optimized` / `NinjaAIController` | Ninja: wall-climbing, fast, evasive melee |
| `Enemy_Melee_AI` | Generic melee grunt |
| `Enemy_Rusher_AI` | Charge-rushes the player at high speed |
| `Enemy_Melee_Thrower` | Throws objects/grenades at the player |
| `Enemey_Sniper_Simple_AI` | Long-range stationary sniper |
| `EnemyTurretController` | Fixed gun turret AI |
| `EnemyMortarController` | Mortar lobbing enemy |
| `SentryGun_AI` | Auto-sentry gun that tracks movement |
| `DroneAI` / `DroneAIController` | Flying drone enemy |
| `Helicopter_AI_NavAgent` | Enemy helicopter with patrol routes |
| `Enemy_Vehicle_AI_NavAgent` | Ground vehicle that uses NavMesh |
| `Enemy_Vehicle_AI_Omni` | Omnidirectional vehicle AI (hovertanks, etc.) |
| `Robot_Titan_AI` | Large mech enemy |
| `MechSentryController` | Stationary mech sentry |
| `Giant_Mech_script` | Boss-scale giant mech |
| `Necro_Bean_AI` | Undead bean enemy type |
| `WarlordBean` | Warlord bean with custom attack pattern |
| `HammerBean` | Bean with a giant hammer weapon |
| `HeavyBean` | Armored heavy bean enemy |
| `SphericalDroneBoss` | Sphere drone boss enemy |
| `MiniTankAI` | Small tank enemy |

---

## 10. Combat & Damage System

### `Damage`
**File:** `Damage.cs`  
Base damage data container. Holds a damage value, damage type, and source reference. Passed from weapons/explosions to health components.

| Field | Type | Description |
|-------|------|-------------|
| (damage value) | `float` | Amount of damage to apply |
| (damage type) | `Damage_Type` | Enum: bullet, explosion, melee, fire, etc. |

### `Damage_Type`
Enum defining damage categories. Used for resistances and effects:
- `Bullet`
- `Explosion`  
- `Melee`
- `Fire`
- (potentially others)

### `BulletsDamage`
Component on bullet prefabs. Carries damage value and applies it on collision with `Enemy_Damage_Receiver` or `IHealthDamage`.

### `Explosion_Damage`
Applied in a radius around an explosion origin. Uses `Physics.OverlapSphere` to find all targets. Falls off with distance.

### `Explosion_Damage_Enemy`
Enemy-specific explosion damage variant with different falloff parameters.

### `Damage_Area` / `Damage_Area_Trigger`
A zone that continuously or on-entry deals damage to anything inside. Used for lava, poison clouds, electrified surfaces.

### `Lava_Damage`
Specialised damage-over-time for lava surfaces.

### `IDamageable` / `IHealthDamage`
Interfaces implemented by any damageable object. Any component implementing these can receive damage from weapons and explosions.

### `Raycast_Damage_Static`
Applies damage instantly via raycast (hitscan weapons). Fires a ray, hits the first `IDamageable`.

---

## 11. Weapon System

The game uses the **Infima Games Low Poly Shooter Pack** as a base weapon framework, extended with custom Killer Bean logic.

### `Procedural_Weapon` / `ProceduralWeapon_Data`
The gun model is procedurally constructed from parts. `ProceduralWeapon_Data` holds the data for each gun variant.

### `GunMod_Data`
Data for attachments (scopes, barrels, grips). Each mod changes stats.

### `Weapon_Part_Stats` / `Weapon_Part_Stats_SHORTCUTS`
Stats on a per-part basis (damage, accuracy, range, fire rate). `SHORTCUTS` is a utility wrapper for quick access.

### `Weapon_Manual_Attributes`
Manual overrides for weapon properties that can't be procedurally determined.

### `Weapon_Patterns`
Defines firing patterns (single shot, burst, full-auto) and shot spread.

### `Weapon_Skins`
Handles visual skin swaps on weapons.

### `Weapon_Pickup`
Placed in the world. When the player interacts, swaps their current weapon.

### `Weapon_Randomizer`
At spawn points, randomly picks a weapon from a pool.

### `WeaponAttachmentManager`
Manages equipping/unequipping attachments and updates stats accordingly.

### `PlayerWeapon`
The player's current weapon slot manager. References active weapon components.

### `Gun_Magazines_in_left_hand`
Animates magazines in the player's left hand during reload.

### `F3DProjectile` / `F3D_Spray_Beam_Turn_Off`
The F3D (Forge 3D) projectile system handles visual bullet trails, impact effects, and spray patterns.

### `Ejected_Bullet_Trick_Control` / `Ejected_Bullet_Trick_script`
Spawns ejected shell casings with stylized slow-mo trajectories during bullet tricks.

### Named Gun Scripts
| Class | Weapon |
|-------|--------|
| `ActualGun` | Generic gun base class |
| `FlameGun` | Flamethrower |
| `SentryGun_AI` | Auto-turret weapon |
| `RailGun` | High-velocity railgun |
| `PlasmaGun` | Plasma energy weapon |
| `LightningGun` | Chain lightning weapon |
| `MiniGun` | Minigun (high fire rate) |
| `GoldGun` | Special gold-plated weapon variant |
| `ShotGun` | Shotgun with spread pattern |
| `HeavyGun` | Slow, high damage weapon |
| `SoloGun` | Single-shot high power gun |
| `SpawnGun` | Spawns entities instead of projectiles |
| `Canon_Projectile_gun_script` | Heavy cannon |
| `rocket_launcher` | RPG-style rocket launcher |
| `Bomber_Gun_script` | Bombing weapon |
| `GrantRopeWeapon` | Grappling rope weapon |
| `GrantStingerWeapon` | Stinger missile weapon |
| `GrantBallWeapon` | Ball/grenade launcher |

---

## 12. Currency System — `CurrencyManager`

**File:** `CurrencyManager.cs`  
**Namespace:** `Il2Cpp`  
**Type:** `MonoBehaviour`, singleton

Three separate currency types exist in the game. All tracked by this singleton.

```csharp
CurrencyManager cm = CurrencyManager.Instance;
```

### Currency Types
| Currency | Fields/Events | Description |
|----------|--------------|-------------|
| **Money** (main) | `OnCurrencyChanged`, `OnCurrencyAdded`, `OnCurrencySpent` | Standard mission/kill reward currency |
| **Eggs** | `OnEggsChanged`, `OnEggsAdded`, `OnEggsSpent` | Secondary collectible currency |
| **Discs** | `OnDiscsChanged`, `OnDiscsAdded`, `OnDiscsSpent` | Premium / rare currency type |

### Configuration Fields
| Field | Type | Description |
|-------|------|-------------|
| `startingCurrency` | `int` | Currency at game start |
| `waveCompletionBonus` | `int` | Currency given for completing a wave |
| `eliteWaveBonus` | `int` | Bonus for an elite wave |
| `bossWaveBonus` | `int` | Bonus for a boss wave |
| `currencyText` | `TMP_Text` | UI text showing current money |

### Debug Fields (force-set values in editor)
| Field | Description |
|-------|-------------|
| `debug_stuff_money` | Force-start with this money amount |
| `debug_stuff_eggs` | Force-start with this egg count |
| `debug_stuff_discs` | Force-start with this disc count |

---

## 13. Mission System

### `Mission_Manager`
Global controller for the active mission. Tracks mission state, triggers objective updates, and fires completion events.

### `Mission_MASTER`
Per-mission root component. Connects mission goals, dialogue, spawn points, and completion conditions.

### `Mission_MASTER_Spawner`
Handles spawning enemies defined by the active mission.

### `Mission_MASTER_Dialogue`
Manages mission briefing and objective update voice lines / comms sequences.

### `Mission_Steps` / `MissionStep` / `MissionStepType`
Mission objectives are broken into steps. Each step has a type (kill X, reach location, retrieve item, protect target, etc.).

### `Mission_Goal_Locations` / `Mission_Location`
Defines the world-space locations relevant to mission objectives.

### `Mission_Object_Activate`
Activates or deactivates world objects when a specific mission step is reached.

### `Mission_Stats_UI`
HUD element showing current mission progress.

### `Trigger_Mission_Step_Goal`
A trigger volume — when the player enters it, a mission step is marked complete.

### `Create_Quest` / `Quest_Giver_script`
Interfaces with the **Pixel Crushers Quest Machine** middleware for side quests and world quests.

---

## 14. Campaign System

### `Campaign`
Top-level campaign data container. Holds a sequence of `Campaign_Section` objects.

### `Campaign_Director` / `Campaign_Director_for_QuestMachine`
Drives the campaign forward, managing which section is active and transitioning between them.

### `Campaign_Section`
A chapter or act within the campaign. Contains ordered missions and unlock conditions.

### `Campaign_Gen`
Procedural campaign generator — creates missions dynamically based on parameters.

### `Campaign_Gen_Goal_Location`
Selects world locations for procedurally generated mission goals.

### `Campaign_Gen_Report_Progress`
Feeds completion data back to the campaign generator to unlock next steps.

### `Campaign_Full_Outline`
Data asset defining the full hand-crafted campaign structure.

### `CampaignSaveData`
Serializable save data for campaign progress.

---

## 15. World & Environment

### `Game_Master_Variables`
**File:** `Game_Master_Variables.cs`  
The global settings object for gameplay-affecting world variables.

| Field | Type | Description |
|-------|------|-------------|
| `mission_active` | `bool` | Whether a mission is currently running |
| `using_campaign_gen` | `bool` | Using procedural campaign generation |
| `mode_microworld` | `bool` | Micro-world mode active |
| `mode_conquest` | `bool` | Conquest mode active |
| `Panda_BT_acitvation_dist` | `float` | Radius around player within which enemy BT ticks |
| `stuff_auto_aim_percent` | `float` | Auto-aim assist strength (0 = off, 1 = full) |
| `_var_enemy_sight_distance` | `FloatVariable` | How far enemies can see the player |
| `_var_explosion_countdown` | `FloatVariable` | Default fuse/countdown for explosions |
| `_var_aquire_player_countdown_max` | `FloatVariable` | How long enemies take to fully lock onto the player |
| `all_weapons_in_game` | `GameObject[]` | All weapon prefabs registered in the game |
| `debug_is_nighttime` | `bool` | Debug override to force nighttime |
| `neardeath_heartbeat` | `AudioSource` | Heartbeat audio for near-death state |

### Methods
| Method | Description |
|--------|-------------|
| `Become_Day()` | Transitions the world to daytime lighting, LUT, and atmosphere |
| `Become_Night()` | Transitions the world to nighttime |

### `DayNight`
A lighter component on specific objects that registers with the `Game_Master_Variables` day/night system to toggle or tween their visual properties.

### `Night_Light_control`
Controls which lights turn on/off during day/night transitions.

### `rotate_skybox`
Slowly rotates the skybox texture over time for dynamic cloud/sky effect.

---

## 16. Health Manager

### `HealthManager`
A general-purpose health component used on the player (separate from `PlayerStats`). `Player_Shooting_Control` holds a reference to this. Handles the low-level health bar UI update.

### `SimpleHealth`
Lightweight health component for simple destructible objects and minor enemies. Has a health value, a damage threshold, and a death event.

### `IPlayerHealth`
Interface implemented by all player health classes. Abstracts health access so different player configurations can plug in.

### `AnimateHealth`
Animates a health bar UI element smoothly as health changes.

### `ArmorHealth`
Health variant with an armor layer. Damage hits armor first before health.

### `Destroyable_Objective_Health`
Health component on mission objective objects (crates, generators, etc.). Death triggers mission failure or objective completion.

### `Explodable_Prop_Health`
Health on explosive barrels and props. On death, triggers `Explodable_Prop_explosion`.

### `SimplePlayerHealth`
Minimal player health for demo/test scenes.

---

## 17. Audio

### `AudioManager`
Singleton managing all game audio. Handles music layers, SFX pools, and ambient sound.

### `RandomAudioPlayer`
Picks randomly from a list of `AudioClip` references and plays one. Used on impacts, footsteps, voice barks.

### `Music_Manager`
Controls the music system. Likely coordinates with `AdaptiveStemMusicManager` (a middleware) for dynamic music that reacts to gameplay intensity.

### `AdaptiveStemMusicManager`
Third-party middleware: cross-fades music stems based on game state (combat, exploration, stealth).

### `KB_Voice`
Manages voice line playback for Killer Bean. Plays barks on events (kill, low health, dive, etc.).

### `Enemy_Voice` / `Enemy_Voice_Manager` / `Enemy_Voice_Library`
Enemy voice bark system. `Enemy_Voice_Library` holds clip pools per enemy type.

### `DialogueDuckingManager`
Lowers background audio volume when dialogue is playing.

### `AudioClipPlayer`
Simple utility: plays an `AudioClip` on `OnEnable`. Useful for one-shot effects.

---

## 18. UI Systems

### `UI_Menu_In_Game`
The main in-game pause/option menu.

### `UI_missions`
Displays active mission objectives on the HUD.

### `UI_Map_Icon` / `UI_Map_Icon_always_up`
World-space icons that appear on the minimap. `always_up` variant keeps the icon vertically aligned regardless of camera tilt.

### `UI_HealthBar_FIX`
Patches a display bug in the health bar; keeps it updated when health changes via direct field writes (bypasses the event system).

### `HurtHUD`
Screen-edge directional damage indicator (red flash on the side the player was hit from).

### `ScreenDamage`
Full-screen blood splatter VFX overlay triggered by taking damage.

### `UI_KB_Biohack_Stats`
HUD panel showing Biohack ability stats.

### `UI_KB_Gun_Mod_Stats`
HUD panel showing stats of the currently equipped gun mod.

### `UI_Elite_Weapon_Stats`
HUD panel for elite/legendary weapon stats display.

### `UI_Key_Rebinding` / `UI_KeyBindings_Manager`
In-game key rebinding menu.

### `UI_Mouse_Sensitivity`
Mouse sensitivity slider in settings.

### `CircleSlider`
Custom circular slider UI component.

### `SliderValuePass` / `TMSliderValuePass`
Passes a slider's float value to another component.

### `ImageColorControl` / `TextColorControl` / `TMTextColorControl`
Tint/color control utilities for UI images and text.

### `UI_Cursor_Access`
Manages cursor lock/unlock state between gameplay and menu modes.

### `Cloud_Status_Icon`
Displays cloud save status in UI.

---

## 19. Save System — `KB_Save_Manager`

**File:** `KB_Save_Manager.cs`  
**Namespace:** `Il2Cpp`

Handles all game saves. Persists player progress, campaign state, settings, and inventory. Uses `CampaignSaveData` and `SettingsSaveData` as serializable containers.

### `SettingsSaveData`
Serializable struct holding all player preferences: graphics quality, sensitivity, keybinds, audio levels.

### `KB_Save_Manager` Methods (from naming pattern)
- Save / load campaign progress
- Save / load player stats (health upgrades, unlocks)
- Save / load settings
- Delete save data

---

## 20. Spawning System

### `Enemy_Spawner`
Spawns enemies at defined points. Configurable: enemy type, count, delay, wave triggers.

### `Enemy_Infinite_Spawner`
Continuously respawns enemies with a configurable delay. Used in endless/survival modes.

### `Enemy_Area_Spawner`
Spawns enemies within a defined area volume.

### `Tower_Area_Spawner`
Spawns enemies from tower positions in conquest mode.

### `Squad_Spawner` / `Squad_Manager`
Spawns coordinated enemy squads. The manager keeps track of all squad members and triggers group tactics.

### `Mission_MASTER_Spawner`
Spawns enemies specific to the current mission step.

### `Spawn_Point_*` Classes
There are 20+ `Spawn_Point_X` classes for different spawn types:
- `Spawn_Point_NPC` — friendly/neutral NPCs
- `Spawn_Point_Sniper` — sniper positions
- `Spawn_Point_Turrent` — turret spawn points
- `Spawn_Point_Prisoner` — prisoner/hostage spawn
- `Spawn_Point_Elite_Gun` — elite weapon spawn
- `Spawn_Point_Cutscene_Bean` — scripted scene beans
- etc.

### `CreaturePoolManager`
Object pool for creature/enemy GameObjects. Reuses instances instead of instantiating/destroying to avoid GC spikes.

### `particleFX_DeSpawn_PoolManager`
Object pool specifically for particle effect GameObjects.

### `Grid_Spawn`
Spawns objects in a grid pattern.

### `PA_spawner`
Spawner for the ProceduralAnimation demo system.

---

## 21. Vehicle System

The vehicle system uses **RCC (Realistic Car Controller)** as a base with custom Killer Bean overrides.

### `CarSimulator`
Custom lightweight car physics for AI vehicles (not the player's).

### `JetSkiInputBridge`
Bridges Killer Bean input to the jet ski vehicle controls.

### `RoverControler_A`
Rover-type vehicle controller.

### `Enemy_Vehicle_AI_NavAgent` / `Enemy_Vehicle_AI_Omni`
AI driving behaviors for enemy vehicles.

### `Enemy_Car_Nav_Controller`
Enemy car that follows NavMesh paths to pursue the player.

### `Enemy_Car_Passengers`
Manages passengers in enemy vehicles: when the car is destroyed, passengers exit or die.

### `Enemy_Vehicle_Transport`
Enemy vehicle that transports units from one location to another.

### `Motorcycle_Enemy_Targetter`
Enemy motorcycle AI that keeps the player in its path.

### `ArcadeBikeController` / `BikeController`
Player-ridden bike/motocross controls.

### `Vehicle_path_point` / `Waypoint_Navagent_Vehicle`
Defines paths for vehicle AI to follow.

### `AxleInfo`
Data for each wheel axle on a vehicle (drive, steer, suspension).

### `Car_Collider_Control`
Manages the vehicle's collision shape at runtime.

### `Car_Ram_Damage`
Applies damage to anything the car physically rams into.

### `Car_Water_Behavior`
Modifies car physics when driving through water.

### `Grant_Boat_Exit` / `GrantBoat_Intro_Cutscene`
Scripted boat entry/exit sequences.

---

## 22. Boss Enemies

### `Boss_Simple_AI` / `Boss_Simple_Intro`
Base boss AI with an introduction sequence before combat begins.

### `Boss_Attack_Trigger` / `Boss_Projectile_Shooter` / `Boss_Projectile_Slow` / `Boss_Projectile_Trigger`
Boss attack phase components. The projectile system creates slow-moving special projectiles.

### `Boss_Dialogue`
Plays boss voice lines at scripted health thresholds.

### `BossCylinderAttack`
A cylindrical area attack used by a boss.

### `SphericalDroneBoss`
A flying sphere drone boss with 360-degree attacks.

### `Giant_Mech_script` / `Giant_Mech_Anim_Events` / `Giant_Mech_Ocean_Manager`
The giant mech boss — has its own animation event system and special ocean traversal logic.

### `MechDefenseSystem`
The mech's defensive countermeasure system (shields, missile intercept).

### `MechLegPlatformECM2`
Makes the player able to stand on the mech's legs as platforms.

---

## 23. Special Beans

### `ShadowBeanAI` / `ShadowBeanMelee` / `ShadowBeanProjectileSystem` / `ShadowBeanExplosionMover`
Shadow Bean: fast assassin-type enemy. Teleports, uses melee and projectile hybrid attacks. The explosion mover launches him backward on certain attacks.

### `WarlordBean`
A powerful named enemy bean with unique attacks and dialogue.

### `HammerBean`
Enemy that wields a giant hammer. Slow but devastating melee range.

### `HeavyBean` / `HeavyBean_Materials_List`
Armored heavy enemy. The materials list allows for dynamic armor damage visualization.

### `Necro_Bean_AI`
Undead bean that revives unless finished with a specific kill method.

### `GrantBean` / `GrantBeanMinion`
Grant is a character in the game. `GrantBeanMinion` are AI-controlled beans that accompany or fight for Grant.

### `CommsBean`
A bean used in communications/cutscene sequences.

### `DeadBean`
The corpse ragdoll used when a bean dies.

### `Bean_ID`
Tags a bean with an identity (friendly, enemy, neutral, key character).

### `is_this_a_Demo_Bean`
Flag component marking beans in demo/tutorial sequences.

---

## 24. Interaction System — `KB_Interactor`

**File:** `KB_Interactor.cs`  
Manages all player interactions: picking up items, opening doors, entering vehicles, interacting with NPCs.

### Related Components
- `Interact_Glyph` — Floating prompt (button icon) shown near interactable objects
- `Door_01` / `Door_02` / `DoorHinge` / `DoorDoubleRotate` / `DoorDoubleSlide` / `DoorVert` / `DoorHori` — Various door types with open/close logic
- `Container_Open_Close` / `Container_Example_Keys_01` — Openable containers (crates, chests)
- `Crate_Open_Close` — Crate variant with physics destruction
- `PickingUp` — Generic pickup interaction component
- `Loot_Pickup` / `Loot_Stats` / `Loot_Trigger_Pickup` — Loot items with stat data
- `Quest_Loot_Pickup` — Loot tied to a quest objective
- `Special_Item` — Key items that trigger scripted events

---

## 25. Skills System

### `Player_Skills`
Manages the player's unlocked skills and their active/passive effects.

### `Skill`
Individual skill data: name, description, unlock cost, effect type, and magnitude.

### `Skill_Cards_List`
Pool of available skill cards to offer the player.

### `Skill_Cards_Offered`
The current selection of skill cards being presented to the player for a choice.

### `Biohack_Data` / `Procedural_Biohack`
The Biohack system is a special skill/ability category. Biohacks are procedurally generated ability modifiers.

### `Stat_Upgrade_Item`
An item in the world or shop that permanently upgrades a stat when collected/purchased.

---

## 26. Shop System — `ShopManager`

**File:** `ShopManager.cs`  
**Namespace:** `Il2Cpp`

Manages the in-game shop.

### Components
| Class | Description |
|-------|-------------|
| `ShopManager` | Top-level shop controller, singleton |
| `ShopItemData` | Data for a single item: name, cost, type, effect |
| `ShopItemButton` | UI button for a shop item |
| `ShopUIManager` | Manages the shop UI panels |
| `Shop_Keeper` | NPC that opens the shop |
| `Shopping_Load_Save_File` | Loads and saves shop state (purchased items) |
| `Store_Rack` | 3D weapon/item display rack in the shop |
| `Store_Item_Cam_Selector` | Switches the preview camera to focus on the selected item |
| `Store_Item_Turn_Off_All_Scripts` | Disables gameplay scripts on an item when it's on display |

---

## 27. Wave System — `WaveManager`

**File:** `WaveManager.cs`  
**Namespace:** `Il2Cpp`

Manages enemy waves in endless/survival mode.

### `WaveManager`
Controls wave sequencing: spawns a wave, waits for all enemies to die, then spawns the next.

### `WaveUIManager`
HUD element showing current wave number and countdown to next wave.

### `Battle_Arena_Control`
Sets up the arena for a wave battle: closes exits, spawns terrain modifiers.

### `BattleSim_Control` / `BattleSim_Cam_control` / `BattleSim_UI_menu` / `BattleSim_Log`
The battle simulation system — a top-down battle viewer/simulator mode.

---

## 28. Object & Transform Utilities

These are small utility components found throughout the game:

| Class | Description |
|-------|-------------|
| `MoveMe` | Lerps a GameObject to a target position |
| `RotateMe` | Continuously rotates a GameObject |
| `Rotate` / `Rotate_Object` / `Rotator` | Various rotation utilities |
| `SmoothTransform` | Smoothly interpolates transform towards a target |
| `simple_mover` | Basic oscillating mover |
| `sinewave_move` | Moves in a sine wave pattern |
| `ResetPosition` | Resets object to its start position on demand |
| `Teleport` / `Teleporter` | Teleports objects/player to a destination |
| `Lock_PosY` | Locks the Y position of an object |
| `Floater` | Bobs an object up and down (water floating) |
| `Pendulum` | Swings an object like a pendulum |
| `Billboard` / `Billboard_lamp` | Keeps an object always facing the camera |
| `ObjectFacingCamera` | Same as Billboard, different implementation |
| `JustRotate` | Single-purpose continuous rotation |
| `SphereScaler` | Scales a sphere collider to match a transform |
| `Randomize_Scale` | Randomly scales an object within a range on start |
| `UNI_ResetTransformOnStart` | Saves and restores the transform on scene reload |
| `UNI_EnableAfterDelay` | Enables a GameObject after N seconds |
| `GameObjectDieAfter` | Destroys a GameObject after N seconds |
| `Delayed_Destroy` | Destroys after a configurable delay |
| `Delayed_Despawn_to_Pool` | Returns to object pool after a delay |
| `ProximityActivate` | Activates objects when the player is within range |
| `DEBUG_Gameobject_Activation` | Debug toggle for enabling/disabling objects |

---

## 29. Third-Party Middleware Used

The game integrates many Unity asset store packages. This is important for modding — some types you'll see in the decompiled code come from these:

| Package | Namespace Prefix | Used For |
|---------|-----------------|----------|
| Animancer | `Il2CppAnimancer` | Animation state machine for player and enemies |
| ECM2 (Easy Character Motion 2) | `Il2CppECM2` | Physics-based character controller |
| Infima Games LowPoly Shooter | `Il2CppInfimaGames.LowPolyShooterPack` | Weapon handling, ADS, reload logic |
| Cinemachine | `Il2CppCinemachine` | Camera system |
| DOTween / DOTween Pro | `Il2CppDG.Tweening` | Tween animations on UI and world objects |
| PandaBT | `Il2CppEnemyAI` | Enemy behavior trees |
| Sensor Toolkit | `Il2CppMicosmo.SensorToolkit` / `Il2CppSensorToolkit` | Sight/hearing sensors for AI |
| More Mountains Feedbacks | `Il2CppMoreMountains.Feedbacks` / `.Tools` | Camera shake, hit feedback |
| SOAP (ScriptableObject Architecture) | `Il2CppObvious.Soap` | `FloatVariable`, `IntVariable` etc. — scriptable object event variables |
| Quest Machine | `Il2CppPixelCrushers.QuestMachine` | Side quest system |
| Final IK | `Il2CppRootMotion.FinalIK` | Inverse kinematics for limb placement |
| Trails FX | `Il2CppTrailsFX` | Bullet/movement trail effects |
| Adaptive Stems Music | `Il2CppAdaptiveStemMusic` | Dynamic music layering |
| RCC (Realistic Car Controller) | `Il2CppEVP` / `RCC_*` | Vehicle physics |
| BuildingCrafter | `Il2CppBuildingCrafter` | Procedural building generation |
| Very Animation | `Il2CppAloneSoft.VeryAnimation` | Animation editing tool (editor only) |
| Highlight Plus | `Il2CppHighlightPlus` | Object highlight outlines |
| Beautify | `Il2CppBeautify` | Post-processing effects |
| AllIn1 3D Shader | `Il2CppAllIn13DShader*` | Special shader effects |
| Gley (Mobile) | `Il2CppGley` | Mobile input support |
| Infima Games FPS | `Il2CppInfimaGames` | FPS weapon pack base |
| FIMSpace | `Il2CppFIMSpace` | IK and procedural animation helpers |
| Rewired | `Rewired_*` | Legacy gamepad/input system |

---

## 30. Quick Mod Reference

The most commonly needed access patterns from mod code:

### Get Player Stats
```csharp
using Il2Cpp;
var ps = PlayerStats.Instance; // null if in menu
if (ps != null) {
    ps.currentHealth = ps.GetMaxHealth();       // full heal
    ps.baseDamageMultiplier = 5f;               // 5x damage
    ps.moveSpeedMultiplier = 2f;                // 2x speed
    ps.fireRateMultiplier = 3f;                 // 3x fire rate
    ps.UnlockDoubleJump();                      // give double jump
    ps.hasDodgeRoll = true;                     // unlock dodge
    string debug = ps.GetStatsDebugString();    // dump all stats
}
```

### Subscribe to Player Death
```csharp
PlayerStats.Instance?.add_OnDeath(new Action(() => {
    MelonLogger.Msg("Player died!");
}));
```

### Get Currency
```csharp
var cm = CurrencyManager.Instance;
cm?.add_OnCurrencyChanged(new Action<int>((amount) => {
    MelonLogger.Msg($"Currency changed: {amount}");
}));
```

### Toggle Day/Night
```csharp
var gmv = GameObject.FindObjectOfType<Game_Master_Variables>();
gmv?.Become_Day();
gmv?.Become_Night();
// or reduce enemy sight:
// gmv._var_enemy_sight_distance.Value = 5f;  // requires FloatVariable ref
```

### Scale Enemies
```csharp
foreach (var go in GameObject.FindGameObjectsWithTag("Enemy"))
    go.transform.localScale = Vector3.one * 0.5f; // tiny enemies
```

### Get Input State
```csharp
var pih = GameObject.FindObjectOfType<Player_Input_Handler>();
if (pih != null) {
    pih.can_fire = false;    // disarm player
    pih.is_slowmo_on = true; // force slowmo
}
```

### Harmony Patch Example (block player death)
```csharp
[HarmonyPatch(typeof(PlayerStats), "Die")]
public static class NoDeath {
    static bool Prefix() => false; // skip Die() entirely
}
```

### Harmony Patch Example (multiply damage taken)
```csharp
[HarmonyPatch(typeof(PlayerStats), "TakeDamage")]
public static class ReduceDamage {
    static void Prefix(ref float damage) {
        damage *= 0.1f; // take 10% damage
    }
}
```

---

*Document generated from decompiled `Assembly-CSharp.dll` (Unity 6000.0.75f1, IL2CPP) via MelonLoader interop + dnSpy export. Class behaviour is inferred from field names, method signatures, and cross-references — some descriptions are approximations. Verify against in-game behaviour when modding.*
