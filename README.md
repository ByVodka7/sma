#include <amxmodx>
#include <fakemeta_util>
#include <hamsandwich>
#include <zombieplague>

// ~ [ Macroses ] ~ //
#define IsCustomItem(%0) (pev(%0, pev_impulse) == WEAPON_SPECIAL_CODE)
#define IsCustomMuzzle(%0) (pev(%0, pev_impulse) == gl_iszAllocString_MuzzleKey)
#define IsPdataSafe(%0) (pev_valid(%0) == 2)

#define get_WeaponState(%1) (get_pdata_int(%1, m_iWeaponState, linux_diff_weapon))
#define set_WeaponState(%1,%2) (set_pdata_int(%1, m_iWeaponState, %2, linux_diff_weapon))

#define weaponHasMaxHits(%0) (get_pdata_int(%0, m_iHitCount, linux_diff_weapon) >= CHARGE_HIT_COUNT)
#define m_iHitCount m_iGlock18ShotsFired

// ~ [ Extra Item ] ~ //
new const WEAPON_ITEM_NAME[] = "Infinity Laser Fist";
const WEAPON_ITEM_COST = 0;

// ~ [ Weapon Settings ] ~ //
new const WEAPON_NATIVE[] = "zp_give_user_laserfist";

new const WEAPON_REFERENCE[] = "weapon_m249";
new const WEAPON_WEAPONLIST[] = "weapon_laserfist";
new const WEAPON_ANIMATION[] = "dualpistols";
const WEAPON_SPECIAL_CODE = 152023;

new const WEAPON_MODEL_VIEW[] = "models/v_laserfist.mdl";
new const WEAPON_MODEL_VIEW_2[] = "models/v_laserfist_2.mdl";
new const WEAPON_MODEL_PLAYER[] = "models/p_laserfist.mdl";
new const WEAPON_MODEL_PLAYER_2[] = "models/p_laserfist_2.mdl";
new const WEAPON_MODEL_WORLD[] = "models/w_laserfist.mdl";

new const WEAPON_SOUNDS[][] = 
{
	"weapons/laserfist_shoota-1.wav", 
	"weapons/laserfist_shootb-1.wav", 
	"weapons/laserfist_shootb_exp.wav"
};

new const WEAPON_MUZZLEFLASH_CLASSNAME[] = "ent_laserfist_mf";
new const WEAPON_MUZZLEFLASH_SPRITE[][] =
{
	"sprites/muzzleflash86.spr", 
	"sprites/muzzleflash87.spr", 
	"sprites/muzzleflash91.spr", 
	"sprites/muzzleflash92.spr"
};

new const WEAPON_SPRITES[][] = 
{
	"sprites/ef_laserfist_laser_explosion.spr", 
	"sprites/ef_laserfist_laserbeam.spr"
};

const WEAPON_DEFAULT_AMMO = 1000;
const WEAPON_MAX_CLIP = 500;
const Float: WEAPON_RATE = 0.05;
const Float: WEAPON_PUNCHANGLE = 0.2;
const Float: WEAPON_DAMAGE = 25.0;

const CHARGE_HIT_COUNT = 60;
const Float: WEAPON_EXP_DMG = 3085.0;
const Float: WEAPON_EXP_RADIUS = 300.0;
const Float: WEAPON_EXP_KNOCKBACK = 300.0;
const WEAPON_EXP_DMGTYPE = DMG_BULLET;

new const iWeaponList[] =
{
	3, 200, -1, -1, 0, 4, 20, 0 // weapon_m249
};

// ~ [ Weapon Animation ] ~ //
// From model: Frames/FPS
#define WEAPON_ANIM_IDLE_TIME 141/30.0
#define WEAPON_ANIM_SHOOTA_LOOP_TIME 21/60.0
#define WEAPON_ANIM_SHOOTA_END_TIME 31/30.0
#define WEAPON_ANIM_SHOOTB_READY_TIME 39/30.0
#define WEAPON_ANIM_SHOOTB_LOOP_TIME 31/30.0
#define WEAPON_ANIM_SHOOTB_SHOOT_TIME 50/30.0
#define WEAPON_ANIM_RELOAD_TIME 81/30.0
#define WEAPON_ANIM_DRAW_TIME 51/30.0

enum _: eWeaponAnim
{
	WEAPON_ANIM_IDLE = 0,
	WEAPON_ANIM_SHOOTA_EMPTY_LOOP,
	WEAPON_ANIM_SHOOTA_EMPTY_END,
	WEAPON_ANIM_SHOOTA_LOOP,
	WEAPON_ANIM_SHOOTA_END,
	WEAPON_ANIM_SHOOTB_READY,
	WEAPON_ANIM_SHOOTB_LOOP,
	WEAPON_ANIM_SHOOTB_SHOOT,
	WEAPON_ANIM_RELOAD,
	WEAPON_ANIM_DRAW
};

// ~ [ Weapon State ] ~ //
enum _: eWeaponState
{
	WPNSTATE_NULL,
	WPNSTATE_CHARGE_READY,
	WPNSTATE_CHARGE_LOOP
};

// ~ [ Offsets ] ~ //
const m_iClip = 51;
const linux_diff_animating = 4;
const linux_diff_player = 5;
const linux_diff_weapon = 4;
const m_rpgPlayerItems = 367;
const m_pNext = 42
const m_iId = 43;
const m_iPrimaryAmmoType = 49;
const m_rgAmmo = 376;
const m_flNextAttack = 83;
const m_flTimeWeaponIdle = 48;
const m_flNextPrimaryAttack = 46;
const m_flNextSecondaryAttack = 47;
const m_pPlayer = 41;
const m_fInReload = 54;
const m_pActiveItem = 373;
const m_rgpPlayerItems_iItemBox = 34;
const m_maxFrame = 35;
const m_iGlock18ShotsFired = 70;
const m_szAnimExtention = 492;
const m_iWeaponState = 74;

// ~ [ Params ] ~ //
new HamHook: gl_HamHook_TraceAttack[4],

	gl_iszAllocString_Entity,
	gl_iszAllocString_ModelView,
	gl_iszAllocString_ModelView_2,
	gl_iszAllocString_ModelPlayer,
	gl_iszAllocString_ModelPlayer_2,
	gl_iszAllocString_MuzzleKey,

	gl_iszModelIndex_SmokePuff,
	gl_iszModelIndex_Sprites[sizeof WEAPON_SPRITES],

	gl_iMsgID_Weaponlist,
	
	gl_iItemID;

// ~ [ AMX Mod X ] ~ //
public plugin_init()
{
	//https://cso.fandom.com/wiki/Infinity_Laser_Fist
	register_plugin("[ZP] Extra-Item: Infinity Laser Fist", "v2.8 | 2023", "Author: Marc2k16 / Code Base: Cristian 505 & Batcoh")

	// Fakemeta
	register_forward(FM_UpdateClientData,	"FM_Hook_UpdateClientData_Post",	true);
	register_forward(FM_SetModel,		"FM_Hook_SetModel_Pre",			false);

	// Weapon
	RegisterHam(Ham_Item_Deploy,		WEAPON_REFERENCE,	"CWeapon__Deploy_Post",		true);
	RegisterHam(Ham_Weapon_PrimaryAttack,	WEAPON_REFERENCE,	"CWeapon__PrimaryAttack_Pre",	false);
	RegisterHam(Ham_Weapon_Reload,		WEAPON_REFERENCE,	"CWeapon__Reload_Pre",		false);
	RegisterHam(Ham_Item_PostFrame,		WEAPON_REFERENCE,	"CWeapon__PostFrame_Pre",	false);
	RegisterHam(Ham_Item_Holster,		WEAPON_REFERENCE,	"CWeapon__Holster_Post",	true);
	RegisterHam(Ham_Item_AddToPlayer,	WEAPON_REFERENCE,	"CWeapon__AddToPlayer_Post",	true);

	// Entity
	RegisterHam(Ham_Think,			"env_sprite",		"CMuzzleFlash__Think_Pre",	false);

	// Trace Attack
	gl_HamHook_TraceAttack[0] = RegisterHam(Ham_TraceAttack,	"func_breakable",	"CEntity__TraceAttack_Pre",  false);
	gl_HamHook_TraceAttack[1] = RegisterHam(Ham_TraceAttack,	"info_target",		"CEntity__TraceAttack_Pre",  false);
	gl_HamHook_TraceAttack[2] = RegisterHam(Ham_TraceAttack,	"player",		"CEntity__TraceAttack_Pre",  false);
	gl_HamHook_TraceAttack[3] = RegisterHam(Ham_TraceAttack,	"hostage_entity",	"CEntity__TraceAttack_Pre",  false);

	// Alloc String
	gl_iszAllocString_Entity = engfunc(EngFunc_AllocString, WEAPON_REFERENCE);
	gl_iszAllocString_ModelView = engfunc(EngFunc_AllocString, WEAPON_MODEL_VIEW);
	gl_iszAllocString_ModelView_2 = engfunc(EngFunc_AllocString, WEAPON_MODEL_VIEW_2);
	gl_iszAllocString_ModelPlayer = engfunc(EngFunc_AllocString, WEAPON_MODEL_PLAYER);
	gl_iszAllocString_ModelPlayer_2 = engfunc(EngFunc_AllocString, WEAPON_MODEL_PLAYER_2);
	gl_iszAllocString_MuzzleKey = engfunc(EngFunc_AllocString, WEAPON_MUZZLEFLASH_CLASSNAME);

	// Messages
	gl_iMsgID_Weaponlist = get_user_msgid("WeaponList");

	fm_ham_hook(false);

	// Chat command
	//register_clcmd("say /lf", "Command_GiveWeapon");

	// Register on Extra-Items
	gl_iItemID = zp_register_extra_item(WEAPON_ITEM_NAME, WEAPON_ITEM_COST, ZP_TEAM_HUMAN);
}

public plugin_precache()
{
	new i;

	// Hook weapon
	register_clcmd(WEAPON_WEAPONLIST, "Command_HookWeapon");

	// Precache models
	engfunc(EngFunc_PrecacheModel, WEAPON_MODEL_VIEW);
	engfunc(EngFunc_PrecacheModel, WEAPON_MODEL_VIEW_2);
	engfunc(EngFunc_PrecacheModel, WEAPON_MODEL_PLAYER);
	engfunc(EngFunc_PrecacheModel, WEAPON_MODEL_PLAYER_2);
	engfunc(EngFunc_PrecacheModel, WEAPON_MODEL_WORLD);

	for(i = 0; i < sizeof WEAPON_MUZZLEFLASH_SPRITE; i++)
		engfunc(EngFunc_PrecacheModel, WEAPON_MUZZLEFLASH_SPRITE[i]);

	// Precache generic
	new szWeaponList[128]; formatex(szWeaponList, charsmax(szWeaponList), "sprites/%s.txt", WEAPON_WEAPONLIST);
	engfunc(EngFunc_PrecacheGeneric, szWeaponList);

	// Precache sounds
	for(i = 0; i < sizeof WEAPON_SOUNDS; i++)
		engfunc(EngFunc_PrecacheSound, WEAPON_SOUNDS[i]);

	// Model Index
	gl_iszModelIndex_SmokePuff = engfunc(EngFunc_PrecacheModel, "sprites/fast_wallpuff1.spr");

	for(i = 0; i < sizeof WEAPON_SPRITES; i++)
		gl_iszModelIndex_Sprites[i] = engfunc(EngFunc_PrecacheModel, WEAPON_SPRITES[i]);
}

public plugin_natives() register_native(WEAPON_NATIVE, "Command_GiveWeapon", 1);

public Command_HookWeapon(iPlayer)
{
	engclient_cmd(iPlayer, WEAPON_REFERENCE);
	return PLUGIN_HANDLED;
}

// ~ [ Zombie Plague ] ~ //
public zp_extra_item_selected(iPlayer, iItem)
{
	if(iItem == gl_iItemID)
		Command_GiveWeapon(iPlayer);
}

public Command_GiveWeapon(iPlayer)
{
	static iItem; iItem = engfunc(EngFunc_CreateNamedEntity, gl_iszAllocString_Entity);
	if(!IsPdataSafe(iItem)) return FM_NULLENT;

	set_pev(iItem, pev_impulse, WEAPON_SPECIAL_CODE);
	ExecuteHam(Ham_Spawn, iItem);
	set_pdata_int(iItem, m_iClip, WEAPON_MAX_CLIP, linux_diff_weapon);
	UTIL_DropWeapon(iPlayer, ExecuteHamB(Ham_Item_ItemSlot, iItem));

	if(!ExecuteHamB(Ham_AddPlayerItem, iPlayer, iItem))
	{
		set_pev(iItem, pev_flags, pev(iItem, pev_flags) | FL_KILLME);
		return 0;
	}

	ExecuteHamB(Ham_Item_AttachToPlayer, iItem, iPlayer);
	UTIL_WeaponList(iPlayer, true);

	static iAmmoType; iAmmoType = m_rgAmmo + get_pdata_int(iItem, m_iPrimaryAmmoType, linux_diff_weapon);

	if(get_pdata_int(iPlayer, iAmmoType, linux_diff_player) < WEAPON_DEFAULT_AMMO)
	set_pdata_int(iPlayer, iAmmoType, WEAPON_DEFAULT_AMMO, linux_diff_player);

	emit_sound(iPlayer, CHAN_ITEM, "items/gunpickup2.wav", VOL_NORM, ATTN_NORM, 0, PITCH_NORM);

	return 1;
}

// ~ [ HamSandwich ] ~ //
public CWeapon__Deploy_Post(iItem)
{
	if(!IsPdataSafe(iItem) || !IsCustomItem(iItem)) return;

	static iPlayer; iPlayer = get_pdata_cbase(iItem, m_pPlayer, linux_diff_weapon);

	if(weaponHasMaxHits(iItem))
	{
		set_pev_string(iPlayer, pev_viewmodel2, gl_iszAllocString_ModelView_2)
		set_pev_string(iPlayer, pev_weaponmodel2, gl_iszAllocString_ModelPlayer_2);
	}
	else
	{
		set_pev_string(iPlayer, pev_viewmodel2, gl_iszAllocString_ModelView)
		set_pev_string(iPlayer, pev_weaponmodel2, gl_iszAllocString_ModelPlayer);
	}

	UTIL_SendWeaponAnim(iPlayer, WEAPON_ANIM_DRAW);

	set_pdata_float(iPlayer, m_flNextAttack, WEAPON_ANIM_DRAW_TIME, linux_diff_player);
	set_pdata_float(iItem, m_flTimeWeaponIdle, WEAPON_ANIM_DRAW_TIME, linux_diff_weapon);

	set_pdata_string(iPlayer, m_szAnimExtention * 4, WEAPON_ANIMATION, -1, linux_diff_player * linux_diff_animating);
}

public CWeapon__PrimaryAttack_Pre(iItem)
{
	if(!IsPdataSafe(iItem) || !IsCustomItem(iItem)) return HAM_IGNORED;

	static iAmmo; iAmmo = get_pdata_int(iItem, m_iClip, linux_diff_weapon);
	if(!iAmmo)
	{
		ExecuteHam(Ham_Weapon_PlayEmptySound, iItem);
		set_pdata_float(iItem, m_flNextPrimaryAttack, 0.2, linux_diff_weapon);

		return HAM_SUPERCEDE;
	}

	static fw_TraceLine; fw_TraceLine = register_forward(FM_TraceLine, "FM_Hook_TraceLine_Post", true);
	static fw_PlayBackEvent; fw_PlayBackEvent = register_forward(FM_PlaybackEvent, "FM_Hook_PlaybackEvent_Pre", false);
	fm_ham_hook(true);		

	ExecuteHam(Ham_Weapon_PrimaryAttack, iItem);
		
	unregister_forward(FM_TraceLine, fw_TraceLine, true);
	unregister_forward(FM_PlaybackEvent, fw_PlayBackEvent);
	fm_ham_hook(false);

	static iPlayer; iPlayer = get_pdata_cbase(iItem, m_pPlayer, linux_diff_weapon);

	static Float: vecPunchangle[3];
	pev(iPlayer, pev_punchangle, vecPunchangle);
	vecPunchangle[0] *= WEAPON_PUNCHANGLE
	vecPunchangle[1] *= WEAPON_PUNCHANGLE
	vecPunchangle[2] *= WEAPON_PUNCHANGLE
	set_pev(iPlayer, pev_punchangle, vecPunchangle);

	if(!weaponHasMaxHits(iItem))
	{
		set_pdata_int(iItem, m_iHitCount, get_pdata_int(iItem, m_iHitCount, linux_diff_weapon) + 1, linux_diff_weapon)

		if(weaponHasMaxHits(iItem))
		{
			set_pev_string(iPlayer, pev_viewmodel2, gl_iszAllocString_ModelView_2);
			set_pev_string(iPlayer, pev_weaponmodel2, gl_iszAllocString_ModelPlayer_2);
		}
	}

	if(pev(iPlayer, pev_weaponanim) != WEAPON_ANIM_SHOOTA_LOOP)
		UTIL_SendWeaponAnim(iPlayer, WEAPON_ANIM_SHOOTA_LOOP);

	//UTIL_CreateMuzzleFlash(iPlayer, WEAPON_MUZZLEFLASH_SPRITE[3], 0, 0.051, 100.0, 1, 0.001); // muzzleflash looks buggy and ugly... ;(
	//UTIL_CreateMuzzleFlash(iPlayer, WEAPON_MUZZLEFLASH_SPRITE[3], 0, 0.051, 100.0, 2, 0.001);

	emit_sound(iPlayer, CHAN_WEAPON, WEAPON_SOUNDS[0], VOL_NORM, ATTN_NORM, 0, PITCH_NORM);

	set_pdata_float(iItem, m_flNextPrimaryAttack, WEAPON_RATE, linux_diff_weapon);
	set_pdata_float(iItem, m_flTimeWeaponIdle, WEAPON_ANIM_SHOOTA_LOOP_TIME, linux_diff_weapon);
	

	return HAM_SUPERCEDE;
}

public CWeapon__Reload_Pre(iItem)
{
	if(!IsPdataSafe(iItem) || !IsCustomItem(iItem)) return HAM_IGNORED;

	static iAmmo; iAmmo = get_pdata_int(iItem, m_iClip, linux_diff_weapon);
	if(iAmmo >= WEAPON_MAX_CLIP) return HAM_SUPERCEDE;

	static iPlayer; iPlayer = get_pdata_cbase(iItem, m_pPlayer, linux_diff_weapon);
	static iAmmoType; iAmmoType = m_rgAmmo + get_pdata_int(iItem, m_iPrimaryAmmoType, linux_diff_weapon);

	if(get_pdata_int(iPlayer, iAmmoType, linux_diff_player) <= 0) return HAM_SUPERCEDE;

	set_pdata_int(iItem, m_iClip, 0, linux_diff_weapon);

	ExecuteHam(Ham_Weapon_Reload, iItem);

	set_pdata_int(iItem, m_iClip, iAmmo, linux_diff_weapon);
	set_pdata_int(iItem, m_fInReload, 1, linux_diff_weapon);

	UTIL_SendWeaponAnim(iPlayer, WEAPON_ANIM_RELOAD);

	set_pdata_float(iPlayer, m_flNextAttack, WEAPON_ANIM_RELOAD_TIME, linux_diff_player);
	set_pdata_float(iItem, m_flTimeWeaponIdle, WEAPON_ANIM_RELOAD_TIME, linux_diff_weapon);
	set_pdata_float(iItem, m_flNextPrimaryAttack, WEAPON_ANIM_RELOAD_TIME, linux_diff_weapon);
	set_pdata_float(iItem, m_flNextSecondaryAttack, WEAPON_ANIM_RELOAD_TIME, linux_diff_weapon);

	return HAM_SUPERCEDE;
}

public CWeapon__PostFrame_Pre(iItem)
{ 
	if(!IsPdataSafe(iItem) || !IsCustomItem(iItem)) return HAM_IGNORED;

	static iAttacker; iAttacker = get_pdata_cbase(iItem, m_pPlayer, linux_diff_weapon);
	static iClip; iClip = get_pdata_int(iItem, m_iClip, linux_diff_weapon);
	static iPlayer; iPlayer = get_pdata_cbase(iItem, m_pPlayer, linux_diff_weapon);
	static iButton; iButton = pev(iPlayer, pev_button);
	static iItemState; iItemState = get_WeaponState(iItem);

	if(get_pdata_int(iItem, m_fInReload, linux_diff_weapon) == 1)
	{
		static iAmmoType; iAmmoType = m_rgAmmo + get_pdata_int(iItem, m_iPrimaryAmmoType, linux_diff_weapon);
		static iPlayer; iPlayer = get_pdata_cbase(iItem, m_pPlayer, linux_diff_weapon);
		static iAmmo; iAmmo = get_pdata_int(iPlayer, iAmmoType, linux_diff_player);
		static j; j = min(WEAPON_MAX_CLIP - iClip, iAmmo);
		
		set_pdata_int(iItem, m_iClip, iClip + j, linux_diff_weapon);
		set_pdata_int(iPlayer, iAmmoType, iAmmo - j, linux_diff_player);
		set_pdata_int(iItem, m_fInReload, 0, linux_diff_weapon);
	}

	if(!(iButton & IN_ATTACK) && pev(iPlayer, pev_weaponanim) == WEAPON_ANIM_SHOOTA_LOOP)
	{
		UTIL_SendWeaponAnim(iPlayer, WEAPON_ANIM_SHOOTA_END);

		set_pdata_float(iItem, m_flTimeWeaponIdle, WEAPON_ANIM_SHOOTA_END_TIME, linux_diff_weapon);
	}

	switch(iItemState)
	{
		case WPNSTATE_NULL:
		{
			set_WeaponState(iItem, WPNSTATE_CHARGE_READY);
		}

		case WPNSTATE_CHARGE_READY:
		{
			if(iButton & IN_ATTACK2 && weaponHasMaxHits(iItem))
			{
				UTIL_SendWeaponAnim(iPlayer, WEAPON_ANIM_SHOOTB_READY);

				set_pdata_float(iPlayer, m_flNextAttack, WEAPON_ANIM_SHOOTB_READY_TIME, linux_diff_player);
				set_pdata_float(iItem, m_flTimeWeaponIdle, WEAPON_ANIM_SHOOTB_READY_TIME, linux_diff_weapon);
				set_pdata_float(iItem, m_flNextPrimaryAttack, WEAPON_ANIM_SHOOTB_READY_TIME, linux_diff_weapon);
				set_pdata_float(iItem, m_flNextSecondaryAttack, WEAPON_ANIM_SHOOTB_READY_TIME, linux_diff_weapon);

				set_WeaponState(iItem, WPNSTATE_CHARGE_LOOP);
			}
		}

		case WPNSTATE_CHARGE_LOOP:
		{
			if(iButton & IN_ATTACK2)
			{
				UTIL_SendWeaponAnim(iPlayer, WEAPON_ANIM_SHOOTB_LOOP);

				UTIL_CreateMuzzleFlash(iPlayer, WEAPON_MUZZLEFLASH_SPRITE[2], 0, 0.1, 255.0, 1, 0.04);
				UTIL_CreateMuzzleFlash(iPlayer, WEAPON_MUZZLEFLASH_SPRITE[2], 0, 0.1, 255.0, 2, 0.04);

				set_pdata_float(iPlayer, m_flNextAttack, WEAPON_ANIM_SHOOTB_LOOP_TIME, linux_diff_player);
				set_pdata_float(iItem, m_flTimeWeaponIdle, WEAPON_ANIM_SHOOTB_LOOP_TIME, linux_diff_weapon);
				set_pdata_float(iItem, m_flNextPrimaryAttack, WEAPON_ANIM_SHOOTB_LOOP_TIME, linux_diff_weapon);
				set_pdata_float(iItem, m_flNextSecondaryAttack, WEAPON_ANIM_SHOOTB_LOOP_TIME, linux_diff_weapon);
			}
			else
			{
				
				set_pdata_float(iItem, m_flNextPrimaryAttack, WEAPON_ANIM_SHOOTB_SHOOT_TIME, linux_diff_weapon);

				set_WeaponState(iItem, WPNSTATE_NULL);

				UTIL_SendWeaponAnim(iPlayer, WEAPON_ANIM_SHOOTB_SHOOT);
				emit_sound(iPlayer, CHAN_WEAPON, WEAPON_SOUNDS[1], VOL_NORM, ATTN_NORM, 0, PITCH_NORM);

				UTIL_CreateMuzzleFlash(iPlayer, WEAPON_MUZZLEFLASH_SPRITE[1], 0, 0.08, 255.0, 1, 0.04);
				UTIL_CreateMuzzleFlash(iPlayer, WEAPON_MUZZLEFLASH_SPRITE[1], 0, 0.08, 255.0, 2, 0.04);

				UTIL_CreateMuzzleFlash(iPlayer, WEAPON_MUZZLEFLASH_SPRITE[0], 0, 0.1, 255.0, 1, 0.04);
				UTIL_CreateMuzzleFlash(iPlayer, WEAPON_MUZZLEFLASH_SPRITE[0], 0, 0.1, 255.0, 2, 0.04);

				// Laserbeam
				new Float: vecOrigin[3]; UTIL_GetWeaponPosition(iAttacker, vecOrigin, 10.0, 11.0, -10.5);
				new Float: vecOrigin2[3]; UTIL_GetWeaponPosition(iAttacker, vecOrigin2, 10.0, -11.0, -10.5);
				new Float: vecEndPos[3]; fm_get_aim_origin(iAttacker, vecEndPos)

				UTIL_CreateBeamPoints(vecOrigin, vecEndPos, gl_iszModelIndex_Sprites[1], 0, 60, 10, 100, 0, 255, 255, 255, 255, 100);
				UTIL_CreateBeamPoints(vecOrigin2, vecEndPos, gl_iszModelIndex_Sprites[1], 0, 60, 10, 100, 0, 255, 255, 255, 255, 100);

				Create_Explosion(iItem);

				set_pdata_float(iPlayer, m_flNextAttack, WEAPON_ANIM_SHOOTB_SHOOT_TIME, linux_diff_player);
				set_pdata_float(iItem, m_flTimeWeaponIdle, WEAPON_ANIM_SHOOTB_SHOOT_TIME, linux_diff_weapon);

				// Reset
				set_pdata_int(iItem, m_iHitCount, 0, linux_diff_weapon);

				set_pev_string(iPlayer, pev_viewmodel2, gl_iszAllocString_ModelView)
				set_pev_string(iPlayer, pev_weaponmodel2, gl_iszAllocString_ModelPlayer);
			}
		}
	}

	return HAM_IGNORED;
}

public CWeapon__Holster_Post(iItem)
{
	if(!IsPdataSafe(iItem) || !IsCustomItem(iItem)) return;

	static iPlayer; iPlayer = get_pdata_cbase(iItem, m_pPlayer, linux_diff_weapon);

	set_WeaponState(iItem, WPNSTATE_NULL);

	set_pdata_float(iItem, m_flNextPrimaryAttack, 0.0, linux_diff_weapon);
	set_pdata_float(iItem, m_flNextSecondaryAttack, 0.0, linux_diff_weapon);
	set_pdata_float(iItem, m_flTimeWeaponIdle, 0.0, linux_diff_weapon);
	set_pdata_float(iPlayer, m_flNextAttack, 0.0, linux_diff_player);
}

public CEntity__TraceAttack_Pre(iVictim, iAttacker, Float: flDamage)
{
	if(!is_user_connected(iAttacker)) return;
	
	static iItem; iItem = get_pdata_cbase(iAttacker, 373, 5);
	if(!IsPdataSafe(iItem) || !IsCustomItem(iItem)) return;

	flDamage *= WEAPON_DAMAGE
	SetHamParamFloat(3, flDamage);
}

public CWeapon__AddToPlayer_Post(iItem, iPlayer)
{
	if(IsPdataSafe(iItem) && IsCustomItem(iItem)) UTIL_WeaponList(iPlayer, true);
	else if(!pev(iItem, pev_impulse)) UTIL_WeaponList(iPlayer, false);
}

public CMuzzleFlash__Think_Pre(const pSprite)
{
	if(pev_valid(pSprite) != 2 || !IsCustomMuzzle(pSprite)) return HAM_IGNORED;

	new Float: flFrame; pev(pSprite, pev_frame, flFrame);
	new Float: flNextThink; pev(pSprite, pev_fuser3, flNextThink);
	new iSpriteType = pev(pSprite, pev_iuser1);

	if(flFrame < get_pdata_float(pSprite, m_maxFrame, 4))
	{
		flFrame++;

		set_pev(pSprite, pev_frame, flFrame);
		set_pev(pSprite, pev_nextthink, get_gametime() + flNextThink);
		
		return HAM_SUPERCEDE;
	}
	else if(iSpriteType)
	{
		flFrame = 0.0;
		
		set_pev(pSprite, pev_frame, flFrame);
		set_pev(pSprite, pev_nextthink, get_gametime() + flNextThink);
		
		return HAM_SUPERCEDE;
	}

	set_pev(pSprite, pev_flags, FL_KILLME);
		
	return HAM_SUPERCEDE;
}

// ~ [ Fakemeta ] ~ //
public FM_Hook_UpdateClientData_Post(iPlayer, SendWeapons, CD_Handle)
{
	if(!is_user_alive(iPlayer)) return;

	static iItem; iItem = get_pdata_cbase(iPlayer, m_pActiveItem, linux_diff_player);
	if(!IsPdataSafe(iItem) || !IsCustomItem(iItem)) return;

	set_cd(CD_Handle, CD_flNextAttack, get_gametime() + 0.001);
}

public FM_Hook_SetModel_Pre(iEntity)
{
	static i, szClassName[32], iItem;
	pev(iEntity, pev_classname, szClassName, charsmax(szClassName));

	if(!equal(szClassName, "weaponbox")) return FMRES_IGNORED;

	for(i = 0; i < 6; i++)
	{
		iItem = get_pdata_cbase(iEntity, m_rgpPlayerItems_iItemBox + i, linux_diff_weapon);
		
		if(IsPdataSafe(iItem) && IsCustomItem(iItem))
		{
			engfunc(EngFunc_SetModel, iEntity, WEAPON_MODEL_WORLD);
			return FMRES_SUPERCEDE;
		}
	}

	return FMRES_IGNORED;
}

public FM_Hook_PlaybackEvent_Pre() return FMRES_SUPERCEDE;
public FM_Hook_TraceLine_Post(const Float: vecOrigin1[3], const Float: vecOrigin2[3], iFlags, iAttacker, iTrace)
{
	if(iFlags & IGNORE_MONSTERS) return FMRES_IGNORED;
	if(!is_user_alive(iAttacker)) return FMRES_IGNORED;

	static pHit; pHit = get_tr2(iTrace, TR_pHit);
	static Float: vecEndPos[3]; get_tr2(iTrace, TR_vecEndPos, vecEndPos);

	if(pHit > 0) if(pev(pHit, pev_solid) != SOLID_BSP) return FMRES_IGNORED;

	engfunc(EngFunc_MessageBegin, MSG_PAS, SVC_TEMPENTITY, vecEndPos, 0);
	write_byte(TE_WORLDDECAL);
	engfunc(EngFunc_WriteCoord, vecEndPos[0]);
	engfunc(EngFunc_WriteCoord, vecEndPos[1]);
	engfunc(EngFunc_WriteCoord, vecEndPos[2]);
	write_byte(random_num(41, 45));
	message_end();

	UTIL_CreateStreakSplash(vecEndPos, 102, random_num(32, 64), 3, 100);

	UTIL_CreateExplosion(vecEndPos, gl_iszModelIndex_SmokePuff, 5, 60, 2|4|8);
	
	// Tracer
	new Float: vecOrigin[3]; UTIL_GetWeaponPosition(iAttacker, vecOrigin, 10.0, 11.0, -10.5);
	new Float: vecOrigin2[3]; UTIL_GetWeaponPosition(iAttacker, vecOrigin2, 10.0, -11.0, -10.5);
	
	new Float: vecVelocity[3];
	vecVelocity[0] = vecEndPos[0] - vecOrigin[0];
	vecVelocity[1] = vecEndPos[1] - vecOrigin[1];
	vecVelocity[2] = vecEndPos[2] - vecOrigin[2];

	xs_vec_normalize(vecVelocity, vecVelocity);
	xs_vec_mul_scalar(vecVelocity, 3500.0, vecVelocity);

	UTIL_CreateUserTracers(vecOrigin, vecVelocity, 5, 7, 10);
	UTIL_CreateUserTracers(vecOrigin2, vecVelocity, 5, 7, 10);

	return FMRES_IGNORED;
}

// ~ [ Others ] ~ //
public fm_ham_hook(bool: bEnabled)
{
	if(bEnabled)
	{
		EnableHamForward(gl_HamHook_TraceAttack[0]);
		EnableHamForward(gl_HamHook_TraceAttack[1]);
		EnableHamForward(gl_HamHook_TraceAttack[2]);
		EnableHamForward(gl_HamHook_TraceAttack[3]);
	}
	else 
	{
		DisableHamForward(gl_HamHook_TraceAttack[0]);
		DisableHamForward(gl_HamHook_TraceAttack[1]);
		DisableHamForward(gl_HamHook_TraceAttack[2]);
		DisableHamForward(gl_HamHook_TraceAttack[3]);
	}
}

public Create_Explosion(iItem)
{
	static iAttacker; iAttacker = get_pdata_cbase(iItem, m_pPlayer, linux_diff_weapon);
	
	new Float: vecEnd[3]; fm_get_aim_origin(iAttacker, vecEnd)
	new Float: vecViewAngle[3]; pev(iAttacker, pev_v_angle, vecViewAngle);
	new Float: vecForward[3]; angle_vector(vecViewAngle, ANGLEVECTOR_FORWARD, vecForward);

	UTIL_CreateExplosion(vecEnd, gl_iszModelIndex_Sprites[0], 10, 30, 0);
	UTIL_CreateStreakSplash(vecEnd, 11, 500, 5, 450);

	new iVictim = FM_NULLENT;
	while((iVictim = engfunc(EngFunc_FindEntityInSphere, iVictim, vecEnd, WEAPON_EXP_RADIUS)) > 0)
	{
		if(pev(iVictim, pev_takedamage) == DAMAGE_NO) 
						continue;
		if(is_user_alive(iVictim))
		{
			if(iVictim == iAttacker || get_user_team(iVictim) != 1)
			continue;
		}
		else if(pev(iVictim, pev_solid) == SOLID_BSP)
		{
			if(pev(iVictim, pev_spawnflags) & SF_BREAK_TRIGGER_ONLY)
				continue;
		}

		if(is_user_alive(iVictim)) emit_sound(iVictim, CHAN_ITEM, WEAPON_SOUNDS[2], VOL_NORM, ATTN_NORM, 0, PITCH_NORM)

		ExecuteHamB(Ham_TakeDamage, iVictim, iAttacker, iAttacker, WEAPON_EXP_DMG, WEAPON_EXP_DMGTYPE);

		UTIL_Knockback(iVictim, vecForward, WEAPON_EXP_KNOCKBACK);
	}
}

// ~ [ Stocks ] ~ //
stock UTIL_SendWeaponAnim(const iPlayer, const iAnim)
{
	set_pev(iPlayer, pev_weaponanim, iAnim);

	message_begin(MSG_ONE, SVC_WEAPONANIM, _, iPlayer);
	write_byte(iAnim);
	write_byte(0);
	message_end();
}

stock UTIL_CreateMuzzleFlash(const pPlayer, const szMuzzleSprite[], const iMuzzleLoop, const Float: flScale, const Float: flBrightness, const iAttachment, Float: flNextThink)
{
	if(global_get(glb_maxEntities) - engfunc(EngFunc_NumberOfEntities) < 100) return FM_NULLENT;
		
	static pSprite, iszAllocStringCached;

	if(iszAllocStringCached || (iszAllocStringCached = engfunc(EngFunc_AllocString, "env_sprite")))
		pSprite = engfunc(EngFunc_CreateNamedEntity, iszAllocStringCached);
		
	if(pev_valid(pSprite) != 2) return FM_NULLENT;
		
	set_pev(pSprite, pev_model, szMuzzleSprite);
	set_pev(pSprite, pev_spawnflags, SF_SPRITE_ONCE);
		
	set_pev(pSprite, pev_classname, WEAPON_MUZZLEFLASH_CLASSNAME);
	set_pev(pSprite, pev_impulse, gl_iszAllocString_MuzzleKey);
	set_pev(pSprite, pev_owner, pPlayer);
	set_pev(pSprite, pev_fuser3, flNextThink);
	set_pev(pSprite, pev_iuser1, iMuzzleLoop);
	set_pev(pSprite, pev_aiment, pPlayer);
	set_pev(pSprite, pev_body, iAttachment);

	set_pev(pSprite, pev_rendermode, kRenderTransAdd);
	set_pev(pSprite, pev_renderamt, flBrightness);

	set_pev(pSprite, pev_scale, flScale);
		
	dllfunc(DLLFunc_Spawn, pSprite)

	return pSprite;
}

stock UTIL_DropWeapon(const iPlayer, const iSlot)
{
	static iEntity, iNext, szWeaponName[32];
	iEntity = get_pdata_cbase(iPlayer, m_rpgPlayerItems + iSlot, linux_diff_player);

	if(iEntity > 0)
	{	   
	do 
	{
		iNext = get_pdata_cbase(iEntity, m_pNext, linux_diff_weapon);
		if(get_weaponname(get_pdata_int(iEntity, m_iId, linux_diff_weapon), szWeaponName, charsmax(szWeaponName)))
		engclient_cmd(iPlayer, "drop", szWeaponName);
	} 
		
	while((iEntity = iNext) > 0);
	}
}

stock UTIL_WeaponList(const iPlayer, bool: bEnabled)
{
	message_begin(MSG_ONE, gl_iMsgID_Weaponlist, _, iPlayer);
	write_string(bEnabled ? WEAPON_WEAPONLIST : WEAPON_REFERENCE);
	write_byte(iWeaponList[0]);
	write_byte(bEnabled ? WEAPON_MAX_CLIP : iWeaponList[1]);
	write_byte(iWeaponList[2]);
	write_byte(iWeaponList[3]);
	write_byte(iWeaponList[4]);
	write_byte(iWeaponList[5]);
	write_byte(iWeaponList[6]);
	write_byte(iWeaponList[7]);
	message_end();
}

stock UTIL_Knockback(iPlayer, Float: vecDirection[3], Float:flKnockBack)
{
	static Float:vecVelocity[3]; pev(iPlayer, pev_velocity, vecVelocity);

	if(pev(iPlayer, pev_flags) & FL_DUCKING) flKnockBack *= 0.7;

	vecVelocity[0] = vecDirection[0] * 50.0 * flKnockBack;
	vecVelocity[1] = vecDirection[1] * 50.0 * flKnockBack;
	vecVelocity[2] = 50.0;

	set_pev(iPlayer, pev_velocity, vecVelocity);
}

stock UTIL_GetWeaponPosition(iPlayer, Float: vecOrigin[3], Float: flAddForward = 0.0, Float: flAddRight = 0.0, Float: flAddUp = 0.0)
{
	static Float: vecAngles[3], Float: vecViewOfs[3];
	static Float: vecDirShooting[3], Float: vecRight[3], Float: vecUp[3];
	
	pev(iPlayer, pev_origin, vecOrigin);
	pev(iPlayer, pev_view_ofs, vecViewOfs);
	
	xs_vec_add(vecOrigin, vecViewOfs, vecOrigin);
	pev(iPlayer, pev_v_angle, vecAngles);
	engfunc(EngFunc_MakeVectors, vecAngles);
	
	global_get(glb_v_forward, vecDirShooting);
	global_get(glb_v_right, vecRight);
	global_get(glb_v_up, vecUp);
	
	xs_vec_mul_scalar(vecDirShooting, flAddForward, vecDirShooting);
	xs_vec_mul_scalar(vecRight, flAddRight, vecRight);
	xs_vec_mul_scalar(vecUp, flAddUp, vecUp);
	
	vecOrigin[0] = vecOrigin[0] + vecDirShooting[0] + vecRight[0] + vecUp[0];
	vecOrigin[1] = vecOrigin[1] + vecDirShooting[1] + vecRight[1] + vecUp[1];
	vecOrigin[2] = vecOrigin[2] + vecDirShooting[2] + vecRight[2] + vecUp[2];
}


stock UTIL_CreateExplosion(Float: iVecPos[3], iszModelIndex, iScale, iFramerate, iFlags)
{
	// https://github.com/baso88/SC_AngelScript/wiki/TE_EXPLOSION
	message_begin(MSG_BROADCAST, SVC_TEMPENTITY)
	write_byte(TE_EXPLOSION); // TE
	engfunc(EngFunc_WriteCoord, iVecPos[0]); // Position X
	engfunc(EngFunc_WriteCoord, iVecPos[1]); // Position Y
	engfunc(EngFunc_WriteCoord, iVecPos[2]); // Position Z
	write_short(iszModelIndex); // Model Index
	write_byte(iScale); // Scale
	write_byte(iFramerate); // Framerate
	write_byte(iFlags); // Flags
	message_end();
}

stock UTIL_CreateStreakSplash(Float: iVecPos[3], iColor, iAmount, iSpeed, iSpeedNoise)
{
	// https://github.com/baso88/SC_AngelScript/wiki/TE_STREAK_SPLASH
	message_begin(MSG_BROADCAST, SVC_TEMPENTITY);
	write_byte(TE_STREAK_SPLASH); // TE
	engfunc(EngFunc_WriteCoord, iVecPos[0]); // Position X
	engfunc(EngFunc_WriteCoord, iVecPos[1]); // Position Y
	engfunc(EngFunc_WriteCoord, iVecPos[2]); // Position Z
	write_coord(random_num(-20, 20)); // Direction X
	write_coord(random_num(-20, 20)); // Direction Y
	write_coord(random_num(-20, 20)); // Direction Z
	write_byte(iColor); // Color
	write_short(iAmount); // Amount
	write_short(iSpeed); // Speed
	write_short(iSpeedNoise); // Speed Noise
	message_end();
}

stock UTIL_CreateBeamPoints(Float: iVecPosStart[3], Float: iVecPosEnd[3], iszModelIndex, iFrameStart, iFrameRate, iLife, iWidth, iNoise, iRed, iGreen, iBlue, iAlpha, iScroll)
{
	// https://github.com/baso88/SC_AngelScript/wiki/TE_BEAMPOINTS
	message_begin(MSG_BROADCAST, SVC_TEMPENTITY)
	write_byte(TE_BEAMPOINTS); // TE
	engfunc(EngFunc_WriteCoord, iVecPosStart[0]); // Position X
	engfunc(EngFunc_WriteCoord, iVecPosStart[1]); // Position Y
	engfunc(EngFunc_WriteCoord, iVecPosStart[2]); // Position Z
	engfunc(EngFunc_WriteCoord, iVecPosEnd[0]); // Position X
	engfunc(EngFunc_WriteCoord, iVecPosEnd[1]); // Position Y
	engfunc(EngFunc_WriteCoord, iVecPosEnd[2]); // Position Z
	write_short(iszModelIndex); // Model Index
	write_byte(iFrameStart); // Framestart
	write_byte(iFrameRate); // Framerate
	write_byte(iLife); // Life
	write_byte(iWidth); // Width
	write_byte(iNoise); // Noise
	write_byte(iRed); // Red
	write_byte(iGreen); // Green
	write_byte(iBlue); // Blue
	write_byte(iAlpha); // Alpha
	write_byte(iScroll); // Scroll
	message_end();
}

stock UTIL_CreateUserTracers(Float: vecOrigin[3], Float: vecVelocity[3], iLife, iColor, iLenght)
{
	// https://github.com/baso88/SC_AngelScript/wiki/TE_USERTRACER
	message_begin(MSG_BROADCAST, SVC_TEMPENTITY);
	write_byte(TE_USERTRACER); // TE
	engfunc(EngFunc_WriteCoord, vecOrigin[0]); // Position X
	engfunc(EngFunc_WriteCoord, vecOrigin[1]); // Position Y
	engfunc(EngFunc_WriteCoord, vecOrigin[2]); // Position Z
	engfunc(EngFunc_WriteCoord, vecVelocity[0]); // Velocity X
	engfunc(EngFunc_WriteCoord, vecVelocity[1]); // Velocity Y
	engfunc(EngFunc_WriteCoord, vecVelocity[2]); // Velocity Z
	write_byte(iLife); // Life
	write_byte(iColor); // Color
	write_byte(iLenght); // Lenght
	message_end();
}
