
]]

local RunService = game:GetService("RunService")
local IS_SERVER = RunService:IsServer()
print("BLACKOUT: script started. Running as " .. (IS_SERVER and "SERVER" or "CLIENT"))

-- =====================================================================
-- SHARED CONFIG
-- =====================================================================
local CFG = {
	MIN_PLAYERS = 1,        -- real players needed to start a match
	BOT_FILL = 8,           -- total combatants when bots are needed
	HUMANS_FOR_NO_BOTS = 8, -- bots are only used as filler: with this many real players queued, no bots are added
	VOTE_TIME = 60,
	COUNTDOWN = 5,
	END_SCREEN_TIME = 14,
	RESPAWN_TIME = 5,
	SPAWN_PROTECT = 2.5,
	REGEN_DELAY = 5,
	REGEN_PER_SEC = 45,
	BOT_DAMAGE = 0.6,      -- bots deal this fraction of weapon damage
	BOT_XP_MULT = 0.4,      -- XP from bot kills is reduced (anti-farming)
	XP_KILL = 100,
	XP_HEADSHOT = 25,
	XP_ASSIST = 40,
	XP_PARTICIPATE = 100,
	XP_WIN = 250,
	AUTOSAVE = 90,
	HUB_POS = Vector3.new(0, 800, 0),
	MELEE_RANGE = 7,
	MELEE_DAMAGE = 100,
	MELEE_COOLDOWN = 0.9,
	BASE_WALK = 16,
	SPRINT_MULT = 1.4,
}

local LEVEL_CAP = 100
local function xpForLevel(l) return 250 + l * 30 end
local function levelFromXp(xp)
	local lvl, rem = 1, math.max(0, xp)
	while lvl < LEVEL_CAP and rem >= xpForLevel(lvl) do
		rem -= xpForLevel(lvl)
		lvl += 1
	end
	return lvl, rem, xpForLevel(lvl)
end

local function shuffle(t)
	for i = #t, 2, -1 do
		local j = math.random(1, i)
		t[i], t[j] = t[j], t[i]
	end
	return t
end

-- =====================================================================
-- WEAPONS  (original designs inspired by futuristic-FPS archetypes)
-- =====================================================================
local function RGB(r, g, b) return Color3.fromRGB(r, g, b) end

local WEAPONS = {
	VX44 = { id = "VX44", name = "AK-47", class = "ASSAULT RIFLE", slot = "primary", unlock = 1,
		desc = "Reliable full-auto all-rounder.",
		dmg = 25, dmgFar = 19, rNear = 35, rFar = 95, rpm = 680, auto = true, mag = 30, reserve = 120,
		reload = 2.3, reloadEmpty = 2.9, hip = 2.4, ads = 0.3, moveSpread = 1.6, headMult = 1.5,
		kickP = 0.9, kickY = 0.35, speed = 1.0, adsTime = 0.22, zoom = 0.68, switch = 0.4,
		color = RGB(255, 140, 30), body = RGB(38, 42, 54), len = 2.8, stats = { 0.5, 0.7, 0.6, 0.6 } },
	KR9 = { id = "KR9", name = "M16A4", class = "BURST RIFLE", slot = "primary", unlock = 4,
		desc = "3-round burst. Precise and deadly.",
		dmg = 31, dmgFar = 24, rNear = 40, rFar = 110, rpm = 950, burst = 3, burstDelay = 0.26, mag = 30, reserve = 120,
		reload = 2.2, reloadEmpty = 2.8, hip = 2.0, ads = 0.2, moveSpread = 1.4, headMult = 1.5,
		kickP = 0.8, kickY = 0.2, speed = 1.0, adsTime = 0.24, zoom = 0.75, switch = 0.4,
		color = RGB(0, 210, 255), body = RGB(44, 48, 60), len = 2.9, stats = { 0.6, 0.55, 0.7, 0.6 } },
	HEX12 = { id = "HEX12", name = "MP5", class = "SMG", slot = "primary", unlock = 2,
		desc = "Fast and mobile. Great up close.",
		dmg = 19, dmgFar = 12, rNear = 20, rFar = 55, rpm = 900, auto = true, mag = 32, reserve = 128,
		reload = 1.9, reloadEmpty = 2.4, hip = 3.4, ads = 0.6, moveSpread = 1.0, headMult = 1.4,
		kickP = 0.6, kickY = 0.5, speed = 1.08, adsTime = 0.16, zoom = 0.85, switch = 0.3,
		color = RGB(190, 90, 255), body = RGB(34, 36, 48), len = 2.2, stats = { 0.35, 0.95, 0.35, 0.9 } },
	BRK8 = { id = "BRK8", name = "R870", class = "SHOTGUN", slot = "primary", unlock = 6,
		desc = "Pump-action pellet blaster. Devastating close.",
		dmg = 13, dmgFar = 3, rNear = 12, rFar = 38, rpm = 75, pellets = 9, mag = 6, reserve = 36,
		reload = 2.6, reloadEmpty = 2.6, hip = 5.0, ads = 3.2, moveSpread = 1.0, headMult = 1.3,
		kickP = 3.5, kickY = 0.6, speed = 0.97, adsTime = 0.24, zoom = 0.86, switch = 0.45,
		color = RGB(255, 70, 70), body = RGB(52, 44, 40), len = 3.0, stats = { 0.95, 0.15, 0.2, 0.7 } },
	THD = { id = "THD", name = "M249 SAW", class = "LIGHT MACHINE GUN", slot = "primary", unlock = 8,
		desc = "Huge magazine. Heavy and loud.",
		dmg = 24, dmgFar = 20, rNear = 40, rFar = 100, rpm = 600, auto = true, mag = 100, reserve = 200,
		reload = 5.0, reloadEmpty = 5.6, hip = 3.2, ads = 0.9, moveSpread = 2.2, headMult = 1.4,
		kickP = 1.1, kickY = 0.6, speed = 0.88, adsTime = 0.32, zoom = 0.8, switch = 0.6,
		color = RGB(255, 200, 0), body = RGB(46, 46, 52), len = 3.3, stats = { 0.55, 0.6, 0.7, 0.2 } },
	KRAIT = { id = "KRAIT", name = "SVD DRAGUNOV", class = "MARKSMAN RIFLE", slot = "primary", unlock = 12,
		desc = "Semi-auto. Two body shots at range.",
		dmg = 52, dmgFar = 42, rNear = 60, rFar = 150, rpm = 280, auto = false, mag = 15, reserve = 60,
		reload = 2.4, reloadEmpty = 3.0, hip = 3.0, ads = 0.1, moveSpread = 1.8, headMult = 1.8,
		kickP = 1.8, kickY = 0.2, speed = 0.96, adsTime = 0.28, zoom = 0.5, switch = 0.45,
		color = RGB(60, 255, 150), body = RGB(40, 46, 52), len = 3.2, stats = { 0.8, 0.35, 0.85, 0.5 } },
	LSX = { id = "LSX", name = "L96 SNIPER", class = "SNIPER RIFLE", slot = "primary", unlock = 10, scope = true,
		desc = "Bolt-action. One shot, one kill.",
		dmg = 100, dmgFar = 100, rNear = 300, rFar = 300, rpm = 48, auto = false, mag = 5, reserve = 20,
		reload = 3.2, reloadEmpty = 3.6, hip = 8.0, ads = 0.0, moveSpread = 3.0, headMult = 2.0,
		kickP = 4.0, kickY = 0.4, speed = 0.92, adsTime = 0.35, zoom = 0.22, switch = 0.6,
		color = RGB(255, 90, 200), body = RGB(36, 40, 46), len = 3.8, stats = { 1.0, 0.1, 1.0, 0.3 } },
	MK2 = { id = "MK2", name = "M1911", class = "PISTOL", slot = "secondary", unlock = 1,
		desc = "Accurate semi-auto sidearm.",
		dmg = 28, dmgFar = 18, rNear = 25, rFar = 70, rpm = 420, auto = false, mag = 12, reserve = 48,
		reload = 1.6, reloadEmpty = 2.0, hip = 2.0, ads = 0.4, moveSpread = 1.0, headMult = 1.5,
		kickP = 1.2, kickY = 0.2, speed = 1.05, adsTime = 0.14, zoom = 0.9, switch = 0.25,
		color = RGB(255, 140, 30), body = RGB(50, 52, 60), len = 1.5, stats = { 0.4, 0.5, 0.4, 0.9 } },
	VPR9 = { id = "VPR9", name = "MAC-10", class = "MACHINE PISTOL", slot = "secondary", unlock = 5,
		desc = "Full-auto pistol. Chews through ammo.",
		dmg = 14, dmgFar = 9, rNear = 15, rFar = 45, rpm = 1000, auto = true, mag = 24, reserve = 96,
		reload = 1.7, reloadEmpty = 2.1, hip = 3.6, ads = 1.4, moveSpread = 1.0, headMult = 1.3,
		kickP = 0.5, kickY = 0.6, speed = 1.06, adsTime = 0.12, zoom = 0.92, switch = 0.25,
		color = RGB(190, 90, 255), body = RGB(44, 40, 52), len = 1.6, stats = { 0.2, 0.9, 0.2, 0.95 } },
}
-- premium weapons (sold in the STORE; unlock = 99 keeps them locked by level)
WEAPONS.ARC9 = { id = "ARC9", name = "ARC-9 VOLTAGE", class = "SMG   PREMIUM", slot = "primary", unlock = 99, premium = true,
	desc = "Arc-tech SMG. Insane fire rate and a huge magazine.",
	dmg = 17, dmgFar = 11, rNear = 22, rFar = 60, rpm = 1100, auto = true, mag = 40, reserve = 160,
	reload = 1.8, reloadEmpty = 2.3, hip = 3.0, ads = 0.5, moveSpread = 0.9, headMult = 1.4,
	kickP = 0.5, kickY = 0.45, speed = 1.1, adsTime = 0.15, zoom = 0.85, switch = 0.28,
	color = RGB(0, 255, 240), body = RGB(28, 34, 50), len = 2.1, stats = { 0.3, 0.9, 0.4, 0.95 } }
WEAPONS.REAPER = { id = "REAPER", name = "REAPER-X", class = "HEAVY RIFLE   PREMIUM", slot = "primary", unlock = 99, premium = true,
	desc = "Hard-hitting full-auto rifle with long reach.",
	dmg = 34, dmgFar = 28, rNear = 45, rFar = 120, rpm = 520, auto = true, mag = 24, reserve = 96,
	reload = 2.6, reloadEmpty = 3.1, hip = 2.6, ads = 0.25, moveSpread = 1.7, headMult = 1.6,
	kickP = 1.3, kickY = 0.4, speed = 0.97, adsTime = 0.26, zoom = 0.7, switch = 0.45,
	color = RGB(255, 50, 60), body = RGB(38, 28, 32), len = 3.0, stats = { 0.75, 0.7, 0.5, 0.55 } }
WEAPONS.WRAITH = { id = "WRAITH", name = "WRAITH-5", class = "4-ROUND BURST   PREMIUM", slot = "primary", unlock = 99, premium = true,
	desc = "Silent-profile burst rifle. Surgical at any range.",
	dmg = 29, dmgFar = 24, rNear = 42, rFar = 115, rpm = 1000, burst = 4, burstDelay = 0.3, mag = 28, reserve = 112,
	reload = 2.2, reloadEmpty = 2.8, hip = 2.1, ads = 0.15, moveSpread = 1.4, headMult = 1.6,
	kickP = 0.7, kickY = 0.2, speed = 1.0, adsTime = 0.22, zoom = 0.72, switch = 0.38,
	color = RGB(180, 90, 255), body = RGB(34, 30, 46), len = 2.9, stats = { 0.7, 0.65, 0.8, 0.6 } }
WEAPONS.TALON = { id = "TALON", name = "TALON .50", class = "HAND CANNON   PREMIUM", slot = "secondary", unlock = 99, premium = true,
	desc = "Heavy sidearm. Two shots to the chest.",
	dmg = 55, dmgFar = 42, rNear = 25, rFar = 70, rpm = 240, auto = false, mag = 8, reserve = 40,
	reload = 1.8, reloadEmpty = 2.2, hip = 1.6, ads = 0.1, moveSpread = 1.0, headMult = 1.8,
	kickP = 2.4, kickY = 0.3, speed = 1.05, adsTime = 0.13, zoom = 0.9, switch = 0.25,
	color = RGB(255, 200, 60), body = RGB(44, 40, 36), len = 1.2, stats = { 0.85, 0.4, 0.3, 0.9 } }
WEAPONS.FNG = { id = "FNG", name = "UZI", class = "SMG", slot = "primary", unlock = 3,
	desc = "Compact submachine gun. Fast handling, quick kills up close.",
	dmg = 21, dmgFar = 13, rNear = 18, rFar = 50, rpm = 820, auto = true, mag = 30, reserve = 150,
	reload = 1.7, reloadEmpty = 2.2, hip = 2.8, ads = 0.5, moveSpread = 0.9, headMult = 1.4,
	kickP = 0.55, kickY = 0.4, speed = 1.1, adsTime = 0.14, zoom = 0.86, switch = 0.28,
	color = RGB(255, 120, 60), body = RGB(40, 36, 36), len = 1.9, stats = { 0.4, 0.85, 0.3, 0.95 } }
WEAPONS.QSR = { id = "QSR", name = "KAR98K", class = "SNIPER RIFLE   QUICKSCOPE", slot = "primary", unlock = 7, scope = true,
	desc = "Light bolt-action. Fast aim time. A headshot is an instant kill.",
	dmg = 82, dmgFar = 72, rNear = 120, rFar = 260, rpm = 70, auto = false, mag = 6, reserve = 30,
	reload = 2.6, reloadEmpty = 3.0, hip = 6.0, ads = 0.0, moveSpread = 2.2, headMult = 2.2,
	kickP = 3.0, kickY = 0.3, speed = 1.0, adsTime = 0.2, zoom = 0.3, switch = 0.4,
	color = RGB(255, 200, 80), body = RGB(48, 44, 36), len = 3.4, stats = { 0.85, 0.2, 0.9, 0.6 } }
WEAPONS.HAWK = { id = "HAWK", name = "M14", class = "SNIPER RIFLE   SEMI-AUTO", slot = "primary", unlock = 14, scope = true,
	desc = "Semi-auto sniper. Two quick body shots, rapid follow-ups.",
	dmg = 62, dmgFar = 55, rNear = 100, rFar = 240, rpm = 170, auto = false, mag = 10, reserve = 40,
	reload = 2.9, reloadEmpty = 3.4, hip = 6.5, ads = 0.0, moveSpread = 2.6, headMult = 1.9,
	kickP = 2.4, kickY = 0.3, speed = 0.94, adsTime = 0.3, zoom = 0.26, switch = 0.5,
	color = RGB(110, 255, 120), body = RGB(36, 44, 40), len = 3.6, stats = { 0.7, 0.4, 0.85, 0.45 } }
WEAPONS.RVL = { id = "RVL", name = ".357 MAGNUM", class = "REVOLVER", slot = "secondary", unlock = 9,
	desc = "Six-shooter. Heavy hits, slow reload.",
	dmg = 48, dmgFar = 34, rNear = 22, rFar = 60, rpm = 190, auto = false, mag = 6, reserve = 36,
	reload = 2.4, reloadEmpty = 2.4, hip = 1.8, ads = 0.1, moveSpread = 1.0, headMult = 1.8,
	kickP = 2.0, kickY = 0.3, speed = 1.04, adsTime = 0.14, zoom = 0.9, switch = 0.28,
	color = RGB(255, 170, 70), body = RGB(60, 52, 46), len = 1.35, stats = { 0.75, 0.3, 0.3, 0.85 } }
WEAPONS.ECLIPSE = { id = "ECLIPSE", name = "ECLIPSE RAILGUN", class = "RAIL SNIPER   PREMIUM", slot = "primary", unlock = 99, premium = true, scope = true,
	desc = "Charged rail sniper. One shot, any range, no drop-off.",
	dmg = 100, dmgFar = 100, rNear = 400, rFar = 400, rpm = 55, auto = false, mag = 4, reserve = 24,
	reload = 2.8, reloadEmpty = 3.2, hip = 8.0, ads = 0.0, moveSpread = 3.0, headMult = 2.0,
	kickP = 3.2, kickY = 0.2, speed = 0.93, adsTime = 0.28, zoom = 0.2, switch = 0.55,
	color = RGB(80, 220, 255), body = RGB(24, 30, 44), len = 4.0, stats = { 1.0, 0.15, 1.0, 0.4 } }
WEAPONS.BANSHEE = { id = "BANSHEE", name = "BANSHEE-AS", class = "AUTO SNIPER   PREMIUM", slot = "primary", unlock = 99, premium = true, scope = true,
	desc = "Full-auto sniper. Hold the trigger and sweep targets down.",
	dmg = 36, dmgFar = 30, rNear = 90, rFar = 220, rpm = 330, auto = true, mag = 20, reserve = 80,
	reload = 3.4, reloadEmpty = 3.9, hip = 7.0, ads = 0.2, moveSpread = 2.6, headMult = 1.8,
	kickP = 1.7, kickY = 0.5, speed = 0.9, adsTime = 0.32, zoom = 0.3, switch = 0.55,
	color = RGB(255, 60, 160), body = RGB(40, 28, 44), len = 3.9, stats = { 0.6, 0.65, 0.85, 0.35 } }
WEAPONS.RG500 = { id = "RG500", name = "SMG RG 500", class = "SUBMACHINE GUN", slot = "primary", unlock = 1,
	desc = "Versatile machine pistol. Fast handling, a huge 40-round mag and a vertical kick that rewards a steady pull-down.",
	dmg = 27, dmgFar = 17, rNear = 22, rFar = 62, rpm = 909, auto = true, mag = 40, reserve = 160,
	reload = 1.8, reloadEmpty = 2.4, hip = 2.4, ads = 0.5, moveSpread = 0.85, headMult = 1.4,
	kickP = 0.8, kickY = 0.55, speed = 1.08, adsTime = 0.2, zoom = 0.85, switch = 0.27,
	color = RGB(188, 226, 54), body = RGB(52, 56, 62), len = 2.5, stats = { 0.45, 0.92, 0.4, 0.92 } }
local PRIMARY_ORDER = { "VX44", "RG500", "HEX12", "KR9", "BRK8", "THD", "LSX", "KRAIT", "FNG", "QSR", "HAWK" }
local SECONDARY_ORDER = { "MK2", "VPR9", "RVL" }
local PREMIUM_PRIMARY = { "ARC9", "REAPER", "WRAITH", "ECLIPSE", "BANSHEE" }
local PREMIUM_SECONDARY = { "TALON" }
local GUNGAME_LADDER = { "MK2", "VPR9", "HEX12", "BRK8", "KR9", "VX44", "THD", "KRAIT", "LSX" }

-- =====================================================================
-- SOUNDS  (Roblox built-in client sounds, layered + effect-shaped so
-- every gun sounds different). Paste official Creator Store sound IDs
-- into CUSTOM_SOUND_IDS (e.g. "rbxassetid://123456") to override a gun.
-- =====================================================================
local SND = {
	snap = "rbxasset://sounds/snap.mp3",
	collide = "rbxasset://sounds/collide.mp3",
	bass = "rbxasset://sounds/bass.mp3",
	hit = "rbxasset://sounds/hit.mp3",
	ping = "rbxasset://sounds/electronicpingshort.mp3",
	click = "rbxasset://sounds/button.mp3",
	click2 = "rbxasset://sounds/switch.mp3",
	clickfast = "rbxasset://sounds/clickfast.mp3",
	flash = "rbxasset://sounds/flashbulb.mp3",
	splat = "rbxasset://sounds/splat.mp3",
	kerplunk = "rbxasset://sounds/Kerplunk.mp3",
	lunge = "rbxasset://sounds/swordlunge.mp3",
	slash = "rbxasset://sounds/swordslash.mp3",
	unsheath = "rbxasset://sounds/unsheath.mp3",
	victory = "rbxasset://sounds/victory.mp3",
	oof = "rbxasset://sounds/uuhhh.mp3",
}
-- layer: p = sound key, s = playback speed, v = volume, len = cut after N sec
local SFX_PROFILE = {
	VX44 = { { "snap", 0.7, 1.0, 0.25 }, { "collide", 2.4, 0.5, 0.09 }, dist = 0.25 },
	KR9 = { { "snap", 1.0, 1.0, 0.2 }, { "flash", 1.6, 0.5, 0.08 }, dist = 0.1, eq = true },
	HEX12 = { { "snap", 1.5, 0.8, 0.15 }, { "clickfast", 3.0, 0.5, 0.05 } },
	RG500 = { { "snap", 1.3, 0.9, 0.14 }, { "clickfast", 2.8, 0.55, 0.05 }, { "collide", 2.4, 0.35, 0.05 } },
	BRK8 = { { "collide", 1.2, 1.0, 0.3 }, { "bass", 1.4, 0.9, 0.25 }, { "snap", 0.5, 1.0, 0.3 }, dist = 0.45 },
	THD = { { "collide", 1.7, 0.7, 0.12 }, { "bass", 1.8, 0.8, 0.1 }, { "snap", 0.6, 0.9, 0.2 }, dist = 0.35 },
	KRAIT = { { "snap", 0.6, 1.0, 0.3 }, { "ping", 0.7, 0.4, 0.15 }, reverb = true },
	LSX = { { "collide", 0.8, 1.0, 0.6 }, { "bass", 0.9, 1.0, 0.5 }, { "snap", 0.4, 1.0, 0.4 }, reverb = true, dist = 0.2 },
	FNG = { { "snap", 1.6, 0.8, 0.12 }, { "clickfast", 2.6, 0.5, 0.05 } },
	QSR = { { "collide", 1.0, 0.9, 0.4 }, { "snap", 0.5, 1.0, 0.3 }, reverb = true },
	HAWK = { { "snap", 0.65, 1.0, 0.3 }, { "collide", 1.3, 0.7, 0.2 }, { "ping", 0.9, 0.3, 0.1 }, reverb = true },
	RVL = { { "collide", 1.3, 1.0, 0.25 }, { "snap", 0.8, 1.0, 0.2 }, reverb = true },
	ECLIPSE = { { "flash", 0.7, 1.0, 0.4 }, { "bass", 1.1, 0.9, 0.4 }, { "ping", 0.5, 0.6, 0.3 }, reverb = true, dist = 0.1 },
	BANSHEE = { { "snap", 0.8, 1.0, 0.2 }, { "collide", 1.5, 0.6, 0.1 }, { "ping", 1.1, 0.3, 0.08 }, reverb = true },
	MK2 = { { "snap", 1.1, 1.0, 0.18 }, { "flash", 2.0, 0.4, 0.06 } },
	VPR9 = { { "snap", 1.7, 0.9, 0.12 }, { "clickfast", 2.5, 0.5, 0.05 } },
	ARC9 = { { "snap", 1.9, 0.8, 0.1 }, { "flash", 2.4, 0.5, 0.06 } },
	REAPER = { { "snap", 0.55, 1.0, 0.25 }, { "collide", 1.9, 0.6, 0.1 }, dist = 0.3 },
	WRAITH = { { "snap", 1.25, 0.8, 0.16 }, { "flash", 1.8, 0.4, 0.06 }, eq = true },
	TALON = { { "collide", 1.0, 1.0, 0.3 }, { "snap", 0.7, 1.0, 0.25 }, reverb = true },
}
-- ANNOUNCER VOICE LINES: paste your own licensed/recorded audio asset IDs here ("rbxassetid://123").
-- Leave a list empty to use the built-in synthesized stingers instead.
local ANNOUNCER_IDS = {
	KILL = {}, DOUBLE = {}, TRIPLE = {}, MULTI = {}, HEADSHOT = {}, START = {}, MIN1 = {}, SEC30 = {}, UAV = {}, LOWHP = {}, DOWN = {}, WIN = {}, LOSE = {},
}
-- MUSIC: paste licensed track IDs from the Creator Store (Audio) to replace the built-in synth score.
-- e.g. menu = { "rbxassetid://123456" }. Leave a list empty to use the built-in dark synth music for that mode.
local MUSIC_IDS = { menu = {}, battle = {} }
local CUSTOM_SOUND_IDS = {
	-- VX44 = "rbxassetid://0",
}

-- =====================================================================
-- MODES / MAPS / LIGHTING
-- =====================================================================
local MODES = {
	TDM = { id = "TDM", name = "TEAM DEATHMATCH", teams = true, limit = 50, time = 300, desc = "First team to 50 eliminations." },
	FFA = { id = "FFA", name = "FREE-FOR-ALL", teams = false, limit = 25, time = 300, desc = "Every soldier for themselves. 25 kills." },
	GUN = { id = "GUN", name = "GUN GAME", teams = false, limit = #GUNGAME_LADDER, time = 420, desc = "Each kill upgrades your weapon. Finish the ladder." },
	FREE = { id = "FREE", name = "FREE ROAM", teams = false, limit = 0, time = 7200, desc = "Explore the map in peace. No enemies and no damage." },
}
local MODE_ORDER = { "TDM", "FFA", "GUN", "FREE" }

local MAPS = {
	NEON = { id = "NEON", name = "DOWNTOWN", tag = "MODERN CITY   MASSIVE   ENTERABLE BUILDINGS", accent = RGB(236, 170, 80),
		desc = "A massive modern city centre. Sixteen blocks around a central plaza: a bank, hotel tower, multi-storey car park, park, construction site, industrial yard and gas station. Most buildings can be entered and climbed to the roof, and an overpass runs along the southern avenue.", size = "640 x 640" },
	STATION = { id = "STATION", name = "TERMINUS CENTRAL", tag = "RAIL TERMINUS   MASSIVE   TRAINS YOU CAN ENTER", accent = RGB(230, 170, 70),
		desc = "A massive rail terminus. Fight from the bus station and forecourt through a grand hall with mezzanines, out across six tracks of parked trains and glazed footbridges, into a freight yard with a signal tower, warehouse, depot and gantry crane.", size = "600 x 600" },
	GRID = { id = "GRID", name = "NIGHT SHIFT", tag = "ABANDONED OFFICE   HORROR   TIGHT CQB", accent = RGB(120, 190, 170),
		desc = "A dead corporate tower after dark. Fight through a cubicle farm, glass meeting rooms, a server wing and an executive mezzanine while the fluorescents flicker overhead. Laptops left glowing, chairs knocked over, and something scratched on the walls.", size = "152 x 152" },
}
local MAP_ORDER = { "NEON", "STATION", "GRID" }

local LIGHTING = {
	HUB = { clock = 17.5, bright = 2.0, ambient = RGB(92, 96, 108), outdoor = RGB(100, 104, 118), fog = RGB(70, 76, 90), fogEnd = 2600,
		exposure = 0.2, density = 0.22, atmo = RGB(120, 128, 145), haze = 0.8, bloom = 0.45, tint = RGB(255, 246, 236), sat = -0.15 },
	NEON = { clock = 15.6, bright = 2.0, ambient = RGB(98, 102, 112), outdoor = RGB(120, 122, 130), fog = RGB(172, 178, 188), fogEnd = 3200,
		exposure = 0.05, density = 0.22, atmo = RGB(186, 192, 202), haze = 0.9, bloom = 0.22, tint = RGB(255, 248, 238), sat = -0.08, shadows = false, spec = 0, diff = 0.35 },
	STATION = { clock = 16.8, bright = 1.7, ambient = RGB(96, 98, 108), outdoor = RGB(118, 116, 122), fog = RGB(176, 168, 160), fogEnd = 3400,
		exposure = -0.05, density = 0.2, atmo = RGB(190, 180, 170), haze = 0.85, bloom = 0.3, tint = RGB(255, 238, 220), sat = -0.1, shadows = false, spec = 0, diff = 0.35 },
	GRID = { clock = 0, bright = 0.6, ambient = RGB(66, 76, 84), outdoor = RGB(44, 54, 64), fog = RGB(14, 20, 22), fogEnd = 3000,
		exposure = 0.15, density = 0.2, atmo = RGB(40, 52, 60), haze = 0.6, bloom = 0.5, tint = RGB(206, 226, 222), sat = -0.3, shadows = false, spec = 0, diff = 0.35 },
}

-- =====================================================================
-- GUNSMITH: attachments (max 5 per gun, one per category)
-- =====================================================================
local ATTACH_MAX = 5
local ATTACH_CATS = { "OPTIC", "MUZZLE", "BARREL", "UNDERBARREL", "MAGAZINE", "STOCK", "LASER", "AMMO" }
-- mul = multiply stat, add = add to stat
local ATTACHMENTS = {
	{ id = "XSIGHT", name = "CROSS SIGHT", cat = "OPTIC", unlock = 1, only = "VX44", desc = "AK-47 only. Open sight with tall posts and a floating cross reticle.", mul = { adsTime = 0.97 } },
	{ id = "RDOT", name = "RED DOT", cat = "OPTIC", unlock = 1, desc = "Red dot reticle. Light zoom, quick aim.", mul = { adsTime = 0.94, ads = 0.95, zoom = 0.78 } },
	{ id = "SCOPE", name = "SNIPER SCOPE", cat = "OPTIC", unlock = 7, desc = "Full scope view with a long-range zoom. Slower to raise.", mul = { zoom = 0.36, ads = 0.9, adsTime = 1.25 } },
	{ id = "ZOOM3", name = "3X SCOPE", cat = "OPTIC", unlock = 14, desc = "Strong magnification with a scope reticle. Slower aim.", mul = { zoom = 0.38, adsTime = 1.16 } },
	{ id = "SUPP", name = "SUPPRESSOR", cat = "MUZZLE", unlock = 3, desc = "Tames recoil. Costs a little range.", mul = { kickP = 0.92, kickY = 0.92, rFar = 0.92, rNear = 0.94 } },
	{ id = "COMP", name = "COMPENSATOR", cat = "MUZZLE", unlock = 9, desc = "Strong vertical recoil control.", mul = { kickP = 0.82, hip = 1.06 } },
	{ id = "LBAR", name = "LONG BARREL", cat = "BARREL", unlock = 5, desc = "More range, slower handling.", mul = { rNear = 1.15, rFar = 1.12, adsTime = 1.08 } },
	{ id = "SBAR", name = "SHORT BARREL", cat = "BARREL", unlock = 11, desc = "Fast handling, less range.", mul = { rNear = 0.9, rFar = 0.9, adsTime = 0.88, hip = 0.88 } },
	{ id = "AGRIP", name = "ANGLED GRIP", cat = "UNDERBARREL", unlock = 4, desc = "Tighter hip fire and moving accuracy.", mul = { hip = 0.82, moveSpread = 0.8 } },
	{ id = "VGRIP", name = "VERTICAL GRIP", cat = "UNDERBARREL", unlock = 13, desc = "Steadier recoil pattern.", mul = { kickP = 0.88, kickY = 0.85 } },
	{ id = "EXTM", name = "EXTENDED MAG", cat = "MAGAZINE", unlock = 2, desc = "+40% magazine, slower reload.", mul = { mag = 1.4, reload = 1.1, reloadEmpty = 1.1 } },
	{ id = "FASTM", name = "QUICK-PULL MAG", cat = "MAGAZINE", unlock = 8, desc = "Much faster reloads.", mul = { reload = 0.78, reloadEmpty = 0.78 } },
	{ id = "LSTK", name = "LIGHT STOCK", cat = "STOCK", unlock = 6, desc = "Move faster, kick harder.", mul = { kickP = 1.08, adsTime = 0.94 }, add = { speed = 0.04 } },
	{ id = "HSTK", name = "HEAVY STOCK", cat = "STOCK", unlock = 15, desc = "Rock-solid recoil, heavier feel.", mul = { kickP = 0.85, kickY = 0.85, adsTime = 1.06 }, add = { speed = -0.03 } },
	{ id = "LASER", name = "LASER SIGHT", cat = "LASER", unlock = 10, desc = "Big hip-fire accuracy boost.", mul = { hip = 0.72, moveSpread = 0.9 } },
	{ id = "HP", name = "HOLLOW POINT", cat = "AMMO", unlock = 12, desc = "+8% damage up close, less range.", mul = { dmg = 1.08, rFar = 0.92 } },
	{ id = "AP", name = "AP ROUNDS", cat = "AMMO", unlock = 18, desc = "Better headshots and long-range damage.", mul = { dmgFar = 1.1 }, add = { headMult = 0.15 } },
	{ id = "PHANTOM", name = "PHANTOM SUPPRESSOR", cat = "MUZZLE", unlock = 1, premium = true, desc = "PREMIUM. Near-silent, big recoil control, no range loss.", mul = { kickP = 0.78, kickY = 0.78, dmg = 1.03 } },
	{ id = "HYPER", name = "HYPER BARREL", cat = "BARREL", unlock = 1, premium = true, desc = "PREMIUM. Extended reach and faster aim.", mul = { rNear = 1.22, rFar = 1.2, adsTime = 0.95 } },
	{ id = "TGRIP", name = "TACTICAL GRIP", cat = "UNDERBARREL", unlock = 1, premium = true, desc = "PREMIUM. Strong recoil control and tighter hip fire.", mul = { kickP = 0.82, kickY = 0.82, hip = 0.85 } },
	{ id = "CSTK", name = "CARBON STOCK", cat = "STOCK", unlock = 1, premium = true, desc = "PREMIUM. Calm recoil and faster aim without the weight.", mul = { kickP = 0.86, adsTime = 0.92 }, add = { speed = 0.02 } },
	{ id = "TRACER", name = "TRACER ROUNDS", cat = "AMMO", unlock = 1, premium = true, desc = "PREMIUM. Glowing rounds with a little extra damage.", mul = { dmg = 1.05, dmgFar = 1.06 } },
	{ id = "DRUM", name = "DRUM MAGAZINE", cat = "MAGAZINE", unlock = 1, premium = true, desc = "PREMIUM. Double the magazine.", mul = { mag = 2.0, reload = 1.2, reloadEmpty = 1.2 } },
}
local ATTACH_BY_ID = {}
for _, a in ipairs(ATTACHMENTS) do ATTACH_BY_ID[a.id] = a end

-- returns a cleaned list (valid ids, one per category, owned at this level, max 5)
local function cleanAttach(list, level, owned, wid)
	local out, used = {}, {}
	if type(list) ~= "table" then return out end
	for _, id in ipairs(list) do
		local a = type(id) == "string" and ATTACH_BY_ID[id]
		if a and (not a.only or wid == nil or a.only == wid) and not used[a.cat] and a.unlock <= (level or 1) and #out < ATTACH_MAX and (not a.premium or owned == true or (type(owned) == "table" and owned["A_" .. a.id])) then
			used[a.cat] = true
			table.insert(out, id)
		end
	end
	return out
end

local effCache = {}
local function effDef(base, list)
	if not list or #list == 0 then return base end
	local key = base.id .. ":" .. table.concat(list, ",")
	local hit = effCache[key]
	if hit then return hit end
	local d = {}
	for k, v in pairs(base) do d[k] = v end
	for _, id in ipairs(list) do
		local a = ATTACH_BY_ID[id]
		if a then
			for k, m in pairs(a.mul or {}) do
				if type(d[k]) == "number" then d[k] = d[k] * m end
			end
			for k, v in pairs(a.add or {}) do
				if type(d[k]) == "number" then d[k] = d[k] + v end
			end
		end
	end
	d.mag = math.max(1, math.floor(d.mag + 0.5))
	for _, id in ipairs(list) do if id == "SCOPE" then d.scopeOpt = true end end
	d.att = list
	effCache[key] = d
	return d
end

-- =====================================================================
-- STORE CATALOG (Robux developer products). Create one Developer Product per item in Creator Hub,
-- then paste its numeric ID below. ID 0 = not configured (in Studio the item is granted free for testing).
-- =====================================================================
local STORE_PRODUCTS = {
	OP_SPECTRE = 0, OP_EMBER = 0, OP_GLACIER = 0, OP_VIPER = 0, OP_ONYX = 0,
	W_ARC9 = 0, W_REAPER = 0, W_WRAITH = 0, W_TALON = 0, W_ECLIPSE = 0, W_BANSHEE = 0,
	C_GOLD = 0, C_OBSIDIAN = 0, C_CRIMSON = 0, C_ARCTIC = 0, C_TOXIC = 0,
	A_PHANTOM = 0, A_HYPER = 0, A_DRUM = 0,
	OP_NOVA = 0, OP_RONIN = 0, OP_TITAN = 0,
	C_NEON = 0, C_MAGMA = 0, C_VOID = 0, C_ROYAL = 0, C_TIGER = 0, C_DIGITAL = 0, C_CHECKER = 0, C_BLOSSOM = 0, C_CARBON = 0, C_OCEAN = 0,
	A_TGRIP = 0, A_CSTK = 0, A_TRACER = 0,
	PACK_ELITE = 0, PACK_STEALTH = 0, PACK_ARCTIC = 0, PASS_PREMIUM = 0,
}
local OPERATORS = {
	OP_STANDARD = { name = "RECON", suit = RGB(66, 74, 62), accent = RGB(255, 140, 30), free = true, desc = "Standard-issue field operator." },
	-- earned by progress
	OP_SCOUT = { name = "SCOUT", suit = RGB(66, 96, 64), accent = RGB(130, 255, 110), req = { kind = "level", n = 5 }, desc = "Forest recon runner." },
	OP_SENTINEL = { name = "SENTINEL", suit = RGB(50, 62, 92), accent = RGB(80, 200, 255), req = { kind = "level", n = 10 }, desc = "Disciplined perimeter guard." },
	OP_HUNTER = { name = "HUNTER", suit = RGB(84, 62, 38), accent = RGB(255, 160, 50), req = { kind = "kills", n = 100 }, desc = "Veteran tracker. 100 career kills." },
	OP_MARKSMAN = { name = "MARKSMAN", suit = RGB(38, 44, 54), accent = RGB(255, 235, 90), req = { kind = "headshots", n = 50 }, desc = "Precision specialist. 50 headshots." },
	OP_VETERAN = { name = "VETERAN", suit = RGB(100, 40, 40), accent = RGB(255, 90, 90), req = { kind = "wins", n = 10 }, desc = "Battle-hardened winner. 10 match wins." },
	OP_WARLORD = { name = "WARLORD", suit = RGB(46, 18, 60), accent = RGB(255, 205, 70), req = { kind = "level", n = 25 }, desc = "Elite commander. Level 25." },
	-- bought in the store
	OP_SPECTRE = { name = "SPECTRE", suit = RGB(22, 26, 36), accent = RGB(60, 150, 255), desc = "Stealth black-ops operator." },
	OP_EMBER = { name = "EMBER", suit = RGB(58, 30, 24), accent = RGB(255, 120, 30), desc = "Heat-forged assault trooper." },
	OP_GLACIER = { name = "GLACIER", suit = RGB(176, 190, 210), accent = RGB(120, 220, 255), desc = "Arctic recon specialist." },
	OP_VIPER = { name = "VIPER", suit = RGB(30, 52, 34), accent = RGB(120, 255, 90), desc = "Toxic-green infiltrator." },
	OP_NOVA = { name = "NOVA", suit = RGB(96, 36, 128), accent = RGB(255, 80, 220), desc = "Neon-lit shock trooper." },
	OP_RONIN = { name = "RONIN", suit = RGB(78, 16, 22), accent = RGB(255, 60, 60), desc = "Crimson blade-runner assassin." },
	OP_TITAN = { name = "TITAN", suit = RGB(92, 98, 110), accent = RGB(255, 170, 40), desc = "Heavy-armour juggernaut." },
	OP_ONYX = { name = "ONYX", suit = RGB(18, 18, 20), accent = RGB(255, 205, 70), desc = "Elite gold-trim commander." },
}
local OPERATOR_ORDER = { "OP_STANDARD", "OP_SCOUT", "OP_SENTINEL", "OP_HUNTER", "OP_MARKSMAN", "OP_VETERAN", "OP_WARLORD",
	"OP_SPECTRE", "OP_EMBER", "OP_GLACIER", "OP_VIPER", "OP_NOVA", "OP_RONIN", "OP_TITAN", "OP_ONYX" }
-- ---------- operator looks (custom soldier builds, see dressOperator) ----------
local OP_LOOKS = {
	OP_STANDARD = { skin = RGB(214, 170, 138), vest = "plate", head = "helmet" },
	OP_SCOUT = { skin = RGB(190, 140, 108), vest = "rig", head = "cap", hair = RGB(60, 40, 24), tail = false },
	OP_SENTINEL = { skin = RGB(120, 84, 62), vest = "plate", head = "visorhelm" },
	OP_HUNTER = { skin = RGB(206, 160, 128), vest = "jacket", head = "hoodantler", tufts = RGB(70, 80, 44) },
	OP_MARKSMAN = { skin = RGB(224, 184, 150), vest = "rig", head = "wrap", tufts = RGB(52, 62, 40) },
	OP_VETERAN = { skin = RGB(180, 130, 100), vest = "plate", head = "beret", hair = RGB(30, 24, 20) },
	OP_WARLORD = { skin = RGB(150, 104, 80), vest = "armor", head = "horned", cape = RGB(70, 14, 24) },
	OP_SPECTRE = { skin = RGB(190, 150, 120), vest = "rig", head = "hoodmask", pants = RGB(14, 16, 22) },
	OP_EMBER = { skin = RGB(160, 110, 84), vest = "plate", head = "flame", padCol = RGB(255, 120, 30) },
	OP_GLACIER = { skin = RGB(230, 196, 170), vest = "parka", head = "arctic", pants = RGB(120, 134, 154) },
	OP_VIPER = { skin = RGB(176, 130, 100), vest = "rig", head = "gasmask", pants = RGB(20, 36, 24) },
	OP_NOVA = { skin = RGB(210, 160, 140), vest = "armor", head = "halo", hair = RGB(255, 80, 220) },
	OP_RONIN = { skin = RGB(210, 166, 130), vest = "armor", head = "kabuto", cape = RGB(60, 8, 12) },
	OP_TITAN = { skin = RGB(110, 76, 58), vest = "armor", head = "heavy", plate = true, pads = true },
	OP_ONYX = { skin = RGB(120, 84, 64), vest = "suit", head = "goldhelm", chain = RGB(255, 205, 70) },
}
-- 16 original operators inspired by the same archetypes (racer, ape, skull-mask, lizard, explorer, astronaut, bunnies, rappers, clown, zombie ...)
local NEW_OPS = {
	{ id = "OP_SPEEDSTER", name = "SPEEDSTER", suit = RGB(220, 40, 36), accent = RGB(255, 255, 255), robux = 449, tag = "Street-racer turned operator",
		look = { skin = RGB(200, 150, 120), top = RGB(214, 36, 34), pants = RGB(30, 30, 36), vest = "jacket", head = "racehelm", trim = RGB(255, 255, 255) },
		lore = "Red race leathers, a full-face helmet and a need for speed. Gets to the objective before the other team has loaded in." },
	{ id = "OP_APEX", name = "APEX", suit = RGB(120, 62, 28), accent = RGB(255, 170, 60), robux = 599, tag = "Alpha of the jungle",
		look = { fur = RGB(176, 84, 30), skin = RGB(176, 84, 30), top = RGB(150, 70, 26), pants = RGB(70, 40, 20), vest = "rig", head = "ape", bareHands = true },
		lore = "Part soldier, part primate. Orange fur, heavy brow and a strong opinion about who owns this territory." },
	{ id = "OP_SKULLCAP", name = "SKULLCAP", suit = RGB(28, 28, 32), accent = RGB(235, 235, 225), robux = 499, tag = "Silent. Masked. Gone.",
		look = { skin = RGB(190, 150, 120), top = RGB(26, 26, 30), pants = RGB(20, 20, 24), vest = "rig", head = "skullmask", glove = RGB(14, 14, 16) },
		lore = "A black cowl, a painted skull and one rule: nobody sees the face. Takes contracts nobody else will." },
	{ id = "OP_GECKO", name = "GECKO", suit = RGB(70, 120, 50), accent = RGB(255, 220, 60), robux = 449, tag = "Cold-blooded stalker",
		look = { scale = RGB(96, 150, 56), skin = RGB(96, 150, 56), top = RGB(60, 100, 44), pants = RGB(44, 64, 36), vest = "rig", head = "lizard", tail = RGB(96, 150, 56), bareHands = true },
		lore = "Scales, a tail and unblinking yellow eyes. Waits in the long grass until the timing is perfect." },
	{ id = "OP_EXPLORER", name = "EXPLORER", suit = RGB(120, 96, 62), accent = RGB(255, 190, 70), robux = 399, tag = "Tomb raider, tactical edition",
		look = { skin = RGB(220, 176, 144), top = RGB(150, 130, 96), pants = RGB(60, 70, 60), vest = "jacket", head = "explorer", hair = RGB(88, 54, 30), holster = true },
		lore = "Braided hair, worn leather and a dual-pistol habit. Has been in every ruin and every firefight since." },
	{ id = "OP_ASTRO", name = "ASTRO", suit = RGB(226, 230, 238), accent = RGB(80, 200, 255), robux = 549, tag = "First soldier in orbit",
		look = { skin = RGB(210, 170, 140), top = RGB(228, 232, 240), pants = RGB(214, 218, 228), boots = RGB(120, 128, 142), glove = RGB(190, 196, 208), vest = "armor", head = "spacehelm" },
		lore = "A pressure suit with a glass dome and an antenna. Training for the moon, deployed to the map instead." },
	{ id = "OP_HOPPER", name = "HOPPER", suit = RGB(236, 236, 240), accent = RGB(255, 150, 190), robux = 349, tag = "Fast, fluffy, lethal",
		look = { fur = RGB(244, 244, 248), skin = RGB(244, 244, 248), top = RGB(230, 230, 236), pants = RGB(200, 204, 214), vest = "hoodie", head = "bunny", inner = RGB(255, 170, 190) },
		lore = "White fur, long ears and a wide smile. Do not let the smile fool you." },
	{ id = "OP_THUMPER", name = "THUMPER", suit = RGB(60, 70, 96), accent = RGB(120, 220, 255), robux = 349, tag = "Grey bunny, heavy boots",
		look = { fur = RGB(130, 138, 156), skin = RGB(130, 138, 156), top = RGB(54, 62, 86), pants = RGB(34, 40, 56), vest = "hoodie", head = "bunny", inner = RGB(200, 150, 190) },
		lore = "Hopper's grey cousin. Slower hop, harder landing." },
	{ id = "OP_STEPPER", name = "STEPPER", suit = RGB(50, 40, 80), accent = RGB(255, 200, 60), robux = 399, tag = "Block party commander",
		look = { skin = RGB(104, 70, 52), top = RGB(40, 34, 70), pants = RGB(26, 26, 36), vest = "hoodie", head = "capbeard", hair = RGB(18, 14, 12), chain = RGB(255, 205, 70) },
		lore = "Fitted cap, full beard and heavy gold chain. Runs the block and the lobby." },
	{ id = "OP_WARDEN", name = "WARDEN", suit = RGB(40, 40, 46), accent = RGB(255, 60, 60), robux = 449, tag = "Bandana, beanie, no mercy",
		look = { skin = RGB(190, 146, 116), top = RGB(36, 36, 42), pants = RGB(28, 32, 44), vest = "jacket", head = "beanie", bandana = RGB(170, 30, 36) },
		lore = "Keeps the peace the hard way. The red bandana is not a fashion choice." },
	{ id = "OP_VANDAL", name = "VANDAL", suit = RGB(236, 238, 244), accent = RGB(255, 255, 255), robux = 399, tag = "White hood, dark glasses",
		look = { skin = RGB(150, 106, 80), top = RGB(236, 238, 244), pants = RGB(70, 76, 90), vest = "hoodie", head = "hoodwhite", fur = RGB(244, 246, 250) },
		lore = "Clean white hoodie, black shades, headphones. Spray cans for walls, rifles for everything else." },
	{ id = "OP_MAESTRO", name = "MAESTRO", suit = RGB(90, 40, 120), accent = RGB(255, 205, 70), robux = 449, tag = "Showtime on the battlefield",
		look = { skin = RGB(200, 160, 130), top = RGB(110, 46, 150), pants = RGB(30, 24, 44), vest = "jacket", head = "shades", hairCol = RGB(40, 30, 24), chain = RGB(255, 205, 70) },
		lore = "Sharp jacket, sharper shades and a stage presence that fills the map." },
	{ id = "OP_NEONPOP", name = "NEON POP", suit = RGB(255, 90, 170), accent = RGB(120, 240, 255), robux = 549, tag = "Pink hair, hard hitter",
		look = { skin = RGB(224, 184, 156), top = RGB(255, 100, 175), pants = RGB(40, 30, 60), vest = "jacket", head = "pinkhair", hairCol = RGB(255, 90, 170), boots = RGB(255, 255, 255) },
		lore = "Bubblegum pink hair, bright jacket and a high score to match." },
	{ id = "OP_VOGUE", name = "VOGUE", suit = RGB(236, 204, 120), accent = RGB(255, 255, 255), robux = 499, tag = "Blonde. Glamorous. Armed.",
		look = { skin = RGB(224, 180, 150), top = RGB(230, 196, 110), pants = RGB(30, 28, 30), vest = "suit", head = "shades", hairCol = RGB(238, 206, 132) },
		lore = "Platinum waves, dark shades and a tailored fit. Arrives in style and leaves in silence." },
	{ id = "OP_UNDEAD", name = "UNDEAD", suit = RGB(80, 90, 70), accent = RGB(180, 255, 120), robux = 599, tag = "Back from the grave",
		look = { skin = RGB(130, 150, 118), top = RGB(74, 80, 66), pants = RGB(46, 50, 40), vest = "jacket", head = "zombie", bareHands = true },
		lore = "Torn clothes, stitched wounds and glowing eyes. Keeps coming no matter what you do." },
	{ id = "OP_HARLEQUIN", name = "HARLEQUIN", suit = RGB(250, 250, 250), accent = RGB(255, 60, 90), robux = 549, tag = "The joke is on you",
		look = { skin = RGB(246, 244, 240), top = RGB(250, 250, 250), pants = RGB(60, 130, 255), vest = "jacket", head = "clown", hairCol = RGB(255, 60, 90), hairCol2 = RGB(60, 130, 255), hairCol3 = RGB(255, 210, 60), glove = RGB(250, 250, 250) },
		lore = "Face paint, puff hair and a big smile. Unpredictable, loud and very dangerous." },
}
for id, l in pairs(OP_LOOKS) do OPERATORS[id].look = l end
for _, n in ipairs(NEW_OPS) do
	OPERATORS[n.id] = { name = n.name, suit = n.suit, accent = n.accent, desc = n.tag, look = n.look }
	table.insert(OPERATOR_ORDER, n.id)
	STORE_PRODUCTS[n.id] = 0
end
-- progress toward an earned operator: st = { level, kills, headshots, wins }
local function operatorProgress(op, st)
	local r = op.req
	if not r then return 0, 0 end
	local cur = (r.kind == "level") and (st.level or 1) or (st[r.kind] or 0)
	return cur, r.n
end
local function operatorOwned(id, owned, st)
	local op = OPERATORS[id]
	if not op then return false end
	if op.free then return true end
	if owned and owned[id] == true then return true end
	if op.req then
		local cur, need = operatorProgress(op, st or {})
		return cur >= need
	end
	return false
end
local CAMOS = {
	C_NONE = { name = "STANDARD", free = true },
	C_GOLD = { name = "GOLD PLATED", body = RGB(212, 170, 60), accent = RGB(255, 230, 140), style = "plates", c1 = RGB(255, 242, 180), mat = Enum.Material.Metal },
	C_OBSIDIAN = { name = "OBSIDIAN", body = RGB(30, 24, 46), accent = RGB(170, 110, 255), style = "cracks", c1 = RGB(200, 140, 255) },
	C_CRIMSON = { name = "CRIMSON", body = RGB(160, 28, 36), accent = RGB(255, 70, 70), style = "tiger", c1 = RGB(24, 6, 8) },
	C_ARCTIC = { name = "ARCTIC", body = RGB(196, 208, 222), accent = RGB(120, 210, 255), style = "blotch", c1 = RGB(150, 186, 228), c2 = RGB(244, 250, 255), mat = Enum.Material.Ice },
	C_TOXIC = { name = "TOXIC", body = RGB(40, 84, 34), accent = RGB(150, 255, 90), style = "digital", c1 = RGB(130, 210, 70), c2 = RGB(14, 34, 12) },
	C_NEON = { name = "NEON NIGHTS", body = RGB(34, 12, 60), accent = RGB(255, 60, 220), style = "grid", c1 = RGB(255, 60, 220), c2 = RGB(40, 220, 255) },
	C_MAGMA = { name = "MAGMA", body = RGB(58, 18, 10), accent = RGB(255, 130, 20), style = "flames", c1 = RGB(255, 110, 20), c2 = RGB(255, 222, 80) },
	C_VOID = { name = "VOID", body = RGB(14, 16, 44), accent = RGB(90, 150, 255), style = "stars", c1 = RGB(255, 255, 255), c2 = RGB(90, 140, 255) },
	C_ROYAL = { name = "ROYAL", body = RGB(64, 36, 140), accent = RGB(255, 215, 80), style = "stripes", c1 = RGB(255, 215, 80) },
	C_TIGER = { name = "TIGER STRIKE", body = RGB(232, 130, 30), accent = RGB(255, 190, 80), style = "tiger", c1 = RGB(30, 16, 8) },
	C_DIGITAL = { name = "URBAN DIGITAL", body = RGB(96, 104, 120), accent = RGB(200, 212, 235), style = "digital", c1 = RGB(180, 190, 206), c2 = RGB(40, 46, 60) },
	C_CHECKER = { name = "CHECKERED", body = RGB(240, 240, 240), accent = RGB(255, 255, 255), style = "checker", c1 = RGB(20, 20, 24), c2 = RGB(240, 240, 240) },
	C_BLOSSOM = { name = "BLOSSOM", body = RGB(255, 170, 200), accent = RGB(255, 255, 255), style = "dots", c1 = RGB(255, 255, 255), c2 = RGB(255, 90, 150) },
	C_CARBON = { name = "CARBON FIBRE", body = RGB(34, 36, 42), accent = RGB(150, 165, 190), style = "carbon", c1 = RGB(104, 112, 128), c2 = RGB(60, 64, 74), mat = Enum.Material.Metal },
	C_OCEAN = { name = "DEEP OCEAN", body = RGB(24, 84, 170), accent = RGB(80, 220, 255), style = "waves", c1 = RGB(80, 220, 255), c2 = RGB(255, 255, 255) },
}
local CAMO_ORDER = { "C_NONE", "C_GOLD", "C_OBSIDIAN", "C_CRIMSON", "C_ARCTIC", "C_TOXIC", "C_NEON", "C_MAGMA", "C_VOID", "C_ROYAL", "C_TIGER", "C_DIGITAL", "C_CHECKER", "C_BLOSSOM", "C_CARBON", "C_OCEAN" }
-- store listing: kind, display name, Robux price, what it grants
local STORE_ITEMS = {
	{ id = "PACK_ELITE", kind = "BUNDLES", name = "BLACKOUT ELITE PACK", robux = 999, desc = "ONYX operator + GOLD camo + REAPER-X rifle + PHANTOM suppressor.",
		grants = { "OP_ONYX", "C_GOLD", "W_REAPER", "A_PHANTOM" }, accent = RGB(255, 205, 70) },
	{ id = "PACK_STEALTH", kind = "BUNDLES", name = "STEALTH PACK", robux = 599, desc = "SPECTRE operator, OBSIDIAN camo and the PHANTOM suppressor.",
		grants = { "OP_SPECTRE", "C_OBSIDIAN", "A_PHANTOM" }, accent = RGB(60, 150, 255) },
	{ id = "PACK_ARCTIC", kind = "BUNDLES", name = "ARCTIC PACK", robux = 749, desc = "GLACIER operator, ARCTIC camo and the WRAITH-5 burst rifle.",
		grants = { "OP_GLACIER", "C_ARCTIC", "W_WRAITH" }, accent = RGB(120, 220, 255) },
	{ id = "OP_SPECTRE", kind = "OPERATORS", name = "SPECTRE", robux = 299, desc = OPERATORS.OP_SPECTRE.desc, accent = OPERATORS.OP_SPECTRE.accent },
	{ id = "OP_EMBER", kind = "OPERATORS", name = "EMBER", robux = 299, desc = OPERATORS.OP_EMBER.desc, accent = OPERATORS.OP_EMBER.accent },
	{ id = "OP_GLACIER", kind = "OPERATORS", name = "GLACIER", robux = 349, desc = OPERATORS.OP_GLACIER.desc, accent = OPERATORS.OP_GLACIER.accent },
	{ id = "OP_VIPER", kind = "OPERATORS", name = "VIPER", robux = 399, desc = OPERATORS.OP_VIPER.desc, accent = OPERATORS.OP_VIPER.accent },
	{ id = "OP_NOVA", kind = "OPERATORS", name = "NOVA", robux = 449, desc = OPERATORS.OP_NOVA.desc, accent = OPERATORS.OP_NOVA.accent },
	{ id = "OP_RONIN", kind = "OPERATORS", name = "RONIN", robux = 399, desc = OPERATORS.OP_RONIN.desc, accent = OPERATORS.OP_RONIN.accent },
	{ id = "OP_TITAN", kind = "OPERATORS", name = "TITAN", robux = 499, desc = OPERATORS.OP_TITAN.desc, accent = OPERATORS.OP_TITAN.accent },
	{ id = "OP_ONYX", kind = "OPERATORS", name = "ONYX", robux = 599, desc = OPERATORS.OP_ONYX.desc, accent = OPERATORS.OP_ONYX.accent },
	{ id = "W_ARC9", kind = "WEAPONS", name = "ARC-9 VOLTAGE", robux = 499, desc = "SMG. 1100 RPM, 40-round mag.", weapon = "ARC9", accent = RGB(0, 255, 240) },
	{ id = "W_REAPER", kind = "WEAPONS", name = "REAPER-X", robux = 599, desc = "Heavy full-auto rifle, long reach.", weapon = "REAPER", accent = RGB(255, 50, 60) },
	{ id = "W_WRAITH", kind = "WEAPONS", name = "WRAITH-5", robux = 599, desc = "4-round burst rifle.", weapon = "WRAITH", accent = RGB(180, 90, 255) },
	{ id = "W_ECLIPSE", kind = "WEAPONS", name = "ECLIPSE RAILGUN", robux = 699, desc = "Rail sniper. One shot, any range.", weapon = "ECLIPSE", accent = RGB(80, 220, 255) },
	{ id = "W_BANSHEE", kind = "WEAPONS", name = "BANSHEE-AS", robux = 649, desc = "Full-auto sniper rifle.", weapon = "BANSHEE", accent = RGB(255, 60, 160) },
	{ id = "W_TALON", kind = "WEAPONS", name = "TALON .50", robux = 399, desc = "Hand cannon sidearm.", weapon = "TALON", accent = RGB(255, 200, 60) },
	{ id = "C_GOLD", kind = "CAMOS", name = "GOLD PLATED", robux = 249, desc = "Weapon camo.", accent = CAMOS.C_GOLD.accent },
	{ id = "C_OBSIDIAN", kind = "CAMOS", name = "OBSIDIAN", robux = 199, desc = "Weapon camo.", accent = CAMOS.C_OBSIDIAN.accent },
	{ id = "C_CRIMSON", kind = "CAMOS", name = "CRIMSON", robux = 199, desc = "Weapon camo.", accent = CAMOS.C_CRIMSON.accent },
	{ id = "C_ARCTIC", kind = "CAMOS", name = "ARCTIC", robux = 199, desc = "Weapon camo.", accent = CAMOS.C_ARCTIC.accent },
	{ id = "C_TOXIC", kind = "CAMOS", name = "TOXIC", robux = 199, desc = "Weapon camo.", accent = CAMOS.C_TOXIC.accent },
	{ id = "C_NEON", kind = "CAMOS", name = "NEON NIGHTS", robux = 249, desc = "Weapon camo.", accent = CAMOS.C_NEON.accent },
	{ id = "C_MAGMA", kind = "CAMOS", name = "MAGMA", robux = 249, desc = "Weapon camo.", accent = CAMOS.C_MAGMA.accent },
	{ id = "C_VOID", kind = "CAMOS", name = "VOID", robux = 299, desc = "Weapon camo.", accent = CAMOS.C_VOID.accent },
	{ id = "C_ROYAL", kind = "CAMOS", name = "ROYAL", robux = 299, desc = "Weapon camo.", accent = CAMOS.C_ROYAL.accent },
	{ id = "C_TIGER", kind = "CAMOS", name = "TIGER STRIKE", robux = 249, desc = "", accent = CAMOS.C_TIGER.accent },
	{ id = "C_DIGITAL", kind = "CAMOS", name = "URBAN DIGITAL", robux = 199, desc = "", accent = CAMOS.C_DIGITAL.accent },
	{ id = "C_CHECKER", kind = "CAMOS", name = "CHECKERED", robux = 249, desc = "", accent = CAMOS.C_CHECKER.accent },
	{ id = "C_BLOSSOM", kind = "CAMOS", name = "BLOSSOM", robux = 299, desc = "", accent = CAMOS.C_BLOSSOM.accent },
	{ id = "C_CARBON", kind = "CAMOS", name = "CARBON FIBRE", robux = 299, desc = "", accent = CAMOS.C_CARBON.accent },
	{ id = "C_OCEAN", kind = "CAMOS", name = "DEEP OCEAN", robux = 249, desc = "", accent = CAMOS.C_OCEAN.accent },
	{ id = "A_TGRIP", kind = "ATTACHMENTS", name = "TACTICAL GRIP", robux = 149, desc = "", attach = "TGRIP", accent = RGB(255, 170, 60) },
	{ id = "A_CSTK", kind = "ATTACHMENTS", name = "CARBON STOCK", robux = 149, desc = "", attach = "CSTK", accent = RGB(120, 255, 200) },
	{ id = "A_TRACER", kind = "ATTACHMENTS", name = "TRACER ROUNDS", robux = 199, desc = "", attach = "TRACER", accent = RGB(255, 90, 200) },
	{ id = "A_PHANTOM", kind = "ATTACHMENTS", name = "PHANTOM SUPPRESSOR", robux = 149, desc = "Muzzle. Best-in-class recoil control.", attach = "PHANTOM", accent = RGB(120, 200, 255) },
	{ id = "A_HYPER", kind = "ATTACHMENTS", name = "HYPER BARREL", robux = 149, desc = "Barrel. Reach and quick aim.", attach = "HYPER", accent = RGB(120, 200, 255) },
	{ id = "A_DRUM", kind = "ATTACHMENTS", name = "DRUM MAGAZINE", robux = 149, desc = "Magazine. Double capacity.", attach = "DRUM", accent = RGB(120, 200, 255) },
}
for _, n in ipairs(NEW_OPS) do
	table.insert(STORE_ITEMS, { id = n.id, kind = "OPERATORS", name = n.name, robux = n.robux, desc = n.tag, lore = n.lore, accent = n.accent })
end
table.insert(STORE_ITEMS, 1, { id = "PASS_PREMIUM", kind = "BUNDLES", name = "BLACKOUT PREMIUM PASS", robux = 799, desc = "Unlocks 50+ premium items (operators, weapons, camos) and bonus XP across all 100 levels.",
	grants = {}, accent = RGB(255, 205, 70) })
local STORE_BY_ID = {}
for _, it in ipairs(STORE_ITEMS) do STORE_BY_ID[it.id] = it end

-- ---------- BATTLE PASS: 100 levels, free track + premium track ----------
local PASS_MAX = 100
local PASS_FREE_ITEMS = { [25] = "C_CRIMSON", [50] = "A_TRACER", [75] = "C_ARCTIC", [100] = "OP_EMBER" }
local PASS_FINAL = { "W_ECLIPSE", "OP_ONYX", "C_GOLD", "W_REAPER" }
local PASS_ADJ = { "IRON", "SILENT", "CRIMSON", "NIGHT", "STORM", "GHOST", "RAZOR", "SHADOW", "FERAL", "ARCTIC", "DEAD", "BLACK", "STEEL", "VIPER", "ROGUE" }
local PASS_NOUN = { "WOLF", "HAWK", "COBRA", "REAPER", "KNIGHT", "FALCON", "SPECTRE", "TIGER", "RAVEN", "SNIPER", "TITAN", "DRAGON", "GHOST", "BADGE", "SKULL" }
local PASS = { free = {}, prem = {} }
do
	-- premium: a real item on every even level, ordered cheapest to best, with a bonus-XP reward on every odd level
	local skip = { PASS_PREMIUM = true }
	for _, id in pairs(PASS_FREE_ITEMS) do skip[id] = true end
	for _, id in ipairs(PASS_FINAL) do skip[id] = true end
	local pool = {}
	for _, it in ipairs(STORE_ITEMS) do
		if not skip[it.id] and it.kind ~= "BUNDLES" then table.insert(pool, it) end
	end
	local rank = { ATTACHMENTS = 1, CAMOS = 2, OPERATORS = 3, WEAPONS = 4 }
	table.sort(pool, function(x, y)
		if x.robux ~= y.robux then return x.robux < y.robux end
		if rank[x.kind] ~= rank[y.kind] then return rank[x.kind] < rank[y.kind] end
		return x.id < y.id
	end)
	local slots = {}
	for l = 2, 98, 2 do table.insert(slots, l) end
	for i, it in ipairs(pool) do
		local l = slots[math.max(1, math.min(#slots, math.floor((i - 0.5) * #slots / #pool + 0.5)))]
		while PASS.prem[l] and l < 98 do l += 2 end
		PASS.prem[l] = { kind = "item", ids = { it.id }, label = it.name }
	end
	for l = 1, 100 do
		local fi = PASS_FREE_ITEMS[l]
		if fi then
			PASS.free[l] = { kind = "item", ids = { fi }, label = STORE_BY_ID[fi] and STORE_BY_ID[fi].name or fi }
		elseif l % 5 == 0 then
			PASS.free[l] = { kind = "card", label = PASS_ADJ[(l * 7) % #PASS_ADJ + 1] .. " " .. PASS_NOUN[(l * 3) % #PASS_NOUN + 1], id = "CARD" .. l }
		else
			local n = 50 + math.floor(l * 1.5)
			PASS.free[l] = { kind = "xp", n = n, label = "+" .. n .. " XP" }
		end
		if l == 100 then
			PASS.prem[l] = { kind = "item", ids = PASS_FINAL, label = "COMMANDER SET" }
		elseif not PASS.prem[l] then
			local n = 150 + l * 6
			PASS.prem[l] = { kind = "xp", n = n, label = "+" .. n .. " XP" }
		end
	end
end

local STORE_TABS = { "BUNDLES", "OPERATORS", "WEAPONS", "CAMOS", "ATTACHMENTS" }

-- short card tagline + longer preview description for every store item
local STORE_TEXT = {
	PACK_ELITE = { "Gold-trim commander loadout", "Everything the top of the leaderboard wears. ONYX armour, a gold-plated REAPER-X heavy rifle and the PHANTOM suppressor to keep your position hidden." },
	PACK_STEALTH = { "Silent. Dark. Deadly.", "Disappear into the blackout. SPECTRE stealth armour, an OBSIDIAN weapon finish and the PHANTOM suppressor for quiet, controlled kills." },
	PACK_ARCTIC = { "Cold steel, burst fire", "GLACIER arctic gear, a frost-white ARCTIC weapon finish and the WRAITH-5 burst rifle for clean, precise eliminations." },
	OP_SPECTRE = { "Stealth black-ops operator", "Matte-black suit with electric-blue trim. Built to vanish in dark corridors and strike from the shadows." },
	OP_EMBER = { "Heat-forged assault trooper", "Scorched armour with a glowing orange visor. Looks right at home when the whole map is on fire." },
	OP_GLACIER = { "Arctic recon specialist", "Pale ice-grey kit with a cyan visor. Calm, cold and very hard to miss." },
	OP_VIPER = { "Toxic-green infiltrator", "Dark green suit with a venom-green glow. Fast, quiet and always one step ahead." },
	OP_NOVA = { "Neon-lit shock trooper", "Violet armour with hot-pink lights. The loudest look in the lobby." },
	OP_RONIN = { "Crimson blade-runner", "Blood-red suit and a burning red visor. A lone assassin with nothing left to prove." },
	OP_TITAN = { "Heavy-armour juggernaut", "Steel-grey plating with amber trim. Built like a wall, walks like one too." },
	OP_ONYX = { "Elite gold-trim commander", "Jet-black armour with gold lights. The mark of a true veteran." },
	W_ARC9 = { "Rapid-fire SMG", "An electric-cyan submachine gun with a huge 40-round magazine. Shreds enemies at close range." },
	W_REAPER = { "Heavy full-auto rifle", "Hits hard and reaches far. Heavier to handle, brutal when the rounds land." },
	W_WRAITH = { "4-round burst rifle", "Precise burst fire with a ghostly violet glow. Rewards patient, steady aim." },
	W_ECLIPSE = { "Charged rail sniper", "Glowing twin rails and a crackling coil. One shot kills at any range, with no bullet drop-off." },
	W_BANSHEE = { "Full-auto sniper rifle", "A pink-lit drum-fed sniper that fires as fast as you can hold the trigger. Sweep a whole lane." },
	W_TALON = { ".50 hand cannon", "A sidearm that hits like a rifle. Few shots, huge damage. Never miss." },
	C_GOLD = { "Polished gold plating", "Gleaming gold finish with a warm glow. Every kill looks expensive." },
	C_OBSIDIAN = { "Volcanic black finish", "Deep black metal with a purple glow, like glass from a volcano." },
	C_CRIMSON = { "Deep red finish", "Dark crimson body with a bright red glow. Aggressive and loud." },
	C_ARCTIC = { "Frosted white finish", "Snow-white body with an icy-blue glow. Clean and cold." },
	C_TOXIC = { "Radioactive green finish", "Dark green body with a glowing toxic-green trim." },
	C_NEON = { "Midnight neon finish", "Deep purple body with hot-pink lights. Made for the night." },
	C_MAGMA = { "Molten lava finish", "Charred black-red body with a lava-orange glow." },
	C_VOID = { "Pure black finish", "Almost-invisible black body with an electric-blue glow." },
	C_ROYAL = { "Royal purple finish", "Deep purple body with a royal gold glow." },
	C_TIGER = { "Orange tiger stripes", "Blazing orange body with black tiger stripes slashed across it. Impossible to ignore." },
	C_DIGITAL = { "Urban pixel camo", "Grey digital pixel pattern, built for concrete jungles." },
	C_CHECKER = { "Black and white checkers", "A racing-flag checkerboard in black and white. Crisp and loud." },
	C_BLOSSOM = { "Pink with white dots", "Soft pink body with bright polka dots. Cute, and still deadly." },
	C_CARBON = { "Woven carbon finish", "Matte black carbon weave with a steel-grey grid. Race-car tech." },
	C_OCEAN = { "Blue wave bands", "Deep blue body with cyan and white wave lines running along the gun." },
	A_TGRIP = { "Grip: steady and fast", "A tactical foregrip with a glowing ring. Cuts recoil and tightens hip fire." },
	A_CSTK = { "Stock: light and steady", "A skeletal carbon stock that tames recoil, speeds up aiming and keeps you quick." },
	A_TRACER = { "Ammo: glowing rounds", "Pink tracer rounds with a glowing band on the gun. A little extra damage at any range." },
	A_PHANTOM = { "Muzzle: quiet and steady", "A near-silent suppressor with serious recoil control and no loss of range. Works on every compatible weapon." },
	A_HYPER = { "Barrel: reach and speed", "An extended barrel that adds range and lets you aim down sights faster." },
	A_DRUM = { "Magazine: double capacity", "A drum magazine that doubles your ammo before reloading. Reloads a little slower." },
}
for _, it in ipairs(STORE_ITEMS) do
	local t = STORE_TEXT[it.id]
	if t then it.desc, it.lore = t[1], t[2] end
end

local function camoDef(def, camoId)
	local c = camoId and CAMOS[camoId]
	if not c or not c.body then return def end
	local d = {}
	for k, v in pairs(def) do d[k] = v end
	d.body = c.body
	d.color = c.accent
	d.bodyMat = c.mat
	d.pat = { style = c.style, c1 = c.c1 or c.accent, c2 = c.c2 or c.accent }
	return d
end

-- =====================================================================
-- OPERATOR CHARACTERS: one shared builder dresses a rig (real R15/R6 avatar, bot rig or menu preview) as a custom soldier.
-- Every added part is named "Op..." (so it can be rebuilt) and the original avatar parts are hidden.
-- B = { head, torso, lower, uarmL, uarmR, farmL, farmR, handL, handR, ulegL, ulegR, llegL, llegR, footL, footR } (any may be nil)
-- =====================================================================
local HEADS = {}
local function dressOperator(model, opId, anchored, B, teamCol)
	local op = OPERATORS[opId] or OPERATORS.OP_STANDARD
	local L = op.look or {}
	local acc = op.accent
	local skin = L.skin or RGB(206, 162, 130)
	local top = L.top or op.suit
	local pants = L.pants or op.suit:Lerp(RGB(10, 12, 16), 0.5)
	local boots = L.boots or RGB(24, 24, 28)
	local glove = L.glove or RGB(20, 21, 25)
	local hairC = L.hair or RGB(40, 30, 24)
	local dark = RGB(22, 23, 28)
	local MET, PLA, NEON, FAB, GLS = Enum.Material.Metal, Enum.Material.SmoothPlastic, Enum.Material.Neon, Enum.Material.Fabric, Enum.Material.Glass
	for _, b in pairs(B) do
		if b and b.Parent then b.Transparency = 1 end
	end
	for _, d in ipairs(model:GetDescendants()) do
		if d:IsA("Shirt") or d:IsA("Pants") or d:IsA("ShirtGraphic") or d:IsA("Accessory") or d:IsA("BodyColors") or d:IsA("CharacterMesh")
			or (d:IsA("Decal") and d.Name == "face") then
			d:Destroy()
		end
	end
	-- P: add a part to a bone. off/rot are in the bone's own space
	local function P(bone, name, size, off, col, mat, shape, rot)
		local base = B[bone]
		if not base then return nil end
		local q = Instance.new("Part")
		q.Name = name
		q.Size = size
		q.Color = col
		q.Material = mat or PLA
		if shape then q.Shape = shape end
		q.CanCollide, q.CanTouch, q.CanQuery, q.Massless, q.CastShadow = false, false, false, true, false
		q.TopSurface, q.BottomSurface = Enum.SurfaceType.Smooth, Enum.SurfaceType.Smooth
		q.Anchored = anchored == true
		q.CFrame = base.CFrame * CFrame.new(off or Vector3.zero) * (rot or CFrame.new())
		q.Parent = model
		if not anchored then
			local w = Instance.new("WeldConstraint")
			w.Part0 = base
			w.Part1 = q
			w.Parent = q
		end
		return q
	end
	local function ball(bone, name, d, off, col, mat) return P(bone, name, Vector3.new(d, d, d), off, col, mat, Enum.PartType.Ball) end
	-- a cylinder whose long axis runs along the bone's Y axis
	local function tube(bone, name, len, d, off, col, mat)
		return P(bone, name, Vector3.new(len, d, d), off, col, mat, Enum.PartType.Cylinder, CFrame.Angles(0, 0, math.pi / 2))
	end
	local function szOf(bone) return B[bone] and B[bone].Size or Vector3.new(1, 1, 1) end
	local rz = function(d) return CFrame.Angles(0, 0, math.rad(d)) end
	local rx = function(d) return CFrame.Angles(math.rad(d), 0, 0) end
	local ry = function(d) return CFrame.Angles(0, math.rad(d), 0) end

	-- ---------------- torso / waist ----------------
	local ts = szOf("torso")
	local hasLower = B.lower ~= nil and B.lower ~= B.torso
	if hasLower then
		P("torso", "OpChest", Vector3.new(ts.X * 1.04, ts.Y * 1.02, ts.Z * 1.08), Vector3.zero, top, FAB)
		local lsz = szOf("lower")
		P("lower", "OpWaist", Vector3.new(lsz.X * 1.04, lsz.Y * 1.06, lsz.Z * 1.08), Vector3.zero, pants, FAB)
	else
		P("torso", "OpChest", Vector3.new(ts.X * 1.04, ts.Y * 0.7, ts.Z * 1.08), Vector3.new(0, ts.Y * 0.15, 0), top, FAB)
		P("torso", "OpWaist", Vector3.new(ts.X * 1.04, ts.Y * 0.32, ts.Z * 1.08), Vector3.new(0, -ts.Y * 0.35, 0), pants, FAB)
	end
	tube("torso", "OpNeck", 0.5, 0.52, Vector3.new(0, ts.Y * 0.5 + 0.02, 0), L.bareNeck == false and top or skin, PLA)
	local beltAt = hasLower and { "lower", Vector3.new(0, szOf("lower").Y * 0.2, 0), szOf("lower") } or { "torso", Vector3.new(0, -ts.Y * 0.33, 0), ts }
	P(beltAt[1], "OpBelt", Vector3.new(beltAt[3].X * 1.08, 0.3, beltAt[3].Z * 1.12), beltAt[2], dark, FAB)
	P(beltAt[1], "OpBuckle", Vector3.new(0.34, 0.24, 0.08), beltAt[2] + Vector3.new(0, 0, -(beltAt[3].Z * 0.56 + 0.02)), L.buckle or acc, NEON)
	if teamCol then
		P("torso", "OpTeamTag", Vector3.new(ts.X * 0.5, 0.14, 0.08), Vector3.new(0, ts.Y * 0.42, -(ts.Z * 0.54 + 0.2)), teamCol, NEON)
	end
	local vest = L.vest or "plate"
	local vestC = top:Lerp(RGB(8, 10, 14), 0.5)
	if vest == "plate" or vest == "rig" then
		local pz = ts.Z * 0.5 + 0.14
		if vest == "plate" then
			P("torso", "OpPlateF", Vector3.new(ts.X * 1.08, ts.Y * 0.84, 0.3), Vector3.new(0, ts.Y * 0.03, -pz), vestC, FAB)
			P("torso", "OpPlateFCore", Vector3.new(ts.X * 0.7, ts.Y * 0.56, 0.12), Vector3.new(0, ts.Y * 0.08, -pz - 0.2), (typeof(L.plate) == "Color3" and L.plate or nil) or top:Lerp(RGB(40, 42, 48), 0.6), MET)
			P("torso", "OpPlateB", Vector3.new(ts.X * 1.08, ts.Y * 0.84, 0.3), Vector3.new(0, ts.Y * 0.03, pz), vestC, FAB)
			P("torso", "OpStripe", Vector3.new(ts.X * 0.5, ts.Y * 0.07, 0.08), Vector3.new(0, ts.Y * 0.33, -pz - 0.28), acc, NEON)
			for _, sx in ipairs({ -1, 1 }) do P("torso", "OpStrap", Vector3.new(ts.X * 0.16, ts.Y * 0.2, ts.Z + 0.36), Vector3.new(sx * ts.X * 0.3, ts.Y * 0.43, 0), vestC, FAB) end
		else
			P("torso", "OpRig", Vector3.new(ts.X * 1.06, ts.Y * 0.56, 0.22), Vector3.new(0, -ts.Y * 0.08, -pz + 0.02), vestC, FAB)
			for _, sx in ipairs({ -1, 1 }) do P("torso", "OpStrap", Vector3.new(ts.X * 0.14, ts.Y * 0.2, ts.Z + 0.3), Vector3.new(sx * ts.X * 0.32, ts.Y * 0.43, 0), vestC, FAB) end
		end
		for k = -1, 1 do P("torso", "OpMagPouch", Vector3.new(ts.X * 0.24, ts.Y * 0.3, 0.22), Vector3.new(k * ts.X * 0.3, -ts.Y * 0.2, -pz - (vest == "plate" and 0.22 or 0.12)), dark, FAB) end
		P("torso", "OpPack", Vector3.new(ts.X * 0.72, ts.Y * 0.64, 0.42), Vector3.new(0, -ts.Y * 0.02, pz + 0.3), dark, FAB)
		P("torso", "OpAntenna", Vector3.new(0.08, ts.Y * 0.9, 0.08), Vector3.new(ts.X * 0.28, ts.Y * 0.6, pz + 0.4), RGB(12, 12, 14), MET)
		P("torso", "OpPackLight", Vector3.new(ts.X * 0.18, 0.1, 0.06), Vector3.new(0, ts.Y * 0.1, pz + 0.52), acc, NEON)
	elseif vest == "hoodie" then
		P("torso", "OpPocket", Vector3.new(ts.X * 0.7, ts.Y * 0.26, 0.14), Vector3.new(0, -ts.Y * 0.28, -(ts.Z * 0.5 + 0.1)), top:Lerp(RGB(0, 0, 0), 0.15), FAB)
		P("torso", "OpHoodBack", Vector3.new(ts.X * 0.7, ts.Y * 0.34, 0.4), Vector3.new(0, ts.Y * 0.4, ts.Z * 0.5 + 0.18), top:Lerp(RGB(0, 0, 0), 0.1), FAB)
		for _, sx in ipairs({ -1, 1 }) do P("torso", "OpString", Vector3.new(0.06, ts.Y * 0.3, 0.06), Vector3.new(sx * 0.18, ts.Y * 0.28, -(ts.Z * 0.5 + 0.06)), RGB(235, 235, 235), FAB) end
	elseif vest == "parka" then
		P("torso", "OpFurRing", Vector3.new(ts.X * 1.1, 0.5, ts.Z * 1.3), Vector3.new(0, ts.Y * 0.46, 0), L.fur or RGB(245, 247, 250), FAB)
		P("torso", "OpZip", Vector3.new(0.1, ts.Y * 0.9, 0.06), Vector3.new(0, 0, -(ts.Z * 0.54 + 0.01)), dark, MET)
		for _, sx in ipairs({ -1, 1 }) do P("torso", "OpPocket", Vector3.new(ts.X * 0.3, ts.Y * 0.26, 0.14), Vector3.new(sx * ts.X * 0.28, -ts.Y * 0.26, -(ts.Z * 0.5 + 0.1)), top:Lerp(RGB(0, 0, 0), 0.15), FAB) end
	elseif vest == "jacket" then
		P("torso", "OpZip", Vector3.new(0.12, ts.Y * 0.95, 0.07), Vector3.new(0, 0, -(ts.Z * 0.54 + 0.01)), L.trim or acc, NEON)
		for _, sx in ipairs({ -1, 1 }) do
			P("torso", "OpLapel", Vector3.new(ts.X * 0.2, ts.Y * 0.4, 0.1), Vector3.new(sx * ts.X * 0.2, ts.Y * 0.3, -(ts.Z * 0.54 + 0.04)), top:Lerp(RGB(255, 255, 255), 0.12), FAB, nil, rz(sx * 14))
		end
		P("torso", "OpShirt", Vector3.new(ts.X * 0.3, ts.Y * 0.8, 0.06), Vector3.new(0, 0, -(ts.Z * 0.54 - 0.01)), L.under or RGB(230, 230, 235), FAB)
	elseif vest == "suit" then
		P("torso", "OpSuitStripe", Vector3.new(0.3, ts.Y * 1.02, ts.Z * 1.1), Vector3.zero, L.trim or acc, PLA)
		P("torso", "OpSuitBadge", Vector3.new(0.5, 0.5, 0.06), Vector3.new(ts.X * 0.25, ts.Y * 0.22, -(ts.Z * 0.54 + 0.02)), RGB(250, 250, 250), PLA)
		P("torso", "OpSuitBadge2", Vector3.new(0.34, 0.34, 0.07), Vector3.new(ts.X * 0.25, ts.Y * 0.22, -(ts.Z * 0.54 + 0.04)), L.trim or acc, NEON)
	elseif vest == "armor" then
		P("torso", "OpArmor", Vector3.new(ts.X * 1.14, ts.Y * 0.9, 0.4), Vector3.new(0, ts.Y * 0.05, -(ts.Z * 0.5 + 0.18)), (typeof(L.plate) == "Color3" and L.plate or nil) or top:Lerp(RGB(60, 64, 72), 0.5), MET)
		P("torso", "OpArmorGlow", Vector3.new(ts.X * 0.8, 0.1, 0.1), Vector3.new(0, ts.Y * 0.22, -(ts.Z * 0.5 + 0.42)), acc, NEON)
		P("torso", "OpArmorB", Vector3.new(ts.X * 1.14, ts.Y * 0.9, 0.4), Vector3.new(0, ts.Y * 0.05, ts.Z * 0.5 + 0.18), (typeof(L.plate) == "Color3" and L.plate or nil) or top:Lerp(RGB(60, 64, 72), 0.5), MET)
	end
	if L.cape then
		P("torso", "OpCape", Vector3.new(ts.X * 1.0, ts.Y * 1.9, 0.12), Vector3.new(0, -ts.Y * 0.4, ts.Z * 0.5 + 0.5), L.cape, FAB)
	end
	if L.chain then
		P("torso", "OpChain", Vector3.new(ts.X * 0.5, 0.1, 0.1), Vector3.new(0, ts.Y * 0.3, -(ts.Z * 0.54 + 0.08)), RGB(240, 200, 70), MET)
		P("torso", "OpChainDrop", Vector3.new(0.14, ts.Y * 0.34, 0.1), Vector3.new(0, ts.Y * 0.1, -(ts.Z * 0.54 + 0.08)), RGB(240, 200, 70), MET)
		P("torso", "OpMedal", Vector3.new(0.36, 0.36, 0.08), Vector3.new(0, -ts.Y * 0.08, -(ts.Z * 0.54 + 0.1)), RGB(255, 215, 90), MET, Enum.PartType.Cylinder, ry(90))
	end
	if L.tail then -- lizard / animal tail
		tube(hasLower and "lower" or "torso", "OpTail", 1.8, 0.34, Vector3.new(0, -0.7, 0.9), L.tail, FAB)
	end

	if L.tufts then -- ghillie / fur tufts
		local rng = Random.new(5)
		for i = 1, 14 do
			local a = rng:NextNumber(0, math.pi * 2)
			P("torso", "OpTuft", Vector3.new(rng:NextNumber(0.3, 0.6), rng:NextNumber(0.5, 1.0), 0.14), Vector3.new(math.cos(a) * ts.X * 0.55, rng:NextNumber(-ts.Y * 0.3, ts.Y * 0.5), math.sin(a) * ts.Z * 0.62),
				(i % 2 == 0) and L.tufts or L.tufts:Lerp(RGB(20, 30, 14), 0.4), FAB, nil, CFrame.Angles(0, a, rng:NextNumber(-0.5, 0.5)))
		end
	end
	-- ---------------- arms ----------------
	for _, side in ipairs({ "L", "R" }) do
		local sx = (side == "L") and -1 or 1
		local ua, fa, ha = B["uarm" .. side], B["farm" .. side], B["hand" .. side]
		local sleeve = L.sleeve or top
		local fore = L.shortSleeves and skin or sleeve
		if ua and fa and ua ~= fa then
			local d = math.min(ua.Size.X, ua.Size.Z) * 0.86
			tube("uarm" .. side, "OpSleeve", ua.Size.Y * 1.02, d, Vector3.zero, sleeve, FAB)
			ball("uarm" .. side, "OpShoulderBall", d * 1.1, Vector3.new(0, ua.Size.Y * 0.4, 0), sleeve, FAB)
			tube("farm" .. side, "OpForearm", fa.Size.Y * 1.02, d * 0.92, Vector3.zero, fore, FAB)
			ball("farm" .. side, "OpElbow", d * 1.0, Vector3.new(0, fa.Size.Y * 0.5, 0), sleeve, FAB)
			if ha then
				local gh = math.max(0.36, ha.Size.Y * 1.5)
				P("hand" .. side, "OpGlove", Vector3.new(d * 0.98, gh, d * 1.04), Vector3.new(0, -0.02, 0), L.bareHands and skin or glove, FAB)
				P("hand" .. side, "OpKnuckle", Vector3.new(d * 0.96, 0.1, 0.12), Vector3.new(0, -gh * 0.45, -d * 0.5), L.bareHands and skin or dark, FAB)
			end
			if L.pads ~= false and vest ~= "hoodie" then
				P("uarm" .. side, "OpShoulder", Vector3.new(ua.Size.X + 0.2, 0.46, ua.Size.Z + 0.2), Vector3.new(0, ua.Size.Y * 0.36, 0), L.padCol or top:Lerp(RGB(30, 32, 38), 0.55), MET)
				P("farm" .. side, "OpElbowPad", Vector3.new(fa.Size.X + 0.1, 0.34, fa.Size.Z + 0.12), Vector3.new(0, fa.Size.Y * 0.38, 0.03), dark, FAB)
			end
			if side == "L" and teamCol then P("uarmL", "OpTeamBand", Vector3.new(ua.Size.X + 0.08, 0.22, ua.Size.Z + 0.08), Vector3.new(0, -ua.Size.Y * 0.05, 0), teamCol, NEON) end
		elseif ua then
			local h, d = ua.Size.Y, math.min(ua.Size.X, ua.Size.Z) * 0.9
			tube("uarm" .. side, "OpSleeve", h * 0.64, d, Vector3.new(0, h * 0.18, 0), sleeve, FAB)
			ball("uarm" .. side, "OpShoulderBall", d * 1.12, Vector3.new(0, h * 0.5 - d * 0.2, 0), sleeve, FAB)
			tube("uarm" .. side, "OpForearm", h * 0.36, d * 0.9, Vector3.new(0, -h * 0.16, 0), fore, FAB)
			P("uarm" .. side, "OpGlove", Vector3.new(d * 1.0, h * 0.22, d * 1.06), Vector3.new(0, -h * 0.4, 0), L.bareHands and skin or glove, FAB)
			if L.pads ~= false and vest ~= "hoodie" then
				P("uarm" .. side, "OpShoulder", Vector3.new(ua.Size.X + 0.24, 0.46, ua.Size.Z + 0.24), Vector3.new(0, h * 0.38, 0), L.padCol or top:Lerp(RGB(30, 32, 38), 0.55), MET)
				P("uarm" .. side, "OpElbowPad", Vector3.new(ua.Size.X + 0.12, 0.34, ua.Size.Z + 0.14), Vector3.new(0, -h * 0.02, 0.03), dark, FAB)
			end
			if side == "L" and teamCol then P("uarmL", "OpTeamBand", Vector3.new(ua.Size.X + 0.1, 0.22, ua.Size.Z + 0.1), Vector3.new(0, h * 0.18, 0), teamCol, NEON) end
		end
	end

	-- ---------------- legs ----------------
	for _, side in ipairs({ "L", "R" }) do
		local ul, ll, ft = B["uleg" .. side], B["lleg" .. side], B["foot" .. side]
		if ul and ll and ul ~= ll then
			local d = math.min(ul.Size.X, ul.Size.Z) * 0.9
			tube("uleg" .. side, "OpThigh", ul.Size.Y * 1.03, d, Vector3.zero, pants, FAB)
			tube("lleg" .. side, "OpShin", ll.Size.Y * 1.03, d * 0.88, Vector3.zero, pants, FAB)
			ball("lleg" .. side, "OpKneeBall", d * 0.98, Vector3.new(0, ll.Size.Y * 0.5, 0), pants, FAB)
			if L.pads ~= false then P("lleg" .. side, "OpKnee", Vector3.new(ll.Size.X + 0.08, 0.42, 0.26), Vector3.new(0, ll.Size.Y * 0.34, -(ll.Size.Z * 0.5 + 0.06)), dark, FAB) end
			P("uleg" .. side, "OpCargo", Vector3.new(0.14, ul.Size.Y * 0.4, ul.Size.Z * 0.6), Vector3.new((side == "L" and -1 or 1) * (ul.Size.X * 0.5 + 0.06), 0, 0), pants:Lerp(RGB(0, 0, 0), 0.15), FAB)
			if ft then
				P("foot" .. side, "OpBoot", Vector3.new(ft.Size.X * 1.06, math.max(0.5, ft.Size.Y * 1.8), ft.Size.Z * 1.28), Vector3.new(0, ft.Size.Y * 0.3, -ft.Size.Z * 0.06), boots, FAB)
				P("foot" .. side, "OpSole", Vector3.new(ft.Size.X * 1.1, 0.12, ft.Size.Z * 1.32), Vector3.new(0, -ft.Size.Y * 0.5, -ft.Size.Z * 0.06), RGB(10, 10, 12), PLA)
				P("lleg" .. side, "OpBootCuff", Vector3.new(ll.Size.X + 0.1, 0.3, ll.Size.Z + 0.1), Vector3.new(0, -ll.Size.Y * 0.4, 0), boots, FAB)
			end
		elseif ul then
			local h, d = ul.Size.Y, math.min(ul.Size.X, ul.Size.Z) * 0.92
			tube("uleg" .. side, "OpThigh", h * 0.72, d, Vector3.new(0, h * 0.14, 0), pants, FAB)
			ball("uleg" .. side, "OpKneeBall", d * 1.0, Vector3.new(0, -h * 0.22, 0), pants, FAB)
			tube("uleg" .. side, "OpShin", h * 0.42, d * 0.9, Vector3.new(0, -h * 0.3, 0), pants, FAB)
			if L.pads ~= false then P("uleg" .. side, "OpKnee", Vector3.new(ul.Size.X + 0.1, 0.42, 0.28), Vector3.new(0, -h * 0.2, -(ul.Size.Z * 0.5 + 0.06)), dark, FAB) end
			P("uleg" .. side, "OpBoot", Vector3.new(ul.Size.X * 1.08, h * 0.28, ul.Size.Z * 1.3), Vector3.new(0, -h * 0.36, -ul.Size.Z * 0.1), boots, FAB)
			P("uleg" .. side, "OpSole", Vector3.new(ul.Size.X * 1.1, 0.12, ul.Size.Z * 1.34), Vector3.new(0, -h * 0.5 + 0.05, -ul.Size.Z * 0.1), RGB(10, 10, 12), PLA)
		end
	end
	if L.holster then
		local ur = B.ulegR or B.uleg
		if ur then P(B.ulegR and "ulegR" or "uleg", "OpHolster", Vector3.new(0.3, ur.Size.Y * 0.5, ur.Size.Z * 0.7), Vector3.new(ur.Size.X * 0.5 + 0.12, 0, 0), dark, FAB) end
	end

	-- ---------------- head ----------------
	local hd = B.head
	if hd then
		local hs = math.min(hd.Size.X, hd.Size.Y, hd.Size.Z)
		if hs < 0.9 then hs = 1.1 end
		local c = { hs = hs, skin = skin, hair = hairC, acc = acc, L = L, top = top, dark = dark }
		function c.H(name, size, off, col, mat, shape, rot) return P("head", name, size, off, col, mat, shape, rot) end
		function c.ball(name, d, off, col, mat) return ball("head", name, d, off, col, mat) end
		function c.tube(name, len, d, off, col, mat) return tube("head", name, len, d, off, col, mat) end
		function c.headBall(col) return ball("head", "OpHead", hs * 0.98, Vector3.zero, col or skin, PLA) end
		-- eyes / brows / nose / mouth on a skin face
		function c.face(o)
			o = o or {}
			for _, sx in ipairs({ -1, 1 }) do
				c.H("OpEyeWhite", Vector3.new(hs * 0.2, hs * 0.11, 0.05), Vector3.new(sx * hs * 0.2, hs * 0.06, -hs * 0.45), o.white or RGB(245, 245, 245), PLA)
				c.H("OpPupil", Vector3.new(hs * 0.09, hs * 0.1, 0.05), Vector3.new(sx * hs * 0.2, hs * 0.06, -hs * 0.475), o.eye or RGB(30, 24, 20), o.glow and NEON or PLA)
				if not o.nobrow then c.H("OpBrow", Vector3.new(hs * 0.25, hs * 0.05, 0.06), Vector3.new(sx * hs * 0.2, hs * 0.2, -hs * 0.43), o.brow or hairC, PLA, nil, rz(sx * -6)) end
			end
			if not o.nonose then c.H("OpNose", Vector3.new(hs * 0.1, hs * 0.2, hs * 0.1), Vector3.new(0, -hs * 0.04, -hs * 0.48), o.nose or skin:Lerp(RGB(120, 70, 50), 0.15), PLA) end
			if not o.nomouth then c.H("OpMouth", Vector3.new(hs * 0.3, hs * 0.04, 0.05), Vector3.new(0, -hs * 0.26, -hs * 0.44), o.mouth or RGB(110, 50, 48), PLA) end
		end
		function c.shades(col, glow)
			c.H("OpShades", Vector3.new(hs * 0.98, hs * 0.2, 0.1), Vector3.new(0, hs * 0.06, -hs * 0.48), col or RGB(10, 10, 14), glow and NEON or GLS)
			c.H("OpShadesBridge", Vector3.new(hs * 0.12, hs * 0.05, 0.1), Vector3.new(0, hs * 0.08, -hs * 0.5), RGB(10, 10, 14), MET)
		end
		function c.ears(col, inner, h, spread, tilt)
			for _, sx in ipairs({ -1, 1 }) do
				c.H("OpEar", Vector3.new(h, hs * 0.26, hs * 0.1), Vector3.new(sx * spread, hs * 0.5 + h * 0.5, hs * 0.1), col, FAB, Enum.PartType.Cylinder, rz(90 + sx * tilt))
				c.H("OpEarIn", Vector3.new(h * 0.8, hs * 0.14, hs * 0.12), Vector3.new(sx * spread, hs * 0.5 + h * 0.5, hs * 0.05), inner, FAB, Enum.PartType.Cylinder, rz(90 + sx * tilt))
			end
		end
		function c.tacHelmet(col, big)
			c.ball("OpHelmet", hs * (big or 1.4), Vector3.new(0, hs * 0.33, hs * 0.04), col, MET)
			c.H("OpHelmetBrim", Vector3.new(hs * 1.3, hs * 0.12, hs * 1.3), Vector3.new(0, hs * 0.18, -hs * 0.02), dark, MET)
			c.H("OpHelmetRail", Vector3.new(hs * 0.18, hs * 0.1, hs * 1.1), Vector3.new(0, hs * 0.98, 0), dark, MET)
			c.H("OpNvg", Vector3.new(hs * 0.32, hs * 0.3, hs * 0.34), Vector3.new(0, hs * 0.72, -hs * 0.62), dark, MET)
			c.H("OpNvgLens", Vector3.new(hs * 0.2, hs * 0.2, hs * 0.04), Vector3.new(0, hs * 0.72, -hs * 0.8), acc, NEON)
			for _, sx in ipairs({ -1, 1 }) do c.H("OpEarCup", Vector3.new(hs * 0.14, hs * 0.5, hs * 0.5), Vector3.new(sx * hs * 0.58, hs * 0.1, hs * 0.05), dark, MET) end
		end
		function c.visor(w)
			c.H("OpVisorFrame", Vector3.new(hs * (w or 1.1), hs * 0.34, 0.1), Vector3.new(0, hs * 0.05, -hs * 0.49), dark, MET)
			c.H("OpVisor", Vector3.new(hs * ((w or 1.1) - 0.08), hs * 0.26, 0.14), Vector3.new(0, hs * 0.05, -hs * 0.52), acc, NEON)
		end
		function c.respirator()
			c.H("OpRespirator", Vector3.new(hs * 0.74, hs * 0.38, hs * 0.2), Vector3.new(0, -hs * 0.27, -hs * 0.5), dark, MET)
			for _, sx in ipairs({ -1, 1 }) do c.H("OpFilter", Vector3.new(hs * 0.3, hs * 0.3, hs * 0.2), Vector3.new(sx * hs * 0.42, -hs * 0.28, -hs * 0.44), top:Lerp(RGB(0, 0, 0), 0.5), MET, Enum.PartType.Cylinder, ry(90)) end
		end
		local style = L.head or "helmet"
		local fn = HEADS and HEADS[style] or nil
		if fn then fn(c) else
			c.headBall(RGB(16, 17, 20))
			c.tacHelmet(top:Lerp(RGB(30, 32, 38), 0.65))
			c.visor()
			c.respirator()
		end
	end
end

-- finds the right rig parts on a real avatar (R15 or R6) or on a bot / preview rig and dresses it
local function dressCharacter(model, opId, anchored, teamCol)
	if not model then return end
	local function f(...)
		for _, n in ipairs({ ... }) do
			local x = model:FindFirstChild(n)
			if x and x:IsA("BasePart") then return x end
		end
	end
	local B = {
		head = f("Head"), torso = f("UpperTorso", "Torso"), lower = f("LowerTorso"),
		uarmL = f("LeftUpperArm", "Left Arm", "LeftArm"), uarmR = f("RightUpperArm", "Right Arm", "RightArm"),
		farmL = f("LeftLowerArm"), farmR = f("RightLowerArm"), handL = f("LeftHand"), handR = f("RightHand"),
		ulegL = f("LeftUpperLeg", "Left Leg", "LeftLeg"), ulegR = f("RightUpperLeg", "Right Leg", "RightLeg"),
		llegL = f("LeftLowerLeg"), llegR = f("RightLowerLeg"), footL = f("LeftFoot"), footR = f("RightFoot"),
	}
	if not B.torso then return end
	dressOperator(model, opId, anchored, B, teamCol)
end

do
local MET, PLA, NEON, FAB, GLS = Enum.Material.Metal, Enum.Material.SmoothPlastic, Enum.Material.Neon, Enum.Material.Fabric, Enum.Material.Glass
local rx = function(d) return CFrame.Angles(math.rad(d), 0, 0) end
local ry = function(d) return CFrame.Angles(0, math.rad(d), 0) end
local rz = function(d) return CFrame.Angles(0, 0, math.rad(d)) end
HEADS.helmet = function(c)
	c.headBall(RGB(16, 17, 20))
	c.tacHelmet(c.top:Lerp(RGB(30, 32, 38), 0.65))
	c.visor()
	c.respirator()
end
HEADS.cap = function(c)
	local hs = c.hs
	c.headBall(c.skin)
	c.face()
	c.tube("OpCap", hs * 0.36, hs * 1.14, Vector3.new(0, hs * 0.42, 0), c.top, FAB)
	c.H("OpBrim", Vector3.new(hs * 0.95, 0.1, hs * 0.6), Vector3.new(0, hs * 0.3, -hs * 0.7), c.top:Lerp(RGB(0, 0, 0), 0.3), FAB)
	for _, sx in ipairs({ -1, 1 }) do c.ball("OpGoggle", hs * 0.3, Vector3.new(sx * hs * 0.26, hs * 0.5, -hs * 0.42), c.acc, NEON) end
	c.H("OpGoggleStrap", Vector3.new(hs * 1.1, hs * 0.1, hs * 1.1), Vector3.new(0, hs * 0.38, 0), c.dark, FAB)
end
HEADS.visorhelm = function(c)
	local hs = c.hs
	c.headBall(c.dark)
	c.ball("OpHelmet", hs * 1.5, Vector3.new(0, hs * 0.12, hs * 0.03), c.top:Lerp(RGB(60, 64, 72), 0.5), MET)
	c.H("OpVisor", Vector3.new(hs * 1.0, hs * 0.5, 0.14), Vector3.new(0, hs * 0.02, -hs * 0.66), c.acc, NEON)
	c.H("OpChin", Vector3.new(hs * 0.8, hs * 0.3, hs * 0.3), Vector3.new(0, -hs * 0.4, -hs * 0.5), c.dark, MET)
	c.H("OpCrest", Vector3.new(hs * 0.14, hs * 0.2, hs * 1.2), Vector3.new(0, hs * 0.82, 0), c.acc, NEON)
end
HEADS.hoodantler = function(c)
	local hs = c.hs
	c.headBall(c.skin)
	c.ball("OpHood", hs * 1.38, Vector3.new(0, hs * 0.04, hs * 0.24), c.top, FAB)
	c.H("OpMaskLow", Vector3.new(hs * 0.94, hs * 0.5, hs * 0.2), Vector3.new(0, -hs * 0.26, -hs * 0.42), c.dark, FAB)
	c.H("OpEyes", Vector3.new(hs * 0.86, hs * 0.12, 0.06), Vector3.new(0, hs * 0.08, -hs * 0.46), c.acc, NEON)
	for _, sx in ipairs({ -1, 1 }) do
		c.H("OpAntler", Vector3.new(hs * 1.1, hs * 0.14, hs * 0.14), Vector3.new(sx * hs * 0.55, hs * 0.95, hs * 0.1), RGB(214, 204, 184), PLA, Enum.PartType.Cylinder, rz(90 - sx * 22))
		c.H("OpAntler2", Vector3.new(hs * 0.6, hs * 0.1, hs * 0.1), Vector3.new(sx * hs * 0.82, hs * 1.12, hs * 0.1), RGB(214, 204, 184), PLA, Enum.PartType.Cylinder, rz(90 - sx * 60))
	end
end
HEADS.wrap = function(c)
	local hs = c.hs
	c.headBall(c.skin)
	c.face()
	c.ball("OpWrap", hs * 1.1, Vector3.new(0, hs * 0.3, hs * 0.04), c.dark, FAB)
	c.H("OpCloth", Vector3.new(hs * 0.98, hs * 0.4, hs * 0.5), Vector3.new(0, -hs * 0.3, -hs * 0.22), c.top:Lerp(RGB(0, 0, 0), 0.4), FAB)
	c.H("OpMonocleArm", Vector3.new(0.1, 0.1, hs * 0.5), Vector3.new(hs * 0.38, hs * 0.18, -hs * 0.6), c.dark, MET)
	c.ball("OpMonocle", hs * 0.36, Vector3.new(hs * 0.38, hs * 0.1, -hs * 0.84), c.acc, NEON)
end
HEADS.beret = function(c)
	local hs = c.hs
	c.headBall(c.skin)
	c.face()
	c.H("OpBeret", Vector3.new(hs * 0.3, hs * 1.3, hs * 1.3), Vector3.new(hs * 0.08, hs * 0.5, 0), c.top, FAB, Enum.PartType.Cylinder, rz(98))
	c.H("OpBadge", Vector3.new(hs * 0.22, hs * 0.22, 0.08), Vector3.new(-hs * 0.3, hs * 0.5, -hs * 0.6), RGB(255, 210, 80), NEON)
	c.H("OpScar", Vector3.new(0.06, hs * 0.4, 0.05), Vector3.new(hs * 0.2, -hs * 0.06, -hs * 0.46), RGB(170, 80, 80), PLA, nil, rz(20))
end
HEADS.horned = function(c)
	local hs = c.hs
	c.headBall(c.dark)
	c.tacHelmet(c.top:Lerp(RGB(30, 32, 38), 0.55))
	c.visor()
	c.respirator()
	for _, sx in ipairs({ -1, 1 }) do
		c.H("OpHorn", Vector3.new(hs * 1.5, hs * 0.28, hs * 0.28), Vector3.new(sx * hs * 0.8, hs * 1.15, 0), c.acc, NEON, Enum.PartType.Cylinder, rz(90 - sx * 28))
		c.H("OpHornTip", Vector3.new(hs * 0.7, hs * 0.18, hs * 0.18), Vector3.new(sx * hs * 1.3, hs * 1.78, 0), c.acc, NEON, Enum.PartType.Cylinder, rz(90 - sx * 52))
	end
end
HEADS.hoodmask = function(c)
	local hs = c.hs
	c.headBall(c.dark)
	c.ball("OpHood", hs * 1.4, Vector3.new(0, hs * 0.08, hs * 0.22), c.top:Lerp(RGB(0, 0, 0), 0.4), FAB)
	c.H("OpHoodPeak", Vector3.new(hs * 1.2, hs * 0.3, hs * 0.6), Vector3.new(0, hs * 0.55, -hs * 0.3), c.top:Lerp(RGB(0, 0, 0), 0.4), FAB, nil, rx(-25))
	c.H("OpEyes", Vector3.new(hs * 0.9, hs * 0.12, 0.06), Vector3.new(0, hs * 0.08, -hs * 0.47), c.acc, NEON)
	c.H("OpMaskLow", Vector3.new(hs * 0.94, hs * 0.45, hs * 0.18), Vector3.new(0, -hs * 0.26, -hs * 0.44), c.dark, MET)
end
HEADS.flame = function(c)
	local hs = c.hs
	HEADS.helmet(c)
	for k = -1, 1 do
		c.H("OpFlame", Vector3.new(hs * 0.2, hs * (0.9 - math.abs(k) * 0.25), hs * 0.7), Vector3.new(k * hs * 0.3, hs * (1.15 - math.abs(k) * 0.1), hs * 0.05), c.acc:Lerp(RGB(255, 230, 120), math.abs(k) * 0.5), NEON, nil, rz(k * 14))
	end
end
HEADS.arctic = function(c)
	local hs = c.hs
	c.headBall(c.dark)
	c.tacHelmet(RGB(226, 234, 246))
	c.visor()
	c.respirator()
	for _, sx in ipairs({ -1, 1 }) do
		local cr = c.H("OpCrystal", Vector3.new(hs * 0.3, hs * 0.9, hs * 0.3), Vector3.new(sx * hs * 0.8, hs * 1.0, 0), c.acc, Enum.Material.Ice, nil, rz(-sx * 20))
		if cr then cr.Transparency = 0.15 end
	end
end
HEADS.gasmask = function(c)
	local hs = c.hs
	c.headBall(c.dark)
	c.tacHelmet(c.top:Lerp(RGB(30, 32, 38), 0.6))
	for _, sx in ipairs({ -1, 1 }) do
		c.ball("OpGoggle", hs * 0.42, Vector3.new(sx * hs * 0.26, hs * 0.06, -hs * 0.5), c.acc, NEON)
		c.H("OpFilter", Vector3.new(hs * 0.4, hs * 0.4, hs * 0.4), Vector3.new(sx * hs * 0.5, -hs * 0.3, -hs * 0.5), c.dark, MET, Enum.PartType.Cylinder, ry(90))
		c.H("OpFilterRing", Vector3.new(hs * 0.44, hs * 0.1, hs * 0.1), Vector3.new(sx * hs * 0.5, -hs * 0.3, -hs * 0.7), c.acc, NEON, Enum.PartType.Cylinder, ry(90))
	end
	c.H("OpMask", Vector3.new(hs * 0.9, hs * 0.5, hs * 0.3), Vector3.new(0, -hs * 0.28, -hs * 0.5), c.dark, MET)
end
HEADS.halo = function(c)
	local hs = c.hs
	HEADS.helmet(c)
	c.H("OpHalo", Vector3.new(hs * 0.12, hs * 1.9, hs * 1.9), Vector3.new(0, hs * 1.3, 0), c.acc, NEON, Enum.PartType.Cylinder, rz(90))
	for _, sx in ipairs({ -1, 1 }) do
		c.H("OpAntenna", Vector3.new(0.06, hs * 0.9, 0.06), Vector3.new(sx * hs * 0.45, hs * 1.0, hs * 0.2), c.dark, MET)
		c.ball("OpAntennaTip", hs * 0.2, Vector3.new(sx * hs * 0.45, hs * 1.5, hs * 0.2), c.acc, NEON)
	end
end
HEADS.kabuto = function(c)
	local hs = c.hs
	c.headBall(c.dark)
	c.ball("OpHelmet", hs * 1.42, Vector3.new(0, hs * 0.3, hs * 0.04), c.top:Lerp(RGB(40, 10, 14), 0.4), MET)
	c.H("OpCrest", Vector3.new(hs * 0.1, hs * 0.9, hs * 0.6), Vector3.new(0, hs * 1.15, -hs * 0.1), c.acc, NEON)
	c.H("OpMenpo", Vector3.new(hs * 0.94, hs * 0.5, hs * 0.3), Vector3.new(0, -hs * 0.24, -hs * 0.48), c.top:Lerp(RGB(60, 8, 12), 0.5), MET)
	c.H("OpEyes", Vector3.new(hs * 0.8, hs * 0.1, 0.06), Vector3.new(0, hs * 0.1, -hs * 0.5), c.acc, NEON)
	c.H("OpHelmBrim", Vector3.new(hs * 1.5, hs * 0.1, hs * 1.5), Vector3.new(0, hs * 0.2, 0), c.dark, MET)
end
HEADS.heavy = function(c)
	local hs = c.hs
	c.headBall(c.dark)
	c.H("OpHelm", Vector3.new(hs * 1.5, hs * 1.45, hs * 1.5), Vector3.new(0, hs * 0.08, hs * 0.03), c.top, MET)
	c.H("OpSlit", Vector3.new(hs * 1.1, hs * 0.16, 0.1), Vector3.new(0, hs * 0.1, -hs * 0.78), c.acc, NEON)
	c.H("OpVent", Vector3.new(hs * 0.5, hs * 0.3, 0.1), Vector3.new(0, -hs * 0.3, -hs * 0.78), c.dark, MET)
end
HEADS.goldhelm = function(c)
	local hs = c.hs
	HEADS.helmet(c)
	c.H("OpGoldCrest", Vector3.new(hs * 0.18, hs * 0.4, hs * 1.3), Vector3.new(0, hs * 1.05, 0), RGB(255, 205, 70), MET)
end
HEADS.racehelm = function(c)
	local hs = c.hs
	c.headBall(c.dark)
	c.ball("OpHelmet", hs * 1.5, Vector3.new(0, hs * 0.1, hs * 0.05), c.top, PLA)
	c.H("OpVisor", Vector3.new(hs * 1.05, hs * 0.5, 0.2), Vector3.new(0, hs * 0.06, -hs * 0.64), RGB(12, 12, 18), GLS)
	c.H("OpVisorGlint", Vector3.new(hs * 0.7, hs * 0.06, 0.22), Vector3.new(0, hs * 0.18, -hs * 0.66), c.acc, NEON)
	c.H("OpHelmStripe", Vector3.new(hs * 0.2, hs * 0.05, hs * 1.2), Vector3.new(0, hs * 0.84, 0), RGB(250, 250, 250), PLA)
	c.H("OpSpoiler", Vector3.new(hs * 0.9, hs * 0.1, hs * 0.3), Vector3.new(0, hs * 0.5, hs * 0.78), c.dark, PLA)
end
HEADS.ape = function(c)
	local hs = c.hs
	local fur = c.L.fur or RGB(176, 84, 30)
	local face = RGB(198, 150, 108)
	c.headBall(fur)
	c.ball("OpFur", hs * 1.22, Vector3.new(0, hs * 0.06, hs * 0.08), fur, FAB)
	c.ball("OpFace", hs * 0.86, Vector3.new(0, -hs * 0.02, -hs * 0.3), face, PLA)
	c.H("OpMuzzle", Vector3.new(hs * 0.62, hs * 0.36, hs * 0.3), Vector3.new(0, -hs * 0.26, -hs * 0.58), face:Lerp(RGB(90, 60, 40), 0.2), PLA)
	c.H("OpBrowRidge", Vector3.new(hs * 0.9, hs * 0.16, hs * 0.2), Vector3.new(0, hs * 0.22, -hs * 0.5), fur:Lerp(RGB(0, 0, 0), 0.4), FAB)
	for _, sx in ipairs({ -1, 1 }) do
		c.ball("OpEye", hs * 0.14, Vector3.new(sx * hs * 0.2, hs * 0.1, -hs * 0.66), RGB(24, 16, 10), PLA)
		c.ball("OpNostril", hs * 0.08, Vector3.new(sx * hs * 0.1, -hs * 0.2, -hs * 0.74), RGB(40, 24, 18), PLA)
		c.ball("OpEar", hs * 0.3, Vector3.new(sx * hs * 0.64, hs * 0.0, hs * 0.1), fur, FAB)
	end
	c.H("OpMouth", Vector3.new(hs * 0.4, hs * 0.05, 0.06), Vector3.new(0, -hs * 0.38, -hs * 0.72), RGB(60, 26, 24), PLA)
	c.H("OpCrownFur", Vector3.new(hs * 0.4, hs * 0.3, hs * 0.8), Vector3.new(0, hs * 0.66, 0), fur:Lerp(RGB(255, 140, 50), 0.2), FAB)
end
HEADS.skullmask = function(c)
	local hs = c.hs
	c.headBall(RGB(14, 14, 16))
	c.ball("OpCowl", hs * 1.34, Vector3.new(0, hs * 0.1, hs * 0.14), RGB(20, 20, 24), FAB)
	c.H("OpSkull", Vector3.new(hs * 0.8, hs * 0.74, 0.1), Vector3.new(0, hs * 0.06, -hs * 0.52), RGB(236, 232, 222), PLA)
	for _, sx in ipairs({ -1, 1 }) do
		c.H("OpSocket", Vector3.new(hs * 0.24, hs * 0.22, 0.12), Vector3.new(sx * hs * 0.2, hs * 0.12, -hs * 0.55), RGB(8, 8, 10), PLA)
		c.H("OpSocketGlow", Vector3.new(hs * 0.08, hs * 0.08, 0.13), Vector3.new(sx * hs * 0.2, hs * 0.12, -hs * 0.56), c.acc, NEON)
	end
	c.H("OpSkullNose", Vector3.new(hs * 0.1, hs * 0.14, 0.12), Vector3.new(0, -hs * 0.06, -hs * 0.55), RGB(8, 8, 10), PLA)
	c.H("OpJaw", Vector3.new(hs * 0.7, hs * 0.2, 0.1), Vector3.new(0, -hs * 0.24, -hs * 0.5), RGB(230, 226, 214), PLA)
	for k = -2, 2 do c.H("OpTeeth", Vector3.new(0.04, hs * 0.2, 0.12), Vector3.new(k * hs * 0.14, -hs * 0.24, -hs * 0.52), RGB(8, 8, 10), PLA) end
	c.H("OpCap", Vector3.new(hs * 1.1, hs * 0.2, hs * 1.1), Vector3.new(0, hs * 0.62, 0), RGB(20, 20, 24), FAB)
end
HEADS.lizard = function(c)
	local hs = c.hs
	local g = c.L.scale or RGB(96, 150, 56)
	c.headBall(g)
	c.H("OpSnout", Vector3.new(hs * 0.5, hs * 0.3, hs * 0.6), Vector3.new(0, -hs * 0.16, -hs * 0.56), g:Lerp(RGB(200, 220, 120), 0.2), PLA)
	c.H("OpMouth", Vector3.new(hs * 0.5, 0.04, hs * 0.5), Vector3.new(0, -hs * 0.3, -hs * 0.58), RGB(40, 24, 20), PLA)
	for _, sx in ipairs({ -1, 1 }) do
		c.ball("OpEye", hs * 0.34, Vector3.new(sx * hs * 0.3, hs * 0.3, -hs * 0.32), RGB(255, 220, 60), NEON)
		c.H("OpSlit", Vector3.new(0.05, hs * 0.26, 0.05), Vector3.new(sx * hs * 0.3, hs * 0.3, -hs * 0.5), RGB(12, 12, 10), PLA)
		c.ball("OpNostril", hs * 0.07, Vector3.new(sx * hs * 0.1, -hs * 0.08, -hs * 0.86), RGB(30, 40, 20), PLA)
	end
	for k = 0, 4 do c.H("OpCrestSpike", Vector3.new(hs * 0.1, hs * (0.3 - k * 0.03), hs * 0.2), Vector3.new(0, hs * 0.55, hs * (0.3 - k * 0.2)), g:Lerp(RGB(255, 200, 60), 0.4), PLA) end
	c.ball("OpHood", hs * 1.2, Vector3.new(0, hs * 0.14, hs * 0.2), c.top, FAB)
end
HEADS.explorer = function(c)
	local hs = c.hs
	c.headBall(c.skin)
	c.face()
	c.ball("OpHair", hs * 1.14, Vector3.new(0, hs * 0.14, hs * 0.1), c.hair, PLA)
	c.H("OpBangs", Vector3.new(hs * 1.0, hs * 0.2, hs * 0.2), Vector3.new(0, hs * 0.38, -hs * 0.4), c.hair, PLA)
	c.tube("OpBraid", hs * 1.6, hs * 0.2, Vector3.new(0, -hs * 0.7, hs * 0.58), c.hair, PLA)
	c.H("OpHeadband", Vector3.new(hs * 1.02, hs * 0.12, hs * 1.02), Vector3.new(0, hs * 0.38, 0), c.acc, FAB)
	c.H("OpGoggleStrap", Vector3.new(hs * 1.0, hs * 0.08, hs * 1.0), Vector3.new(0, hs * 0.58, 0), c.dark, FAB)
	for _, sx in ipairs({ -1, 1 }) do c.ball("OpGoggle", hs * 0.3, Vector3.new(sx * hs * 0.22, hs * 0.58, -hs * 0.36), RGB(180, 220, 255), GLS) end
end
HEADS.spacehelm = function(c)
	local hs = c.hs
	c.headBall(c.skin)
	c.face({ eye = RGB(60, 20, 90) })
	local gl = c.ball("OpGlass", hs * 1.7, Vector3.zero, RGB(170, 200, 255), GLS)
	if gl then gl.Transparency = 0.7 end
	c.H("OpRing", Vector3.new(hs * 0.22, hs * 1.3, hs * 1.3), Vector3.new(0, -hs * 0.62, 0), RGB(210, 214, 226), MET, Enum.PartType.Cylinder, rz(90))
	c.H("OpRingGlow", Vector3.new(hs * 0.06, hs * 1.36, hs * 1.36), Vector3.new(0, -hs * 0.5, 0), c.acc, NEON, Enum.PartType.Cylinder, rz(90))
	c.tube("OpAntenna", hs * 0.8, 0.07, Vector3.new(hs * 0.4, hs * 1.05, 0), c.dark, MET)
	c.ball("OpAntennaTip", hs * 0.2, Vector3.new(hs * 0.4, hs * 1.5, 0), c.acc, NEON)
end
HEADS.bunny = function(c)
	local hs = c.hs
	local fur = c.L.fur or RGB(240, 240, 244)
	c.headBall(fur)
	c.ball("OpMask", hs * 1.14, Vector3.new(0, 0, hs * 0.02), fur, FAB)
	c.ears(fur, c.L.inner or RGB(255, 170, 190), hs * 1.7, hs * 0.26, 7)
	for _, sx in ipairs({ -1, 1 }) do
		c.ball("OpEye", hs * 0.2, Vector3.new(sx * hs * 0.22, hs * 0.1, -hs * 0.5), RGB(20, 18, 22), PLA)
		c.ball("OpEyeShine", hs * 0.07, Vector3.new(sx * hs * 0.19, hs * 0.14, -hs * 0.6), RGB(255, 255, 255), NEON)
		c.ball("OpCheek", hs * 0.2, Vector3.new(sx * hs * 0.36, -hs * 0.14, -hs * 0.46), RGB(255, 170, 180), PLA)
		c.H("OpTooth", Vector3.new(hs * 0.1, hs * 0.16, 0.05), Vector3.new(sx * hs * 0.05, -hs * 0.34, -hs * 0.54), RGB(255, 255, 255), PLA)
	end
	c.ball("OpNose", hs * 0.14, Vector3.new(0, -hs * 0.08, -hs * 0.56), RGB(255, 140, 170), PLA)
end
HEADS.street = function(c)
	local hs = c.hs
	c.headBall(c.skin)
	c.face()
	c.tube("OpCap", hs * 0.34, hs * 1.14, Vector3.new(0, hs * 0.42, 0), c.top, FAB)
	c.H("OpBrim", Vector3.new(hs * 0.95, 0.1, hs * 0.6), Vector3.new(0, hs * 0.3, -hs * 0.7), c.top:Lerp(RGB(0, 0, 0), 0.3), FAB)
	c.H("OpBeard", Vector3.new(hs * 0.84, hs * 0.36, hs * 0.3), Vector3.new(0, -hs * 0.34, -hs * 0.32), c.hair, FAB)
	c.shades(RGB(10, 10, 14))
end
HEADS.capbeard = HEADS.street
HEADS.beanie = function(c)
	local hs = c.hs
	c.headBall(c.skin)
	c.face({ nomouth = true, nonose = true })
	c.ball("OpBeanie", hs * 1.1, Vector3.new(0, hs * 0.24, hs * 0.02), c.top, FAB)
	c.tube("OpBeanieBand", hs * 0.2, hs * 1.06, Vector3.new(0, hs * 0.14, 0), c.top:Lerp(RGB(255, 255, 255), 0.15), FAB)
	c.H("OpBandana", Vector3.new(hs * 1.0, hs * 0.44, hs * 0.5), Vector3.new(0, -hs * 0.28, -hs * 0.22), c.L.bandana or RGB(170, 30, 36), FAB)
	c.H("OpBandanaKnot", Vector3.new(hs * 0.3, hs * 0.3, hs * 0.2), Vector3.new(hs * 0.5, -hs * 0.3, hs * 0.3), c.L.bandana or RGB(170, 30, 36), FAB)
end
HEADS.hoodwhite = function(c)
	local hs = c.hs
	c.headBall(c.skin)
	c.face()
	c.ball("OpHood", hs * 1.4, Vector3.new(0, hs * 0.06, hs * 0.24), c.L.fur or RGB(244, 246, 250), FAB)
	c.shades(RGB(12, 12, 16))
	c.H("OpHeadband", Vector3.new(hs * 1.4, hs * 0.14, hs * 0.14), Vector3.new(0, hs * 0.5, 0), c.dark, MET, Enum.PartType.Cylinder, rz(90))
	for _, sx in ipairs({ -1, 1 }) do c.H("OpEarCup", Vector3.new(hs * 0.22, hs * 0.5, hs * 0.5), Vector3.new(sx * hs * 0.58, hs * 0.04, hs * 0.06), c.dark, MET) end
end
HEADS.pinkhair = function(c)
	local hs = c.hs
	local pink = c.L.hairCol or RGB(255, 90, 170)
	c.headBall(c.skin)
	c.face({ eye = RGB(40, 20, 30) })
	c.ball("OpHair", hs * 1.14, Vector3.new(0, hs * 0.14, hs * 0.12), pink, PLA)
	c.H("OpHairBack", Vector3.new(hs * 1.05, hs * 1.7, hs * 0.26), Vector3.new(0, -hs * 0.55, hs * 0.54), pink, PLA)
	c.H("OpBangs", Vector3.new(hs * 1.0, hs * 0.2, hs * 0.2), Vector3.new(0, hs * 0.38, -hs * 0.4), pink, PLA)
	c.H("OpLips", Vector3.new(hs * 0.26, hs * 0.07, 0.06), Vector3.new(0, -hs * 0.26, -hs * 0.46), RGB(220, 40, 90), PLA)
	c.H("OpEarring", Vector3.new(0.08, hs * 0.2, 0.08), Vector3.new(hs * 0.52, -hs * 0.1, 0), RGB(255, 215, 90), MET)
end
HEADS.shades = function(c)
	local hs = c.hs
	local blond = c.L.hairCol or RGB(236, 204, 130)
	c.headBall(c.skin)
	c.face({ nobrow = true })
	c.ball("OpHair", hs * 1.16, Vector3.new(0, hs * 0.16, hs * 0.1), blond, PLA)
	for _, sx in ipairs({ -1, 1 }) do c.H("OpHairSide", Vector3.new(hs * 0.2, hs * 0.8, hs * 0.7), Vector3.new(sx * hs * 0.5, -hs * 0.14, hs * 0.1), blond, PLA) end
	c.H("OpBangs", Vector3.new(hs * 1.0, hs * 0.18, hs * 0.2), Vector3.new(0, hs * 0.4, -hs * 0.4), blond, PLA)
	c.shades(RGB(14, 14, 18))
	c.H("OpLips", Vector3.new(hs * 0.26, hs * 0.07, 0.06), Vector3.new(0, -hs * 0.26, -hs * 0.46), RGB(200, 30, 50), PLA)
end
HEADS.zombie = function(c)
	local hs = c.hs
	local rot = c.L.skin or RGB(130, 150, 118)
	c.headBall(rot)
	for _, sx in ipairs({ -1, 1 }) do
		c.ball("OpEyeSocket", hs * 0.3, Vector3.new(sx * hs * 0.2, hs * 0.08, -hs * 0.36), RGB(30, 34, 28), PLA)
		c.ball("OpEye", hs * 0.2, Vector3.new(sx * hs * 0.2, hs * 0.08, -hs * 0.44), RGB(230, 255, 200), NEON)
	end
	c.H("OpJaw", Vector3.new(hs * 0.6, hs * 0.22, hs * 0.3), Vector3.new(0, -hs * 0.34, -hs * 0.38), RGB(60, 30, 30), PLA)
	for k = -2, 2 do c.H("OpTeeth", Vector3.new(hs * 0.07, hs * 0.1, 0.05), Vector3.new(k * hs * 0.11, -hs * 0.28, -hs * 0.52), RGB(220, 214, 190), PLA) end
	c.H("OpWound", Vector3.new(hs * 0.3, hs * 0.22, 0.06), Vector3.new(-hs * 0.3, hs * 0.3, -hs * 0.44), RGB(120, 20, 24), PLA)
	c.H("OpStitch", Vector3.new(hs * 0.36, 0.04, 0.07), Vector3.new(hs * 0.2, hs * 0.34, -hs * 0.45), RGB(20, 20, 20), PLA, nil, rz(-10))
	for k = 0, 3 do c.H("OpHairPatch", Vector3.new(hs * 0.16, hs * 0.2, hs * 0.16), Vector3.new((k - 1.5) * hs * 0.2, hs * 0.52, (k % 2) * hs * 0.1), RGB(50, 54, 46), PLA) end
	c.H("OpNose", Vector3.new(hs * 0.1, hs * 0.18, hs * 0.1), Vector3.new(0, -hs * 0.06, -hs * 0.48), rot:Lerp(RGB(60, 30, 30), 0.4), PLA)
end
HEADS.clown = function(c)
	local hs = c.hs
	local c1, c2 = c.L.hairCol or RGB(255, 60, 90), c.L.hairCol2 or RGB(60, 130, 255)
	c.headBall(RGB(246, 244, 240))
	for _, sx in ipairs({ -1, 1 }) do
		c.H("OpEyePaint", Vector3.new(hs * 0.24, hs * 0.34, 0.06), Vector3.new(sx * hs * 0.2, hs * 0.1, -hs * 0.45), sx < 0 and c1 or c2, PLA, nil, rz(sx * 45))
		c.ball("OpEye", hs * 0.1, Vector3.new(sx * hs * 0.2, hs * 0.1, -hs * 0.5), RGB(20, 20, 24), PLA)
		c.ball("OpPuff", hs * 0.7, Vector3.new(sx * hs * 0.62, hs * 0.34, hs * 0.04), sx < 0 and c1 or c2, FAB)
		c.H("OpSmile", Vector3.new(hs * 0.12, hs * 0.12, 0.06), Vector3.new(sx * hs * 0.34, -hs * 0.2, -hs * 0.44), RGB(220, 30, 50), PLA)
	end
	c.ball("OpNose", hs * 0.2, Vector3.new(0, -hs * 0.06, -hs * 0.52), RGB(235, 40, 50), PLA)
	c.H("OpSmileMid", Vector3.new(hs * 0.5, hs * 0.1, 0.06), Vector3.new(0, -hs * 0.28, -hs * 0.46), RGB(220, 30, 50), PLA)
	c.H("OpRuff", Vector3.new(hs * 0.3, hs * 1.3, hs * 1.3), Vector3.new(0, -hs * 0.6, 0), RGB(250, 250, 250), FAB, Enum.PartType.Cylinder, rz(90))
	c.ball("OpTuft", hs * 0.5, Vector3.new(0, hs * 0.55, 0), c.L.hairCol3 or RGB(255, 210, 60), FAB)
end

end
local function styleOperator(model, opId, anchored) dressCharacter(model, opId, anchored) end


-- =====================================================================
-- CONTROLS / SETTINGS DEFAULTS
-- =====================================================================
local KEY_ACTIONS = {
	{ id = "MoveF", label = "Move Forward" }, { id = "MoveB", label = "Move Back" },
	{ id = "MoveL", label = "Move Left" }, { id = "MoveR", label = "Move Right" },
	{ id = "Jump", label = "Jump" }, { id = "Sprint", label = "Sprint" }, { id = "Slide", label = "Slide / Crouch" },
	{ id = "Fire", label = "Fire" }, { id = "ADS", label = "Aim Down Sights" }, { id = "Reload", label = "Reload" },
	{ id = "Swap", label = "Swap Weapon" }, { id = "Slot1", label = "Primary Weapon" }, { id = "Slot2", label = "Secondary Weapon" },
	{ id = "Melee", label = "Melee" }, { id = "Grenade", label = "Throw Frag" }, { id = "Stim", label = "Use Stim" },
	{ id = "Airstrike", label = "Call Airstrike" },
	{ id = "Scoreboard", label = "Scoreboard" }, { id = "Menu", label = "Open Menu" },
}
local PAD_ACTIONS = {
	{ id = "Jump", label = "Jump" }, { id = "Sprint", label = "Sprint" }, { id = "Slide", label = "Slide / Crouch" },
	{ id = "Fire", label = "Fire" }, { id = "ADS", label = "Aim Down Sights" }, { id = "Reload", label = "Reload" },
	{ id = "Swap", label = "Swap Weapon" }, { id = "Melee", label = "Melee" }, { id = "Grenade", label = "Throw Frag" },
	{ id = "Stim", label = "Use Stim" }, { id = "Airstrike", label = "Call Airstrike" }, { id = "Scoreboard", label = "Scoreboard" }, { id = "Menu", label = "Open Menu" },
}
local MOBILE_BUTTONS = {
	{ id = "Fire", label = "FIRE", size = 92 }, { id = "ADS", label = "AIM", size = 70 }, { id = "Reload", label = "RELOAD", size = 62 },
	{ id = "Slide", label = "SLIDE", size = 62 }, { id = "Swap", label = "SWAP", size = 62 }, { id = "Sprint", label = "SPRINT", size = 66 },
	{ id = "Grenade", label = "FRAG", size = 54 }, { id = "Stim", label = "STIM", size = 54 },
	{ id = "Airstrike", label = "STRIKE", size = 54 },
}
local DEFAULT_KEYS = {
	MoveF = "W", MoveB = "S", MoveL = "A", MoveR = "D", Jump = "Space", Sprint = "LeftShift", Slide = "C",
	Fire = "MouseButton1", ADS = "MouseButton2", Reload = "R", Swap = "Q", Slot1 = "One", Slot2 = "Two",
	Melee = "V", Grenade = "G", Stim = "H", Airstrike = "T", Scoreboard = "Tab", Menu = "M",
}
local DEFAULT_PAD = {
	Jump = "ButtonA", Sprint = "ButtonL3", Slide = "ButtonB", Fire = "ButtonR2", ADS = "ButtonL2",
	Reload = "ButtonX", Swap = "ButtonY", Melee = "ButtonR3", Grenade = "ButtonR1", Stim = "ButtonL1", Airstrike = "DPadUp", Scoreboard = "ButtonSelect", Menu = "ButtonStart",
}
local DEFAULT_MOBILE = {
	Fire = { 0.84, 0.62 }, ADS = { 0.70, 0.78 }, Reload = { 0.93, 0.40 }, Slide = { 0.93, 0.80 },
	Swap = { 0.60, 0.58 }, Sprint = { 0.14, 0.60 }, Grenade = { 0.76, 0.40 }, Stim = { 0.60, 0.40 }, Airstrike = { 0.68, 0.26 },
}
local DEFAULT_G = {
	sens = 1.0, sensSet = false, adsSens = 0.7, fov = 85, hud = 1.0, shake = 1.0, invertY = false,
	crossColor = 1, colorblind = false, sfx = 1.0, mobScale = 1.0, mobOpacity = 0.75, autoSprint = false,
	voice = true, voiceVol = 1.0, hitSound = true, hitMark = true, feed = true, bob = 1.0, showFps = true, bright = 0.0, crossScale = 1.0,
	quality = 0, toggleAds = false, musicOn = true, musicVol = 0.5,
	ySens = 1.0, scopeSens = 0.4, padX = 1.0, padY = 1.0, padAds = 0.7, padInvert = false, padDead = 0.15, touchSens = 1.0,
	master = 1.0, tilt = 1.0, crossAlpha = 1.0, crossDot = true,
}
local CROSS_COLORS = { RGB(255, 255, 255), RGB(60, 255, 140), RGB(0, 220, 255), RGB(255, 220, 0), RGB(255, 80, 200), RGB(255, 90, 60) }

-- =====================================================================
-- SHARED: gun model builder (server = third person, client = viewmodel)
-- =====================================================================
local function buildGunModel(def, anchored)
	local m = Instance.new("Model")
	m.Name = "BO_Gun"
	local function part(name, size, off, color, mat)
		local p = Instance.new("Part")
		p.Name = name
		p.Size = size
		p.Color = color
		p.Material = mat or Enum.Material.Metal
		p.CanCollide = false
		p.CanQuery = false
		p.CanTouch = false
		p.Massless = true
		p.Anchored = anchored
		p.TopSurface = Enum.SurfaceType.Smooth
		p.BottomSurface = Enum.SurfaceType.Smooth
		p.CFrame = CFrame.new(off)
		p.Parent = m
		return p
	end
	local L = def.len
	local bodyLen = L * 0.46
	local body = part("Body", Vector3.new(0.34, 0.42, bodyLen), Vector3.new(0, 0, 0), def.body, def.bodyMat)
	m.PrimaryPart = body
	local isShotgun = def.pellets ~= nil
	local barrelLen = L * (def.scope and 0.5 or 0.42)
	part("Barrel", Vector3.new(isShotgun and 0.22 or 0.14, isShotgun and 0.22 or 0.14, barrelLen),
		Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen / 2)), def.body:Lerp(RGB(10, 10, 14), 0.4))
	part("Muzzle", Vector3.new(0.2, 0.2, 0.18), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.05)), RGB(15, 15, 20))
	if L > 2 then
		part("Stock", Vector3.new(0.28, 0.4, L * 0.26), Vector3.new(0, -0.04, bodyLen / 2 + L * 0.13), def.body:Lerp(RGB(0, 0, 0), 0.2))
	end
	part("Grip", Vector3.new(0.2, 0.5, 0.24), Vector3.new(0, -0.42, bodyLen * 0.28), RGB(24, 24, 30), Enum.Material.SmoothPlastic)
	if not isShotgun then
		part("Mag", Vector3.new(0.2, 0.62, 0.3), Vector3.new(0, -0.48, -bodyLen * 0.06), RGB(26, 26, 32))
	else
		part("Pump", Vector3.new(0.3, 0.26, 0.7), Vector3.new(0, -0.1, -bodyLen / 2 - barrelLen * 0.4), RGB(70, 55, 45))
	end
	if def.scope then
		part("Scope", Vector3.new(0.24, 0.24, L * 0.3), Vector3.new(0, 0.42, -0.1), RGB(18, 18, 22))
		part("ScopeLens", Vector3.new(0.18, 0.18, 0.06), Vector3.new(0, 0.42, -0.1 - L * 0.15), def.color, Enum.Material.Neon)
	else
		-- iron sights: two rear posts with a gap and a front post, all topping out at y = 0.37 so the sight line is level.
		-- With an optic fitted the irons are left off, so they can never sit in front of the lens.
		local hasOptic = false
		for _, oid in ipairs(def.att or {}) do
			local oa = ATTACH_BY_ID[oid]
			if oa and oa.cat == "OPTIC" then hasOptic = true end
		end
		if not hasOptic then
			local rsz = -bodyLen * 0.12
			part("Sight", Vector3.new(0.22, 0.05, 0.12), Vector3.new(0, 0.255, rsz), RGB(18, 18, 22))
			part("SightL", Vector3.new(0.05, 0.16, 0.07), Vector3.new(-0.08, 0.29, rsz), RGB(18, 18, 22))
			part("SightR", Vector3.new(0.05, 0.16, 0.07), Vector3.new(0.08, 0.29, rsz), RGB(18, 18, 22))
			part("FrontSight", Vector3.new(0.05, 0.2, 0.05), Vector3.new(0, 0.27, -(bodyLen / 2 + barrelLen * 0.88)), RGB(18, 18, 22))
		end
	end
	part("Accent", Vector3.new(0.36, 0.08, bodyLen * 0.6), Vector3.new(0, 0.12, 0), def.color, Enum.Material.Neon)
	-- extra detail: rails, handguard, ejection port, trigger guard, brake, butt pad, magazine plate
	local dkm = RGB(20, 22, 28)
	part("Rail", Vector3.new(0.16, 0.05, bodyLen * 0.85), Vector3.new(0, 0.235, 0), dkm)
	part("EjectPort", Vector3.new(0.36, 0.09, bodyLen * 0.22), Vector3.new(0, 0.07, -bodyLen * 0.08), dkm)
	part("Charging", Vector3.new(0.4, 0.07, 0.14), Vector3.new(0, 0.16, bodyLen * 0.2), RGB(56, 58, 66))
	part("TriggerGuard", Vector3.new(0.2, 0.05, 0.3), Vector3.new(0, -0.235, bodyLen * 0.1), dkm)
	part("Brake", Vector3.new(0.26, 0.26, 0.1), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.16)), RGB(30, 32, 38))
	if not isShotgun then
		part("Handguard", Vector3.new(0.3, 0.32, barrelLen * 0.5), Vector3.new(0, 0.02, -(bodyLen / 2 + barrelLen * 0.25)), def.body:Lerp(RGB(255, 255, 255), 0.1))
		part("HandguardGlow", Vector3.new(0.05, 0.02, barrelLen * 0.32), Vector3.new(0, 0.1, -(bodyLen / 2 + barrelLen * 0.25)), def.color, Enum.Material.Neon)
		part("MagPlate", Vector3.new(0.22, 0.07, 0.34), Vector3.new(0, -0.8, -bodyLen * 0.06), RGB(52, 54, 62))
	end
	if L > 2 then
		part("ButtPad", Vector3.new(0.3, 0.46, 0.07), Vector3.new(0, -0.04, bodyLen / 2 + L * 0.26 + 0.035), RGB(14, 14, 18), Enum.Material.SmoothPlastic)
		part("CheekRest", Vector3.new(0.24, 0.08, L * 0.16), Vector3.new(0, 0.2, bodyLen / 2 + L * 0.12), def.body:Lerp(RGB(255, 255, 255), 0.08))
	end
	-- camo pattern: slabs that sink INTO the receiver sides (never coplanar with another face, so no flicker)
	if def.pat and def.pat.style then
		local pt = def.pat
		local rng = Random.new(11)
		local PL, NE = Enum.Material.SmoothPlastic, Enum.Material.Neon
		local hz = bodyLen / 2 - 0.05
		local function side(w, h, z, y, col, mat, rot)
			h = math.min(h, 0.38)
			z = math.clamp(z, -hz + w / 2, hz - w / 2)
			for _, sx in ipairs({ -1, 1 }) do
				local p = part("Pat", Vector3.new(0.1, h, w), Vector3.new(sx * 0.17, y, z), col, mat or PL)
				if rot then p.CFrame = CFrame.new(sx * 0.17, y, z) * rot end
			end
		end
		local st = pt.style
		if st == "stripes" then
			for k = 1, 3 do side(0.1, 0.38, -hz + 0.1 + (2 * hz - 0.2) * (k - 1) / 2, 0, pt.c1, NE) end
			side(2 * hz - 0.1, 0.03, 0, 0.12, pt.c1, NE)
			side(2 * hz - 0.1, 0.03, 0, -0.12, pt.c1, NE)
		elseif st == "tiger" then
			for k = 1, 7 do
				side(0.06 + (k % 2) * 0.04, 0.3, -hz + 0.08 + (2 * hz - 0.16) * (k - 1) / 6, ((k % 3) - 1) * 0.03, pt.c1, PL, CFrame.Angles(math.rad(26), 0, 0))
			end
		elseif st == "blotch" then
			for k = 1, 9 do
				side(rng:NextNumber(0.16, 0.34), rng:NextNumber(0.1, 0.2), -hz + 0.1 + (2 * hz - 0.2) * (k - 1) / 8, rng:NextNumber(-0.1, 0.1), (k % 2 == 0) and pt.c1 or pt.c2)
			end
		elseif st == "digital" then
			for ix = 1, 8 do
				for iy = 1, 3 do
					local r = rng:NextNumber()
					if r > 0.42 then
						side(0.1, 0.11, -hz + 0.07 + (2 * hz - 0.14) * (ix - 1) / 7, (iy - 2) * 0.12, (r > 0.72) and pt.c1 or pt.c2)
					end
				end
			end
		elseif st == "checker" then
			for ix = 1, 7 do
				for iy = 1, 3 do
					if (ix + iy) % 2 == 0 then side(0.12, 0.12, -hz + 0.07 + (2 * hz - 0.14) * (ix - 1) / 6, (iy - 2) * 0.13, pt.c1) end
				end
			end
		elseif st == "grid" then
			for k = 1, 4 do side(0.025, 0.38, -hz + 0.05 + (2 * hz - 0.1) * (k - 1) / 3, 0, pt.c1, NE) end
			for _, gy in ipairs({ -0.12, 0, 0.12 }) do side(2 * hz - 0.05, 0.025, 0, gy, pt.c2, NE) end
		elseif st == "flames" then
			for k = 1, 6 do
				local h = 0.12 + 0.08 * ((k * 5) % 4)
				local zc = -hz + 0.08 + (2 * hz - 0.16) * (k - 1) / 5
				side(0.12, h, zc, -0.19 + h / 2, pt.c1, NE)
				side(0.06, h * 0.55, zc, -0.19 + h * 0.275, pt.c2, NE)
			end
		elseif st == "stars" then
			for _ = 1, 14 do
				local sz = rng:NextNumber(0.03, 0.065)
				side(sz, sz, rng:NextNumber(-hz, hz), rng:NextNumber(-0.17, 0.17), (rng:NextNumber() > 0.5) and pt.c1 or pt.c2, NE)
			end
		elseif st == "dots" then
			for ix = 1, 7 do
				side(0.1, 0.1, -hz + 0.08 + (2 * hz - 0.16) * (ix - 1) / 6, (ix % 2 == 0) and 0.09 or -0.09, (ix % 2 == 0) and pt.c1 or pt.c2, PL, CFrame.Angles(math.rad(45), 0, 0))
			end
		elseif st == "plates" then
			for k = 1, 3 do side((2 * hz) * 0.27, 0.3, -hz + (2 * hz) * (k - 0.5) / 3, 0, pt.c1, Enum.Material.Metal) end
		elseif st == "cracks" then
			for k = 1, 3 do
				local zc = -hz + (2 * hz) * (k - 0.5) / 3
				side(0.035, 0.22, zc - 0.02, 0.1, pt.c1, NE, CFrame.Angles(math.rad(32), 0, 0))
				side(0.035, 0.22, zc + 0.02, -0.1, pt.c1, NE, CFrame.Angles(math.rad(-32), 0, 0))
			end
		elseif st == "waves" then
			for _, gy in ipairs({ -0.12, 0, 0.12 }) do side(2 * hz - 0.05, 0.035, 0, gy, (gy == 0) and pt.c2 or pt.c1, NE) end
			for k = 1, 5 do side(0.035, 0.12, -hz + 0.1 + (2 * hz - 0.2) * (k - 1) / 4, (k % 2 == 0) and 0.06 or -0.06, pt.c1, NE) end
		elseif st == "carbon" then
			for k = 1, 9 do side(0.05, 0.38, -hz + 0.05 + (2 * hz - 0.1) * (k - 1) / 8, 0, (k % 2 == 0) and pt.c1 or pt.c2, Enum.Material.Metal) end
			for k = 1, 3 do side(2 * hz - 0.05, 0.04, 0, (k - 2) * 0.12, pt.c1, Enum.Material.Metal) end
		end
		-- repeat the theme on the handguard and stock so the whole gun matches
		local function band(zc, len, y, hh, hw2, col, mat)
			for _, sx in ipairs({ -1, 1 }) do part("Pat", Vector3.new(0.1, hh, len), Vector3.new(sx * hw2, y, zc), col, mat or PL) end
		end
		local bandMat = (st == "plates" or st == "carbon") and Enum.Material.Metal or ((st == "digital" or st == "checker" or st == "tiger" or st == "blotch" or st == "dots") and PL or NE)
		if not isShotgun then
			local zc0 = -(bodyLen / 2 + barrelLen * 0.25)
			for k = -1, 1 do band(zc0 + k * barrelLen * 0.16, barrelLen * 0.07, 0.02, 0.22, 0.15, (k == 0) and (pt.c2 or pt.c1) or pt.c1, bandMat) end
		end
		if L > 2 then
			local zs = bodyLen / 2 + L * 0.13
			for k = -1, 1, 2 do band(zs + k * L * 0.06, L * 0.04, -0.04, 0.26, 0.13, pt.c1, bandMat) end
		end
	end
	-- unique details for the premium weapons
	do
		local id2 = def.id
		local acc2, dk3 = def.color, RGB(18, 20, 26)
		if id2 == "ARC9" then
			for _, sx in ipairs({ -1, 1 }) do part("ArcFin", Vector3.new(0.06, 0.4, barrelLen * 0.6), Vector3.new(sx * 0.2, 0.0, -(bodyLen / 2 + barrelLen * 0.3)), dk3) end
			part("ArcCoil", Vector3.new(0.36, 0.36, 0.1), Vector3.new(0, 0.02, -(bodyLen / 2 + barrelLen * 0.25)), acc2, Enum.Material.Neon)
			part("ArcCoil2", Vector3.new(0.36, 0.36, 0.1), Vector3.new(0, 0.02, -(bodyLen / 2 + barrelLen * 0.5)), acc2, Enum.Material.Neon)
			part("ArcDrum", Vector3.new(0.34, 0.3, 0.3), Vector3.new(0, 0.38, bodyLen * 0.28), dk3)
		elseif id2 == "REAPER" then
			for k = 0, 4 do part("ReapFin", Vector3.new(0.36, 0.36, 0.08), Vector3.new(0, 0.05, -(bodyLen / 2 + 0.2 + k * 0.2)), dk3) end
			part("ReapHandle", Vector3.new(0.1, 0.3, bodyLen * 0.5), Vector3.new(0, 0.5, 0), dk3)
			part("ReapBox", Vector3.new(0.3, 0.5, 0.6), Vector3.new(0.34, -0.2, -bodyLen * 0.1), RGB(70, 60, 50))
			for _, sx in ipairs({ -1, 1 }) do part("ReapBipod", Vector3.new(0.05, 0.6, 0.05), Vector3.new(sx * 0.12, -0.3, -(bodyLen / 2 + barrelLen * 0.55)), dk3, Enum.Material.Metal, nil) end
		elseif id2 == "WRAITH" then
			for _, sx in ipairs({ -1, 1 }) do
				local w = part("WraithBlade", Vector3.new(0.03, 0.5, bodyLen * 0.9), Vector3.new(sx * 0.2, 0.05, 0), acc2, Enum.Material.Neon)
				w.Transparency = 0.45
			end
			for k = 1, 3 do part("WraithRing", Vector3.new(0.34, 0.34, 0.05), Vector3.new(0, 0.05, -(bodyLen / 2 + k * barrelLen * 0.22)), acc2, Enum.Material.Neon) end
			part("WraithFin", Vector3.new(0.05, 0.42, 0.9), Vector3.new(0, 0.42, bodyLen * 0.1), dk3)
		elseif id2 == "FNG" then
			part("FangMag", Vector3.new(0.2, 0.55, 0.3), Vector3.new(0, -0.62, -bodyLen * 0.25), dk3).CFrame = CFrame.new(0, -0.62, -bodyLen * 0.25) * CFrame.Angles(math.rad(-14), 0, 0)
			part("FangFore", Vector3.new(0.14, 0.4, 0.14), Vector3.new(0, -0.35, -bodyLen / 2 - 0.1), dk3)
			for k = 0, 2 do part("FangVent", Vector3.new(0.36, 0.05, 0.08), Vector3.new(0, 0.1, -bodyLen / 2 - 0.1 - k * 0.14), acc2, Enum.Material.Neon) end
			part("FangStock", Vector3.new(0.05, 0.4, 0.6), Vector3.new(0.19, 0.05, bodyLen / 2 + 0.3), dk3)
		elseif id2 == "QSR" then
			part("QsrBolt", Vector3.new(0.4, 0.07, 0.07), Vector3.new(0.3, 0.12, bodyLen * 0.12), RGB(190, 190, 200), Enum.Material.Metal)
			part("QsrKnob", Vector3.new(0.14, 0.14, 0.14), Vector3.new(0.52, 0.12, bodyLen * 0.12), acc2, Enum.Material.Neon).Shape = Enum.PartType.Ball
			for _, sx in ipairs({ -1, 1 }) do part("QsrBipod", Vector3.new(0.04, 0.7, 0.04), Vector3.new(sx * 0.14, -0.35, -(bodyLen / 2 + barrelLen * 0.5)), RGB(80, 80, 90), Enum.Material.Metal) end
			part("QsrCheek", Vector3.new(0.3, 0.12, 0.6), Vector3.new(0, 0.26, bodyLen / 2 + 0.2), RGB(90, 70, 50))
			part("QsrBand", Vector3.new(0.17, 0.17, 0.1), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen * 0.7)), acc2, Enum.Material.Neon)
		elseif id2 == "HAWK" then
			part("HawkShroud", Vector3.new(0.3, 0.3, barrelLen * 0.75), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen * 0.4)), dk3)
			for k = 1, 4 do part("HawkRing", Vector3.new(0.33, 0.33, 0.05), Vector3.new(0, 0.05, -(bodyLen / 2 + k * barrelLen * 0.16)), acc2, Enum.Material.Neon) end
			part("HawkMag", Vector3.new(0.22, 0.8, 0.4), Vector3.new(0, -0.55, -bodyLen * 0.05), dk3)
			part("HawkRest", Vector3.new(0.12, 0.5, 0.12), Vector3.new(0, -0.45, bodyLen / 2 + 0.7), dk3)
		elseif id2 == "RVL" then
			local cy = part("RvlCylinder", Vector3.new(0.5, 0.42, 0.42), Vector3.new(0, 0.02, -bodyLen * 0.1), RGB(120, 124, 136), Enum.Material.Metal)
			cy.Shape = Enum.PartType.Cylinder
			cy.CFrame = CFrame.new(0, 0.02, -bodyLen * 0.1) * CFrame.Angles(0, math.rad(90), 0)
			for k = 0, 5 do
				local a6 = math.rad(k * 60)
				part("RvlChamber", Vector3.new(0.07, 0.07, 0.52), Vector3.new(math.cos(a6) * 0.14, 0.02 + math.sin(a6) * 0.14, -bodyLen * 0.1), RGB(10, 10, 14))
			end
			part("RvlHammer", Vector3.new(0.07, 0.2, 0.12), Vector3.new(0, 0.28, bodyLen / 2 + 0.02), RGB(160, 160, 170), Enum.Material.Metal)
			part("RvlWood", Vector3.new(0.26, 0.55, 0.3), Vector3.new(0, -0.45, bodyLen * 0.34), RGB(120, 70, 36), Enum.Material.Wood)
			part("RvlBarrelRib", Vector3.new(0.1, 0.06, barrelLen * 0.9), Vector3.new(0, 0.17, -(bodyLen / 2 + barrelLen * 0.5)), acc2, Enum.Material.Neon)
		elseif id2 == "ECLIPSE" then
			for _, sx in ipairs({ -1, 1 }) do
				part("EclRail", Vector3.new(0.07, 0.07, barrelLen * 1.15), Vector3.new(sx * 0.17, 0.12, -(bodyLen / 2 + barrelLen * 0.55)), acc2, Enum.Material.Neon)
				part("EclRail2", Vector3.new(0.07, 0.07, barrelLen * 1.15), Vector3.new(sx * 0.17, -0.1, -(bodyLen / 2 + barrelLen * 0.55)), acc2, Enum.Material.Neon)
			end
			for k = 1, 5 do part("EclCoil", Vector3.new(0.46, 0.46, 0.07), Vector3.new(0, 0.05, -(bodyLen / 2 + k * barrelLen * 0.17)), dk3) end
			part("EclCore", Vector3.new(0.3, 0.3, bodyLen * 0.5), Vector3.new(0, -0.02, 0), acc2, Enum.Material.Neon).Transparency = 0.5
			part("EclFin", Vector3.new(0.04, 0.5, 1.1), Vector3.new(0, 0.5, bodyLen / 2 + 0.4), dk3)
		elseif id2 == "BANSHEE" then
			local dr = part("BanDrum", Vector3.new(0.3, 0.8, 0.8), Vector3.new(0, -0.7, -bodyLen * 0.08), dk3)
			dr.Shape = Enum.PartType.Cylinder
			dr.CFrame = CFrame.new(0, -0.7, -bodyLen * 0.08) * CFrame.Angles(0, 0, math.rad(90)) * CFrame.Angles(0, math.rad(90), 0)
			part("BanDrumGlow", Vector3.new(0.34, 0.2, 0.2), Vector3.new(0, -0.7, -bodyLen * 0.08), acc2, Enum.Material.Neon)
			for k = 0, 4 do part("BanVent", Vector3.new(0.34, 0.05, 0.1), Vector3.new(0, 0.05 + (k % 2) * 0.1, -(bodyLen / 2 + 0.15 + k * 0.17)), acc2, Enum.Material.Neon) end
			part("BanCone", Vector3.new(0.3, 0.3, 0.3), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.3)), dk3)
			part("BanSkull", Vector3.new(0.05, 0.3, 0.7), Vector3.new(0.19, 0.0, bodyLen * 0.05), acc2, Enum.Material.Neon)
		elseif id2 == "TALON" then
			for k = 0, 3 do part("TalonRib", Vector3.new(0.3, 0.06, 0.1), Vector3.new(0, 0.2, -(bodyLen / 2 + 0.1 + k * 0.16)), dk3) end
			part("TalonWindow", Vector3.new(0.38, 0.18, 0.3), Vector3.new(0, 0.1, -bodyLen * 0.15), acc2, Enum.Material.Neon)
			part("TalonHammer", Vector3.new(0.08, 0.2, 0.1), Vector3.new(0, 0.3, bodyLen / 2 + 0.05), dk3)
			part("TalonBrake", Vector3.new(0.34, 0.34, 0.2), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.3)), RGB(40, 42, 50))
		end
	end
	-- ---------- real-world gun looks (AK-47, M16, MP5 ...): gunmetal, polymer and wood instead of neon sci-fi trim ----------
	do
		local rid = def.id
		local REAL = { VX44 = true, KR9 = true, HEX12 = true, BRK8 = true, THD = true, KRAIT = true, LSX = true, FNG = true, QSR = true, HAWK = true, MK2 = true, VPR9 = true, RVL = true, RG500 = true }
		if REAL[rid] then
			local WOOD, GUNM, BLK, OLV = RGB(128, 76, 40), RGB(58, 60, 66), RGB(28, 29, 32), RGB(64, 70, 54)
			local WM, MT, PL = Enum.Material.Wood, Enum.Material.Metal, Enum.Material.SmoothPlastic
			local function get(n) return m:FindFirstChild(n) end
			local function tint(n, col, mat)
				local x = get(n)
				if x then x.Color = col if mat then x.Material = mat end end
				return x
			end
			local function tilt(n, deg)
				local x = get(n)
				if x then x.CFrame = x.CFrame * CFrame.Angles(math.rad(deg), 0, 0) end
			end
			for _, x in ipairs(m:GetChildren()) do
				if x:IsA("BasePart") and x.Material == Enum.Material.Neon and string.sub(x.Name, 1, 3) ~= "Att" and x.Name ~= "ScopeLens" then
					if x.Name == "Accent" or x.Name == "HandguardGlow" then x.Transparency = 1 else x.Material = MT x.Color = GUNM end
				end
			end
			local zf = -(bodyLen / 2 + barrelLen * 0.25) -- handguard centre
			if rid == "VX44" then -- AK-47
				tint("Body", GUNM, MT) tint("Barrel", BLK, MT) tint("Stock", WOOD, WM) tint("ButtPad", BLK, PL) tint("Grip", RGB(96, 58, 34), WM)
				tint("Handguard", WOOD, WM) tint("Mag", RGB(70, 60, 50), MT) tint("CheekRest", WOOD, WM)
				local mg = get("Mag")
				if mg then mg.Size = Vector3.new(0.22, 0.86, 0.34) mg.CFrame = CFrame.new(0, -0.58, -bodyLen * 0.1) * CFrame.Angles(math.rad(16), 0, 0) end
				part("AKGasTube", Vector3.new(0.1, 0.1, barrelLen * 0.5), Vector3.new(0, 0.16, -(bodyLen / 2 + barrelLen * 0.5)), GUNM, MT)
				part("AKGasBlock", Vector3.new(0.18, 0.2, 0.2), Vector3.new(0, 0.1, -(bodyLen / 2 + barrelLen * 0.78)), BLK, MT)
				part("AKUpperGuard", Vector3.new(0.28, 0.1, barrelLen * 0.42), Vector3.new(0, 0.17, zf - 0.02), WOOD, WM)
				part("AKDustCover", Vector3.new(0.32, 0.06, bodyLen * 0.8), Vector3.new(0, 0.23, bodyLen * 0.06), GUNM, MT)
				part("AKBrake", Vector3.new(0.18, 0.18, 0.3), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.24)), BLK, MT)
				part("AKSelector", Vector3.new(0.38, 0.04, bodyLen * 0.5), Vector3.new(0, 0.03, bodyLen * 0.1), BLK, MT)
				part("AKStockBolt", Vector3.new(0.3, 0.1, 0.1), Vector3.new(0, 0.02, bodyLen / 2 + 0.1), BLK, MT)
			elseif rid == "KR9" then -- M16A4
				tint("Body", BLK, PL) tint("Barrel", GUNM, MT) tint("Stock", BLK, PL) tint("Grip", BLK, PL) tint("Handguard", BLK, PL) tint("Mag", GUNM, MT) tint("CheekRest", BLK, PL)
				local mg = get("Mag")
				if mg then mg.Size = Vector3.new(0.2, 0.72, 0.28) end
				part("M16Lower", Vector3.new(0.34, 0.24, bodyLen * 0.52), Vector3.new(0, -0.14, bodyLen * 0.12), BLK, PL)
				part("M16FlashHider", Vector3.new(0.18, 0.18, 0.34), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.22)), BLK, MT)
				part("M16HandRing", Vector3.new(0.34, 0.34, 0.08), Vector3.new(0, 0.02, zf - barrelLen * 0.22), GUNM, MT)
				part("M16FSB", Vector3.new(0.12, 0.2, 0.16), Vector3.new(0, 0.12, -(bodyLen / 2 + barrelLen * 0.86)), GUNM, MT)
				part("M16FwdAssist", Vector3.new(0.1, 0.1, 0.14), Vector3.new(0.2, 0.08, bodyLen * 0.1), GUNM, MT)
				part("M16EjectCover", Vector3.new(0.04, 0.1, 0.3), Vector3.new(0.18, 0.06, -bodyLen * 0.05), GUNM, MT)
			elseif rid == "HEX12" then -- MP5
				tint("Body", BLK, PL) tint("Barrel", GUNM, MT) tint("Stock", BLK, PL) tint("Grip", BLK, PL) tint("Handguard", BLK, PL) tint("Mag", GUNM, MT)
				local mg = get("Mag")
				if mg then mg.Size = Vector3.new(0.18, 0.95, 0.22) mg.CFrame = CFrame.new(0, -0.6, -bodyLen * 0.12) * CFrame.Angles(math.rad(-10), 0, 0) end
				local st = get("Stock")
				if st then st.Size = Vector3.new(0.12, 0.12, st.Size.Z) st.CFrame = st.CFrame * CFrame.new(0, 0.12, 0) end
				part("MP5StockLow", Vector3.new(0.12, 0.1, L * 0.26), Vector3.new(0, -0.18, bodyLen / 2 + L * 0.13), BLK, PL)
				part("MP5StockEnd", Vector3.new(0.14, 0.4, 0.08), Vector3.new(0, -0.03, bodyLen / 2 + L * 0.26), BLK, PL)
				part("MP5Shroud", Vector3.new(0.34, 0.3, barrelLen * 0.75), Vector3.new(0, 0.04, -(bodyLen / 2 + barrelLen * 0.38)), BLK, PL)
				part("MP5Ring", Vector3.new(0.24, 0.24, 0.1), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen * 0.88)), GUNM, MT)
				part("MP5Hook", Vector3.new(0.05, 0.1, 0.16), Vector3.new(0, 0.2, bodyLen * 0.44), GUNM, MT)
			elseif rid == "BRK8" then -- Remington 870
				tint("Body", GUNM, MT) tint("Barrel", BLK, MT) tint("Stock", WOOD, WM) tint("Pump", WOOD, WM) tint("ButtPad", BLK, PL) tint("CheekRest", WOOD, WM) tint("Grip", WOOD, WM)
				local pu = get("Pump")
				if pu then pu.Size = Vector3.new(0.34, 0.3, 0.9) end
				part("R870MagTube", Vector3.new(0.14, 0.14, barrelLen * 0.95), Vector3.new(0, -0.12, -(bodyLen / 2 + barrelLen * 0.5)), GUNM, MT)
				part("R870Cap", Vector3.new(0.18, 0.18, 0.1), Vector3.new(0, -0.12, -(bodyLen / 2 + barrelLen * 0.98)), BLK, MT)
				part("R870Bead", Vector3.new(0.05, 0.05, 0.05), Vector3.new(0, 0.16, -(bodyLen / 2 + barrelLen * 0.98)), RGB(230, 200, 90), MT)
				part("R870Loader", Vector3.new(0.3, 0.1, bodyLen * 0.3), Vector3.new(0, -0.22, bodyLen * 0.08), GUNM, MT)
			elseif rid == "THD" then -- M249 SAW
				tint("Body", BLK, MT) tint("Barrel", GUNM, MT) tint("Stock", BLK, PL) tint("Handguard", BLK, PL) tint("Mag", OLV, MT) tint("Grip", BLK, PL)
				local mg = get("Mag")
				if mg then mg.Size = Vector3.new(0.5, 0.52, 0.62) mg.CFrame = CFrame.new(0, -0.5, -bodyLen * 0.04) end
				part("SAWFeed", Vector3.new(0.4, 0.14, 0.5), Vector3.new(0, 0.2, -bodyLen * 0.05), GUNM, MT)
				part("SAWHeatShield", Vector3.new(0.3, 0.2, barrelLen * 0.5), Vector3.new(0, 0.1, -(bodyLen / 2 + barrelLen * 0.32)), BLK, MT)
				part("SAWHandle", Vector3.new(0.1, 0.18, barrelLen * 0.3), Vector3.new(0, 0.24, -(bodyLen / 2 + barrelLen * 0.5)), GUNM, MT)
				for _, sx in ipairs({ -1, 1 }) do
					local lg = part("SAWBipod", Vector3.new(0.05, 0.62, 0.05), Vector3.new(sx * 0.16, -0.34, -(bodyLen / 2 + barrelLen * 0.62)), GUNM, MT)
					lg.CFrame = CFrame.new(sx * 0.2, -0.34, -(bodyLen / 2 + barrelLen * 0.62)) * CFrame.Angles(0, 0, math.rad(sx * 14))
				end
			elseif rid == "KRAIT" then -- SVD Dragunov
				tint("Body", GUNM, MT) tint("Barrel", BLK, MT) tint("Stock", WOOD, WM) tint("Handguard", WOOD, WM) tint("Grip", WOOD, WM) tint("CheekRest", WOOD, WM) tint("Mag", RGB(70, 60, 50), MT)
				local mg = get("Mag")
				if mg then mg.Size = Vector3.new(0.2, 0.6, 0.3) mg.CFrame = CFrame.new(0, -0.5, -bodyLen * 0.08) * CFrame.Angles(math.rad(10), 0, 0) end
				part("SVDHole", Vector3.new(0.12, 0.4, 0.5), Vector3.new(0, -0.3, bodyLen / 2 + L * 0.1), BLK, MT).Transparency = 0.2
				part("SVDFlashHider", Vector3.new(0.2, 0.2, 0.34), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.22)), BLK, MT)
				part("SVDGas", Vector3.new(0.1, 0.1, barrelLen * 0.4), Vector3.new(0, 0.17, -(bodyLen / 2 + barrelLen * 0.4)), GUNM, MT)
			elseif rid == "LSX" then -- bolt-action sniper (L96)
				tint("Body", OLV, PL) tint("Barrel", BLK, MT) tint("Stock", OLV, PL) tint("Grip", OLV, PL) tint("ButtPad", BLK, PL) tint("CheekRest", OLV, PL) tint("Mag", BLK, MT)
				tint("Scope", BLK, MT) tint("ScopeLens", RGB(110, 160, 190), Enum.Material.Glass)
				part("L96Bolt", Vector3.new(0.34, 0.07, 0.07), Vector3.new(0.26, 0.1, bodyLen * 0.14), RGB(190, 190, 200), MT)
				part("L96BoltKnob", Vector3.new(0.14, 0.14, 0.14), Vector3.new(0.46, 0.1, bodyLen * 0.14), BLK, PL).Shape = Enum.PartType.Ball
				part("L96Brake", Vector3.new(0.2, 0.2, 0.36), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.24)), BLK, MT)
				for _, sx in ipairs({ -1, 1 }) do
					local lg = part("L96Bipod", Vector3.new(0.04, 0.7, 0.04), Vector3.new(sx * 0.2, -0.34, -(bodyLen / 2 + barrelLen * 0.5)), GUNM, MT)
					lg.CFrame = CFrame.new(sx * 0.22, -0.34, -(bodyLen / 2 + barrelLen * 0.5)) * CFrame.Angles(0, 0, math.rad(sx * 10))
				end
			elseif rid == "RG500" then -- VMP-style machine pistol: slotted top rail, quad-rail handguard, ported brake, slab stock, lime trim
				local LIME, PANEL, DGRY = RGB(188, 226, 54), RGB(176, 180, 186), RGB(44, 46, 52)
				tint("Body", DGRY, MT) tint("Barrel", BLK, MT) tint("Stock", DGRY, PL) tint("Grip", BLK, PL) tint("Handguard", BLK, MT) tint("Mag", GUNM, MT) tint("CheekRest", DGRY, PL)
				local mg = get("Mag")
				if mg then mg.Size = Vector3.new(0.24, 0.9, 0.36) mg.CFrame = CFrame.new(0, -0.64, -bodyLen * 0.02) * CFrame.Angles(math.rad(6), 0, 0) end
				local gp = get("Grip")
				if gp then gp.CFrame = gp.CFrame * CFrame.Angles(math.rad(14), 0, 0) end
				-- receiver + slotted top rail (kept below the iron-sight line)
				part("RGUpper", Vector3.new(0.38, 0.12, bodyLen * 1.1), Vector3.new(0, 0.22, -0.02), BLK, MT)
				part("RGRail", Vector3.new(0.22, 0.05, bodyLen * 1.5), Vector3.new(0, 0.3, -bodyLen * 0.12), BLK, MT)
				for k = 0, 11 do part("RGRailSlot", Vector3.new(0.26, 0.07, 0.035), Vector3.new(0, 0.3, -bodyLen * 0.12 - bodyLen * 0.72 + k * (bodyLen * 1.44 / 11)), GUNM, MT) end
				-- quad-rail handguard with slotted rails, side vents and lime light strips
				local hz = -(bodyLen / 2 + barrelLen * 0.32)
				part("RGHand", Vector3.new(0.42, 0.36, barrelLen * 0.62), Vector3.new(0, 0.02, hz), RGB(36, 38, 44), MT)
				part("RGHandTop", Vector3.new(0.2, 0.05, barrelLen * 0.62), Vector3.new(0, 0.23, hz), BLK, MT)
				for k = 0, 5 do
					local z = hz - barrelLen * 0.27 + k * barrelLen * 0.108
					part("RGSlotT", Vector3.new(0.24, 0.06, 0.035), Vector3.new(0, 0.23, z), GUNM, MT)
					part("RGSlotB", Vector3.new(0.2, 0.05, 0.035), Vector3.new(0, -0.19, z), GUNM, MT)
					for _, sx in ipairs({ -1, 1 }) do part("RGSlotS", Vector3.new(0.05, 0.07, 0.04), Vector3.new(sx * 0.22, 0.03, z), GUNM, MT) end
				end
				for _, sx in ipairs({ -1, 1 }) do
					part("RGVent", Vector3.new(0.03, 0.14, barrelLen * 0.26), Vector3.new(sx * 0.215, 0.02, hz + barrelLen * 0.08), BLK, MT)
					part("RGHandLine", Vector3.new(0.03, 0.025, barrelLen * 0.55), Vector3.new(sx * 0.215, 0.14, hz), LIME, Enum.Material.Neon)
				end
				-- ported muzzle brake
				local mz = -(bodyLen / 2 + barrelLen + 0.12)
				part("RGBrake", Vector3.new(0.22, 0.22, 0.36), Vector3.new(0, 0.05, mz), BLK, MT)
				for k = 0, 2 do part("RGBrakeRing", Vector3.new(0.25, 0.25, 0.04), Vector3.new(0, 0.05, mz - 0.12 + k * 0.12), GUNM, MT) end
				for _, sx in ipairs({ -1, 1 }) do part("RGPort", Vector3.new(0.04, 0.08, 0.1), Vector3.new(sx * 0.115, 0.05, mz), RGB(10, 10, 12), MT) end
				part("RGPortTop", Vector3.new(0.08, 0.04, 0.1), Vector3.new(0, 0.165, mz), RGB(10, 10, 12), MT)
				-- grey side panels with lime accent lines, ejection port, charging handle
				for _, sx in ipairs({ -1, 1 }) do
					part("RGPanel", Vector3.new(0.04, 0.22, bodyLen * 0.55), Vector3.new(sx * 0.19, 0, bodyLen * 0.1), PANEL, MT)
					part("RGPanelLine", Vector3.new(0.045, 0.02, bodyLen * 0.55), Vector3.new(sx * 0.19, 0.12, bodyLen * 0.1), LIME, Enum.Material.Neon)
				end
				part("RGEject", Vector3.new(0.04, 0.08, 0.26), Vector3.new(0.2, 0.1, -bodyLen * 0.12), RGB(10, 10, 12), MT)
				part("RGChargeStem", Vector3.new(0.14, 0.05, 0.08), Vector3.new(-0.26, 0.2, bodyLen * 0.12), GUNM, MT)
				part("RGChargeKnob", Vector3.new(0.06, 0.1, 0.14), Vector3.new(-0.34, 0.2, bodyLen * 0.12), BLK, MT)
				part("RGSelector", Vector3.new(0.04, 0.06, 0.14), Vector3.new(-0.2, 0.0, bodyLen * 0.34), LIME, MT)
				-- trigger guard, grip pad
				part("RGTrigGuard", Vector3.new(0.05, 0.1, 0.38), Vector3.new(0, -0.22, bodyLen * 0.08), BLK, MT)
				part("RGTrigger", Vector3.new(0.04, 0.14, 0.04), Vector3.new(0, -0.18, bodyLen * 0.12), GUNM, MT)
				part("RGGripPad", Vector3.new(0.24, 0.06, 0.3), Vector3.new(0, -0.7, bodyLen * 0.28 + 0.12), LIME, PL)
				-- slab stock: grey plates, lime stripe, rubber pad, cheek riser
				local sz = bodyLen / 2 + L * 0.13
				for _, sx in ipairs({ -1, 1 }) do
					part("RGStockPlate", Vector3.new(0.04, 0.3, L * 0.2), Vector3.new(sx * 0.15, -0.04, sz), PANEL, MT)
					part("RGStockLine", Vector3.new(0.045, 0.02, L * 0.2), Vector3.new(sx * 0.15, 0.1, sz), LIME, Enum.Material.Neon)
				end
				part("RGStockBack", Vector3.new(0.3, 0.46, 0.1), Vector3.new(0, -0.04, bodyLen / 2 + L * 0.26 + 0.04), BLK, PL)
				part("RGStockRiser", Vector3.new(0.24, 0.1, L * 0.18), Vector3.new(0, 0.2, sz), DGRY, PL)
				-- magazine detail: ribs, lime base plate, ammo LED
				part("RGMagBase", Vector3.new(0.26, 0.06, 0.38), Vector3.new(0, -1.1, -bodyLen * 0.02 + 0.04), LIME, PL)
				for k = 0, 3 do part("RGMagRib", Vector3.new(0.26, 0.025, 0.04), Vector3.new(0, -0.4 - k * 0.2, -bodyLen * 0.02 - 0.19), GUNM, MT) end
				part("RGLed", Vector3.new(0.03, 0.05, 0.2), Vector3.new(0.19, -0.12, bodyLen * 0.3), LIME, Enum.Material.Neon)
			elseif rid == "FNG" then -- UZI
				tint("Body", BLK, MT) tint("Barrel", GUNM, MT) tint("Stock", BLK, MT) tint("Grip", BLK, PL) tint("Handguard", BLK, MT) tint("Mag", BLK, MT)
				local mg = get("Mag")
				if mg then mg.Size = Vector3.new(0.18, 0.9, 0.24) mg.CFrame = CFrame.new(0, -0.58, bodyLen * 0.28) end
				part("UziTop", Vector3.new(0.38, 0.12, bodyLen * 0.95), Vector3.new(0, 0.16, 0), BLK, MT)
				part("UziFrontCap", Vector3.new(0.34, 0.3, 0.12), Vector3.new(0, 0.03, -bodyLen / 2 - 0.02), GUNM, MT)
				part("UziGripSafety", Vector3.new(0.1, 0.1, 0.3), Vector3.new(0, -0.2, bodyLen * 0.34), GUNM, MT)
			elseif rid == "QSR" then -- Kar98k
				tint("Body", WOOD, WM) tint("Barrel", GUNM, MT) tint("Stock", WOOD, WM) tint("Grip", WOOD, WM) tint("Handguard", WOOD, WM) tint("CheekRest", WOOD, WM) tint("ButtPad", BLK, PL)
				tint("Scope", BLK, MT) tint("ScopeLens", RGB(110, 160, 190), Enum.Material.Glass)
				part("KarWoodFore", Vector3.new(0.3, 0.26, barrelLen * 0.8), Vector3.new(0, -0.03, -(bodyLen / 2 + barrelLen * 0.42)), WOOD, WM)
				part("KarBandF", Vector3.new(0.32, 0.3, 0.06), Vector3.new(0, -0.03, -(bodyLen / 2 + barrelLen * 0.82)), GUNM, MT)
				part("KarBandR", Vector3.new(0.32, 0.3, 0.06), Vector3.new(0, -0.03, -(bodyLen / 2 + barrelLen * 0.2)), GUNM, MT)
			elseif rid == "HAWK" then -- M14
				tint("Body", GUNM, MT) tint("Barrel", BLK, MT) tint("Stock", WOOD, WM) tint("Grip", WOOD, WM) tint("Handguard", WOOD, WM) tint("CheekRest", WOOD, WM) tint("ButtPad", BLK, PL)
				tint("Scope", BLK, MT) tint("ScopeLens", RGB(110, 160, 190), Enum.Material.Glass)
				tint("HawkShroud", WOOD, WM) tint("HawkMag", GUNM, MT) tint("HawkRest", BLK, MT)
				part("M14Brake", Vector3.new(0.2, 0.2, 0.3), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.22)), BLK, MT)
			elseif rid == "MK2" then -- M1911
				tint("Body", RGB(64, 66, 72), MT) tint("Barrel", GUNM, MT) tint("Grip", RGB(110, 66, 36), WM) tint("Mag", GUNM, MT)
				part("1911Slide", Vector3.new(0.32, 0.22, bodyLen * 1.0 + barrelLen * 0.5), Vector3.new(0, 0.14, -barrelLen * 0.22), RGB(78, 80, 86), MT)
				part("1911Hammer", Vector3.new(0.06, 0.12, 0.1), Vector3.new(0, 0.26, bodyLen / 2 + 0.04), GUNM, MT)
				part("1911PanelL", Vector3.new(0.04, 0.4, 0.2), Vector3.new(-0.12, -0.42, bodyLen * 0.28), RGB(96, 56, 30), WM)
				part("1911PanelR", Vector3.new(0.04, 0.4, 0.2), Vector3.new(0.12, -0.42, bodyLen * 0.28), RGB(96, 56, 30), WM)
			elseif rid == "VPR9" then -- MAC-10
				tint("Body", BLK, MT) tint("Barrel", GUNM, MT) tint("Grip", BLK, PL) tint("Mag", BLK, MT)
				part("MacTop", Vector3.new(0.36, 0.1, bodyLen), Vector3.new(0, 0.2, 0), GUNM, MT)
				part("MacSuppressor", Vector3.new(0.22, 0.22, 0.55), Vector3.new(0, 0.05, -(bodyLen / 2 + barrelLen + 0.3)), BLK, MT)
				part("MacStockWire", Vector3.new(0.05, 0.05, 0.6), Vector3.new(0, -0.1, bodyLen / 2 + 0.32), GUNM, MT)
				part("MacStrap", Vector3.new(0.06, 0.2, 0.06), Vector3.new(0, -0.2, bodyLen / 2 + 0.05), GUNM, MT)
			elseif rid == "RVL" then -- .357 Magnum
				tint("Body", GUNM, MT) tint("Barrel", RGB(86, 88, 96), MT) tint("Grip", WOOD, WM)
			end
		end
	end
	if def.att then
		local tip = -(bodyLen / 2 + barrelLen)
		local steel, dk = RGB(112, 118, 132), RGB(24, 26, 32)
		local NEON, PLA = Enum.Material.Neon, Enum.Material.SmoothPlastic
		local optDy = 0 -- the AK-47 carries its optics lower, so the reticle sits just above the dust cover instead of floating high over the gun
		local function ap(name, size, off, col, mat, rot, shape)
			if optDy ~= 0 then off = off + Vector3.new(0, optDy, 0) end
			local p = part(name, size, off, col, mat)
			if rot then p.CFrame = CFrame.new(off) * rot end
			if shape then p.Shape = shape end
			return p
		end
		local function rzz(d) return CFrame.Angles(0, 0, math.rad(d)) end
		local function rxx(d) return CFrame.Angles(math.rad(d), 0, 0) end
		for _, id in ipairs(def.att) do
			local a = ATTACH_BY_ID[id]
			if a then
				local cat = a.cat
				optDy = 0
				if cat == "OPTIC" and not def.scope and (not a.only or a.only == def.id) then
					local oz = -bodyLen * 0.1
					if id == "XSIGHT" then
						-- AK-47 cross sight: base plate, two tall posts (kept visible while aiming) and an invisible lens that sets the sight line
						ap("AttXBase", Vector3.new(0.36, 0.1, 0.2), Vector3.new(0, 0.3, oz), dk)
						ap("AttXPostL", Vector3.new(0.06, 0.32, 0.08), Vector3.new(-0.12, 0.46, oz), dk)
						ap("AttXPostR", Vector3.new(0.06, 0.32, 0.08), Vector3.new(0.12, 0.46, oz), dk)
						ap("AttLens", Vector3.new(0.1, 0.1, 0.02), Vector3.new(0, 0.46, oz), def.color, NEON).Transparency = 1
					elseif id == "RDOT" and def.id == "VX44" then
						-- AK-47 red dot: an open window (base plate, two side posts, top bar) that stays visible while aiming, with the dot floating inside it
						ap("AttFrameBase", Vector3.new(0.3, 0.06, 0.4), Vector3.new(0, 0.3, oz), dk)
						ap("AttFramePostL", Vector3.new(0.05, 0.28, 0.1), Vector3.new(-0.13, 0.44, oz), dk)
						ap("AttFramePostR", Vector3.new(0.05, 0.28, 0.1), Vector3.new(0.13, 0.44, oz), dk)
						ap("AttFrameTop", Vector3.new(0.31, 0.05, 0.1), Vector3.new(0, 0.59, oz), dk)
						ap("AttLens", Vector3.new(0.1, 0.1, 0.02), Vector3.new(0, 0.46, oz), def.color, NEON).Transparency = 1
					elseif id == "ZOOM3" then
						ap("AttOptic", Vector3.new(0.26, 0.26, 1.15), Vector3.new(0, 0.54, oz), dk)
						ap("AttOptic2", Vector3.new(0.36, 0.36, 0.24), Vector3.new(0, 0.54, oz - 0.62), dk)
						ap("AttOptic3", Vector3.new(0.12, 0.22, 0.3), Vector3.new(0, 0.36, oz + 0.25), steel)
						ap("AttOptic4", Vector3.new(0.12, 0.22, 0.3), Vector3.new(0, 0.36, oz - 0.25), steel)
						ap("AttOptic5", Vector3.new(0.14, 0.12, 0.14), Vector3.new(0, 0.74, oz), steel)
						ap("AttLens", Vector3.new(0.28, 0.28, 0.04), Vector3.new(0, 0.54, oz - 0.76), def.color, NEON)
					elseif id == "SCOPE" then
						ap("AttOptic", Vector3.new(0.28, 0.28, 1.5), Vector3.new(0, 0.58, oz), dk)
						ap("AttOptic2", Vector3.new(0.4, 0.4, 0.34), Vector3.new(0, 0.58, oz - 0.85), dk)
						ap("AttOptic3", Vector3.new(0.36, 0.36, 0.26), Vector3.new(0, 0.58, oz + 0.78), dk)
						ap("AttOptic4", Vector3.new(0.14, 0.26, 0.3), Vector3.new(0, 0.38, oz + 0.3), steel)
						ap("AttOptic5", Vector3.new(0.14, 0.26, 0.3), Vector3.new(0, 0.38, oz - 0.3), steel)
						ap("AttLens", Vector3.new(0.34, 0.34, 0.04), Vector3.new(0, 0.58, oz - 1.03), def.color, NEON)
					else
						ap("AttOptic", Vector3.new(0.24, 0.22, 0.46), Vector3.new(0, 0.45, oz), dk)
						ap("AttOptic2", Vector3.new(0.14, 0.16, 0.34), Vector3.new(0, 0.32, oz), steel)
						ap("AttLens", Vector3.new(0.17, 0.17, 0.04), Vector3.new(0, 0.47, oz - 0.24), def.color, NEON)
					end
				elseif cat == "MUZZLE" then
					if id == "COMP" then
						ap("AttMuzzle", Vector3.new(0.32, 0.24, 0.4), Vector3.new(0, 0.05, tip - 0.25), steel)
						for _, sx in ipairs({ -1, 1 }) do ap("AttMuzzleVent", Vector3.new(0.34, 0.06, 0.18), Vector3.new(0, 0.05 + sx * 0.08, tip - 0.25), def.color, NEON) end
					elseif id == "PHANTOM" then
						ap("AttMuzzle", Vector3.new(0.27, 0.27, 1.1), Vector3.new(0, 0.05, tip - 0.6), dk)
						ap("AttMuzzleBand", Vector3.new(0.3, 0.3, 0.08), Vector3.new(0, 0.05, tip - 0.6), def.color, NEON)
						ap("AttMuzzleRing", Vector3.new(0.3, 0.3, 0.06), Vector3.new(0, 0.05, tip - 1.05), steel)
					else
						ap("AttMuzzle", Vector3.new(0.25, 0.25, 0.85), Vector3.new(0, 0.05, tip - 0.5), dk)
						for k = 0, 2 do ap("AttMuzzleRing", Vector3.new(0.3, 0.3, 0.06), Vector3.new(0, 0.05, tip - 0.2 - k * 0.25), steel) end
					end
				elseif cat == "BARREL" then
					if id == "SBAR" then
						ap("AttBarrel", Vector3.new(0.3, 0.3, 0.5), Vector3.new(0, 0.05, tip + barrelLen * 0.5), steel)
					else
						ap("AttBarrel", Vector3.new(0.2, 0.2, 1.0), Vector3.new(0, 0.05, tip - 0.4), steel)
						ap("AttBarrelRing", Vector3.new(0.24, 0.24, 0.06), Vector3.new(0, 0.05, tip - 0.1), def.color, NEON)
						ap("AttBarrelRing2", Vector3.new(0.24, 0.24, 0.06), Vector3.new(0, 0.05, tip - 0.7), def.color, NEON)
					end
				elseif cat == "UNDERBARREL" then
					if id == "TGRIP" then
						ap("AttGrip", Vector3.new(0.22, 0.6, 0.22), Vector3.new(0, -0.4, -bodyLen / 2 - 0.2), dk)
						ap("AttGripRing", Vector3.new(0.28, 0.08, 0.28), Vector3.new(0, -0.5, -bodyLen / 2 - 0.2), def.color, NEON)
						ap("AttGripLight", Vector3.new(0.14, 0.14, 0.3), Vector3.new(0, -0.15, -bodyLen / 2 - 0.5), steel)
						ap("AttGripLens", Vector3.new(0.1, 0.1, 0.04), Vector3.new(0, -0.15, -bodyLen / 2 - 0.66), RGB(255, 255, 220), NEON)
					elseif id == "AGRIP" then
						ap("AttGrip", Vector3.new(0.2, 0.55, 0.2), Vector3.new(0, -0.38, -bodyLen / 2 - 0.25), steel, PLA, rxx(-28))
					else
						ap("AttGrip", Vector3.new(0.2, 0.7, 0.2), Vector3.new(0, -0.42, -bodyLen / 2 - 0.2), steel, PLA)
						ap("AttGripRing", Vector3.new(0.26, 0.07, 0.26), Vector3.new(0, -0.55, -bodyLen / 2 - 0.2), def.color, NEON)
					end
				elseif cat == "MAGAZINE" then
					if id == "DRUM" then
						ap("AttMag", Vector3.new(0.5, 1.0, 1.0), Vector3.new(0, -0.78, -bodyLen * 0.06), steel, Enum.Material.Metal, nil, Enum.PartType.Cylinder)
						ap("AttMagBand", Vector3.new(0.54, 0.5, 0.5), Vector3.new(0, -0.78, -bodyLen * 0.06), def.color, NEON, nil, Enum.PartType.Cylinder)
					elseif id == "FASTM" then
						ap("AttMag", Vector3.new(0.22, 0.9, 0.34), Vector3.new(0, -0.66, -bodyLen * 0.06), steel)
						ap("AttMagTab", Vector3.new(0.1, 0.24, 0.12), Vector3.new(0, -1.14, -bodyLen * 0.06), def.color, NEON)
						ap("AttMagBand", Vector3.new(0.24, 0.08, 0.36), Vector3.new(0, -0.5, -bodyLen * 0.06), def.color, NEON)
					else
						ap("AttMag", Vector3.new(0.22, 1.15, 0.34), Vector3.new(0, -0.8, -bodyLen * 0.06), steel)
						ap("AttMagBand", Vector3.new(0.24, 0.08, 0.36), Vector3.new(0, -0.55, -bodyLen * 0.06), def.color, NEON)
					end
				elseif cat == "STOCK" then
					if id == "CSTK" then
						ap("AttStock", Vector3.new(0.07, 0.5, 0.7), Vector3.new(0, -0.05, bodyLen / 2 + 0.4), dk)
						ap("AttStock2", Vector3.new(0.3, 0.07, 0.7), Vector3.new(0, 0.2, bodyLen / 2 + 0.4), dk)
						ap("AttStock3", Vector3.new(0.3, 0.07, 0.7), Vector3.new(0, -0.3, bodyLen / 2 + 0.4), dk)
						ap("AttStockPad", Vector3.new(0.3, 0.56, 0.08), Vector3.new(0, -0.05, bodyLen / 2 + 0.78), def.color, NEON)
					elseif id == "LSTK" then
						ap("AttStock", Vector3.new(0.07, 0.07, 0.8), Vector3.new(0, 0.1, bodyLen / 2 + 0.4), steel)
						ap("AttStock2", Vector3.new(0.07, 0.07, 0.8), Vector3.new(0, -0.2, bodyLen / 2 + 0.4), steel)
						ap("AttStock3", Vector3.new(0.12, 0.42, 0.1), Vector3.new(0, -0.05, bodyLen / 2 + 0.82), dk)
					else
						ap("AttStock", Vector3.new(0.34, 0.6, 0.7), Vector3.new(0, -0.05, bodyLen / 2 + 0.4), steel)
						ap("AttStockPad", Vector3.new(0.36, 0.64, 0.1), Vector3.new(0, -0.05, bodyLen / 2 + 0.8), def.color, NEON)
					end
				elseif cat == "LASER" then
					ap("AttLaser", Vector3.new(0.16, 0.16, 0.4), Vector3.new(0.22, -0.02, -bodyLen / 2 - 0.2), dk)
					ap("AttLaserLens", Vector3.new(0.1, 0.1, 0.04), Vector3.new(0.22, -0.02, -bodyLen / 2 - 0.42), RGB(255, 40, 40), NEON)
					ap("AttLaserBeam", Vector3.new(0.03, 0.03, 4), Vector3.new(0.22, -0.02, -bodyLen / 2 - 2.45), RGB(255, 40, 40), NEON).Transparency = 0.55
				elseif cat == "AMMO" then
					local col = (id == "AP") and RGB(80, 220, 255) or ((id == "TRACER") and RGB(255, 90, 200) or RGB(255, 150, 40))
					ap("AttAmmo", Vector3.new(0.4, 0.1, 0.16), Vector3.new(0, 0.12, bodyLen * 0.3), col, NEON)
					ap("AttAmmo2", Vector3.new(0.4, 0.1, 0.16), Vector3.new(0, 0.12, bodyLen * 0.1), col, NEON)
				end
			end
		end
	end
	if not anchored then
		for _, p in ipairs(m:GetChildren()) do
			if p:IsA("BasePart") and p ~= body then
				local w = Instance.new("WeldConstraint")
				w.Part0 = body
				w.Part1 = p
				w.Parent = body
			end
		end
	end
	return m
end

-- =====================================================================
-- SERVER
-- =====================================================================
local function runServer()
	local Players = game:GetService("Players")
	local Workspace = game:GetService("Workspace")
	local RS = game:GetService("ReplicatedStorage")
	local DataStoreService = game:GetService("DataStoreService")
	local PathfindingService = game:GetService("PathfindingService")
	local Debris = game:GetService("Debris")

	Players.RespawnTime = CFG.RESPAWN_TIME

	-- ---------- remotes + replicated state ----------
	local Net = {}
	local NetFolder = Instance.new("Folder")
	NetFolder.Name = "BO_Net"
	for _, n in ipairs({ "Queue", "Daily", "Fire", "Reload", "Swap", "Melee", "Vote", "VoteState", "Loadout", "Settings", "Profile",
		"Feed", "ShotFx", "Hit", "Ammo", "Dmg", "Died", "Spawned", "Xp", "LevelUp", "Results", "Scores", "Banner", "ClientReady", "Throw", "Stim", "Attach", "Buy", "EquipItem", "StoreMsg", "Airstrike", "Streak", "Pass" }) do
		local r = Instance.new("RemoteEvent")
		r.Name = n
		r.Parent = NetFolder
		Net[n] = r
	end
	NetFolder.Parent = RS

	local State = Instance.new("Folder")
	State.Name = "BO_State"
	State.Parent = RS
	local function setState(k, v) State:SetAttribute(k, v) end
	setState("Phase", "LOBBY")
	setState("Status", "WAITING FOR PLAYERS")
	setState("EndsAt", 0)
	setState("ModeId", "")
	setState("MapId", "")
	setState("ScoreAlpha", 0)
	setState("ScoreBravo", 0)
	setState("Limit", 0)

	local rlTimes = {}
	local function rateOk(player, key, interval)
		local t = rlTimes[player]
		if not t then
			t = {}
			rlTimes[player] = t
		end
		local now = os.clock()
		if t[key] and now - t[key] < interval then return false end
		t[key] = now
		return true
	end

	-- ---------- profiles (DataStore, retries, session lock) ----------
	local store
	do
		local ok, s = pcall(function() return DataStoreService:GetDataStore("ProjectBlackout_v1") end)
		if ok then store = s end
	end
	local Profiles = {}
	local function defaultProfile()
		return { v = 1, xp = 0, kills = 0, deaths = 0, assists = 0, wins = 0, losses = 0, matches = 0,
			headshots = 0, shots = 0, hits = 0, loadout = { primary = "VX44", secondary = "MK2" }, settings = nil,
			owned = {}, equip = { operator = "OP_STANDARD", camo = "C_NONE" } }
	end
	local NUMERIC = { "xp", "kills", "deaths", "assists", "wins", "losses", "matches", "headshots", "shots", "hits" }
	local function sanitizeProfile(d)
		local def = defaultProfile()
		if type(d) ~= "table" then return def end
		for _, k in ipairs(NUMERIC) do
			if type(d[k]) ~= "number" or d[k] ~= d[k] or d[k] < 0 then d[k] = 0 end
		end
		d.v = 1
		if type(d.loadout) ~= "table" then d.loadout = def.loadout end
		if not WEAPONS[d.loadout.primary] or WEAPONS[d.loadout.primary].slot ~= "primary" then d.loadout.primary = "VX44" end
		if not WEAPONS[d.loadout.secondary] or WEAPONS[d.loadout.secondary].slot ~= "secondary" then d.loadout.secondary = "MK2" end
		if type(d.owned) ~= "table" then d.owned = {} end
		if type(d.passClaimed) ~= "table" then d.passClaimed = {} end
		if type(d.daily) ~= "table" then d.daily = { last = 0, streak = 0 } end
		if type(d.daily.last) ~= "number" or d.daily.last ~= d.daily.last then d.daily.last = 0 end
		if type(d.daily.streak) ~= "number" or d.daily.streak < 0 or d.daily.streak > 7 then d.daily.streak = 0 end
		if type(d.equip) ~= "table" then d.equip = { operator = "OP_STANDARD", camo = "C_NONE" } end
		if d.equip.streak ~= "BOMBER" then d.equip.streak = "AIRSTRIKE" end
		if not OPERATORS[d.equip.operator] or not operatorOwned(d.equip.operator, d.owned, { level = (levelFromXp(d.xp or 0)), kills = d.kills, headshots = d.headshots, wins = d.wins }) then d.equip.operator = "OP_STANDARD" end
		if not CAMOS[d.equip.camo] or not (CAMOS[d.equip.camo].free or d.owned[d.equip.camo]) then d.equip.camo = "C_NONE" end
		for _, wid in ipairs({ "primary", "secondary" }) do
			local w = WEAPONS[d.loadout[wid]]
			if w and w.premium and not d.owned["W_" .. w.id] then d.loadout[wid] = (wid == "primary") and "VX44" or "MK2" end
		end
		if type(d.loadout.att) ~= "table" then d.loadout.att = {} end
		for wid, l in pairs(d.loadout.att) do
			if not WEAPONS[wid] then d.loadout.att[wid] = nil else d.loadout.att[wid] = cleanAttach(l, LEVEL_CAP, d.owned, wid) end
		end
		return d
	end

	local function levelOf(player)
		local p = Profiles[player]
		return p and (levelFromXp(p.xp)) or 1
	end

	local function sendProfile(player)
		local p = Profiles[player]
		if not p or not player.Parent then return end
		local lvl, into, need = levelFromXp(p.xp)
		player:SetAttribute("Level", lvl)
		Net.Profile:FireClient(player, {
			xp = p.xp, level = lvl, into = into, need = need, kills = p.kills, deaths = p.deaths, assists = p.assists,
			wins = p.wins, losses = p.losses, matches = p.matches, headshots = p.headshots, shots = p.shots, hits = p.hits,
			loadout = p.loadout, settings = p.settings, saved = not p.noSave, owned = p.owned, equip = p.equip, daily = p.daily, passClaimed = p.passClaimed,
		})
	end

	local function loadProfile(player)
		local temp = defaultProfile()
		temp.noSave = true
		Profiles[player] = temp
		sendProfile(player)
		if not store then
			warn("BLACKOUT: DataStore unavailable. Progress will not be saved.")
			return
		end
		local key = "P_" .. player.UserId
		local data
		for attempt = 1, 4 do
			local ok, res = pcall(function()
				return store:UpdateAsync(key, function(old)
					old = old or defaultProfile()
					local l = old.lock
					if type(l) == "table" and l.job ~= game.JobId and os.time() - (l.t or 0) < 90 then return nil end
					old.lock = { job = game.JobId, t = os.time() }
					return old
				end)
			end)
			if ok and res then
				data = res
				break
			end
			if not ok and tostring(res):find("Studio") then
				warn("BLACKOUT: Studio API access is off, so XP will not save. Turn on Game Settings > Security > "
					.. "'Enable Studio Access to API Services' (game must be published).")
				return
			end
			if not ok then warn("BLACKOUT: profile load failed: " .. tostring(res)) end
			task.wait(attempt * 1.5)
		end
		if not player.Parent then return end
		if not data then
			warn("BLACKOUT: could not load profile for " .. player.Name .. ". Using a temporary profile (will not save).")
			return
		end
		local clean = sanitizeProfile(data)
		clean.settings = type(data.settings) == "table" and data.settings or nil
		Profiles[player] = clean
		sendProfile(player)
	end

	local function saveProfile(player, release)
		local p = Profiles[player]
		if not p or p.noSave or not store then return end
		local key = "P_" .. player.UserId
		local ok, err = pcall(function()
			store:UpdateAsync(key, function(old)
				if old and type(old.lock) == "table" and old.lock.job ~= game.JobId and os.time() - (old.lock.t or 0) < 90 then
					return nil
				end
				local out = {}
				for k, v in pairs(p) do out[k] = v end
				out.noSave = nil
				out.lock = (not release) and { job = game.JobId, t = os.time() } or nil
				return out
			end)
		end)
		if not ok then warn("BLACKOUT: save failed for " .. player.Name .. ": " .. tostring(err)) end
	end

	-- ---------- settings validation ----------
	local function validKeyName(v)
		if type(v) ~= "string" or #v > 24 then return false end
		if v == "MouseButton1" or v == "MouseButton2" or v == "MouseButton3" then return true end
		local ok, res = pcall(function() return Enum.KeyCode[v] end)
		return ok and res ~= nil
	end
	local function sanitizeSettings(t)
		if type(t) ~= "table" then return nil end
		local out = { keys = {}, pad = {}, mobile = {}, g = {} }
		if type(t.keys) == "table" then
			for _, a in ipairs(KEY_ACTIONS) do
				local v = t.keys[a.id]
				if validKeyName(v) then out.keys[a.id] = v end
			end
		end
		if type(t.pad) == "table" then
			for _, a in ipairs(PAD_ACTIONS) do
				local v = t.pad[a.id]
				if type(v) == "string" and v:sub(1, 6) == "Button" and validKeyName(v) then out.pad[a.id] = v end
			end
		end
		if type(t.mobile) == "table" then
			for _, b in ipairs(MOBILE_BUTTONS) do
				local v = t.mobile[b.id]
				if type(v) == "table" and type(v[1]) == "number" and type(v[2]) == "number" and v[1] == v[1] and v[2] == v[2] then
					out.mobile[b.id] = { math.clamp(v[1], 0.03, 0.97), math.clamp(v[2], 0.05, 0.95) }
				end
			end
		end
		if type(t.g) == "table" then
			local g = t.g
			local function num(k, lo, hi)
				if type(g[k]) == "number" and g[k] == g[k] then out.g[k] = math.clamp(g[k], lo, hi) end
			end
			num("sens", 0.1, 3) num("adsSens", 0.1, 2) num("fov", 70, 110) num("hud", 0.7, 1.4) num("shake", 0, 1)
			num("sfx", 0, 1) num("mobScale", 0.6, 1.6) num("mobOpacity", 0.2, 1)
			num("voiceVol", 0, 1) num("bob", 0, 1.5) num("bright", -0.1, 0.2) num("crossScale", 0.6, 2) num("quality", 0, 3) num("musicVol", 0, 1)
			num("ySens", 0.1, 3) num("scopeSens", 0.1, 2) num("padX", 0.2, 3) num("padY", 0.2, 3) num("padAds", 0.2, 2) num("padDead", 0, 0.5)
			num("touchSens", 0.2, 3) num("master", 0, 1) num("tilt", 0, 1) num("crossAlpha", 0.2, 1)
			for _, bk in ipairs({ "voice", "hitSound", "hitMark", "feed", "showFps", "toggleAds", "musicOn" }) do
				if type(g[bk]) == "boolean" then out.g[bk] = g[bk] end
			end
			if type(g.crossColor) == "number" then out.g.crossColor = math.clamp(math.floor(g.crossColor), 1, #CROSS_COLORS) end
			if type(g.invertY) == "boolean" then out.g.invertY = g.invertY end
			if type(g.padInvert) == "boolean" then out.g.padInvert = g.padInvert end
			if type(g.crossDot) == "boolean" then out.g.crossDot = g.crossDot end
			if type(g.sensSet) == "boolean" then out.g.sensSet = g.sensSet end
			if type(g.colorblind) == "boolean" then out.g.colorblind = g.colorblind end
			if type(g.autoSprint) == "boolean" then out.g.autoSprint = g.autoSprint end
		end
		return out
	end

	-- ---------- world helpers ----------
	local WorldFolder = Instance.new("Folder")
	WorldFolder.Name = "BO_World"
	WorldFolder.Parent = Workspace
	local HubFolder = Instance.new("Folder")
	HubFolder.Name = "BO_Hub"
	HubFolder.Parent = WorldFolder
	local MapFolder = Instance.new("Folder")
	MapFolder.Name = "BO_Map"
	MapFolder.Parent = WorldFolder
	local BotFolder = Instance.new("Folder")
	BotFolder.Name = "BO_Bots"
	BotFolder.Parent = Workspace

	-- the default Roblox Baseplate has its top at y = 0, the same height as our floors: sink it out of the way
	for _, inst in ipairs(Workspace:GetChildren()) do
		if inst:IsA("BasePart") and inst.Name == "Baseplate" then
			inst.Position = inst.Position - Vector3.new(0, 8, 0)
		end
	end
	local buildN = 0
	local function blk(parent, cx, by, cz, sx, sy, sz, color, mat, name)
		buildN += 1
		if buildN % 120 == 0 then task.wait() end -- spread map building over several frames (no server hitch)
		local p = Instance.new("Part")
		p.Anchored = true
		p.CanTouch = false
		p.CastShadow = (sx * sy * sz) > 300 and name ~= "Floor" and name ~= "HubFloor" and name ~= "Boundary"
		p.Size = Vector3.new(sx, sy, sz)
		p.Position = Vector3.new(cx, by + sy / 2, cz)
		p.Color = color
		p.Material = mat or Enum.Material.SmoothPlastic
		if name == "Floor" or name == "HubFloor" then
			-- self-lit floor: it can never go black or flash, whatever the lighting does
			p.Material = Enum.Material.Neon
			p.Color = color:Lerp(RGB(255, 255, 255), 0.18)
			-- sit 0.06 above y = 0: a template Baseplate or Terrain has its top at exactly y = 0, and two surfaces at the same
			-- height fight for the screen (z-fighting), which flickers whenever the camera moves or the FOV changes (aiming)
			p.Position = p.Position + Vector3.new(0, 0.06, 0)
		end
		p.Name = name or "Block"
		p.TopSurface = Enum.SurfaceType.Smooth
		p.BottomSurface = Enum.SurfaceType.Smooth
		p.Parent = parent
		return p
	end
	local function blkR(parent, cx, by, cz, sx, sy, sz, rotY, color, mat, name)
		local p = blk(parent, cx, by, cz, sx, sy, sz, color, mat, name)
		p.CFrame = CFrame.new(cx, by + sy / 2, cz) * CFrame.Angles(0, math.rad(rotY), 0)
		return p
	end
	local function neon(parent, cx, by, cz, sx, sy, sz, color)
		local p = blk(parent, cx, by, cz, sx, sy, sz, color, Enum.Material.Neon, "Neon")
		p.CanCollide = false
		return p
	end
	local function ramp(parent, a, b, width, color, mat)
		local len = (b - a).Magnitude
		local p = Instance.new("Part")
		p.Anchored = true
		p.Size = Vector3.new(width, 1, len)
		p.CFrame = CFrame.lookAt(a, b) * CFrame.new(0, -0.5, -len / 2)
		p.Color = color
		p.Material = mat or Enum.Material.Concrete
		p.Name = "Ramp"
		p.TopSurface = Enum.SurfaceType.Smooth
		p.BottomSurface = Enum.SurfaceType.Smooth
		p.Parent = parent
		return p
	end
	local function lamp(parent, pos, color, range, bright)
		local p = Instance.new("Part")
		p.Anchored = true
		p.Transparency = 1
		p.CanCollide = false
		p.CanQuery = false
		p.Size = Vector3.new(1, 1, 1)
		p.Position = pos
		p.Name = "Lamp"
		p.Parent = parent
		local l = Instance.new("PointLight")
		l.Color = color
		l.Range = range or 40
		l.Brightness = bright or 2
		l.Parent = p
	end
	local function signboard(parent, pos, size, face, text, color)
		local p = Instance.new("Part")
		p.Anchored = true
		p.Size = size
		p.Position = pos
		p.Color = RGB(12, 12, 18)
		p.Material = Enum.Material.SmoothPlastic
		p.Name = "Sign"
		p.Parent = parent
		local g = Instance.new("SurfaceGui")
		g.Face = face
		g.SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud
		g.PixelsPerStud = 40
		g.LightInfluence = 0
		g.Parent = p
		local t = Instance.new("TextLabel")
		t.Size = UDim2.new(1, 0, 1, 0)
		t.BackgroundTransparency = 1
		t.Text = text
		t.TextScaled = true
		t.Font = Enum.Font.GothamBlack
		t.TextColor3 = color
		t.Parent = g
	end
	local function boundary(f, half, h)
		local function wall(cx, cz, sx, sz)
			local p = blk(f, cx, -5, cz, sx, h + 5, sz, RGB(0, 0, 0), Enum.Material.SmoothPlastic, "Bound")
			p.Transparency = 1
		end
		wall(0, -(half + 2), half * 2 + 8, 4)
		wall(0, half + 2, half * 2 + 8, 4)
		wall(-(half + 2), 0, 4, half * 2 + 8)
		wall(half + 2, 0, 4, half * 2 + 8)
		local c = blk(f, 0, h, 0, half * 2 + 8, 2, half * 2 + 8, RGB(0, 0, 0), Enum.Material.SmoothPlastic, "Ceiling")
		c.Transparency = 1
		for _, d in ipairs(f:GetChildren()) do
			if d.Name == "Bound" or d.Name == "Ceiling" then d.CastShadow = false; d.CanQuery = false end
		end
	end

	-- ---------- soldier body (blocky tactical operator; limbs are Motor6D joints so clients can animate them) ----------
	local JOINTED = { Torso = true, LeftArm = true, RightArm = true, LeftLeg = true, RightLeg = true }
	local function buildBotBody(name, color, static, uniform)
		uniform = uniform or RGB(36, 38, 40)
		local m = Instance.new("Model")
		m.Name = name
		local root = Instance.new("Part")
		root.Name = "HumanoidRootPart"
		root.Size = Vector3.new(2, 2, 1)
		root.Transparency = 1
		root.Anchored = static
		root.CanCollide = not static
		root.CanTouch = false
		root.Parent = m
		m.PrimaryPart = root
		local rc = root.CFrame
		local vestC, plateC, gearC, darkC = RGB(44, 48, 54), RGB(78, 84, 88), RGB(86, 82, 62), RGB(30, 30, 34)
		local glow = color:Lerp(RGB(0, 0, 0), 0.4)
		local function mk(n, size, cf, col, mat, att)
			local p = Instance.new("Part")
			p.Name = n
			p.Size = size
			p.Color = col
			p.Material = mat or Enum.Material.Fabric
			p.CanCollide = false
			p.CanTouch = false
			p.Massless = true
			p.Anchored = static
			p.CastShadow = false
			p.TopSurface = Enum.SurfaceType.Smooth
			p.BottomSurface = Enum.SurfaceType.Smooth
			p.CFrame = cf
			if att then p:SetAttribute("Att", att) end
			p.Parent = m
			return p
		end
		local torso = mk("Torso", Vector3.new(2, 2, 1), rc, uniform)
		torso.CastShadow = true
		local function onTorso(n, size, off, col, mat) return mk(n, size, rc * CFrame.new(off), col, mat, "Torso") end
		onTorso("Head", Vector3.new(1.1, 1.1, 1.1), Vector3.new(0, 1.65, 0), RGB(176, 134, 100), Enum.Material.SmoothPlastic)
		-- limbs (arms are posed holding a rifle; the pose is baked into the joint)
		local function limb(n, size, joint, rot, col, att)
			local cf = rc * CFrame.new(joint) * rot * CFrame.new(0, -size.Y / 2, 0)
			local p = mk(n, size, cf, col, nil, nil)
			return p, cf
		end
		local rotR = CFrame.Angles(0, math.rad(14), 0) * CFrame.Angles(math.rad(78), 0, 0)
		local rotL = CFrame.Angles(0, math.rad(-22), 0) * CFrame.Angles(math.rad(86), 0, 0)
		local armR, cfR = limb("RightArm", Vector3.new(0.8, 2, 0.8), Vector3.new(1.4, 0.85, 0), rotR, uniform)
		local armL, cfL = limb("LeftArm", Vector3.new(0.8, 2, 0.8), Vector3.new(-1.4, 0.85, 0), rotL, uniform)
		local legR, cfLR = limb("RightLeg", Vector3.new(0.9, 2, 0.9), Vector3.new(0.5, -1, 0), CFrame.new(), uniform)
		local legL, cfLL = limb("LeftLeg", Vector3.new(0.9, 2, 0.9), Vector3.new(-0.5, -1, 0), CFrame.new(), uniform)
		local function det(n, size, limbCF, off, col, mat, att)
			return mk(n, size, limbCF * CFrame.new(off), col, mat, att)
		end
		if not static then
			local hum = Instance.new("Humanoid")
			hum.RigType = Enum.HumanoidRigType.R15
			hum.HipHeight = 2
			hum.RequiresNeck = false
			hum.BreakJointsOnDeath = false
			hum.MaxHealth = 100
			hum.Health = 100
			hum.WalkSpeed = 14
			hum.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None
			hum.HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff
			hum.Parent = m
		end
		return m
	end
	-- joints: Torso -> root, arms -> Torso, legs -> root, everything else welded to its attach part
	local function weldBot(m)
		local root = m.PrimaryPart
		local torso = m:FindFirstChild("Torso")
		local function motor(p0, p1)
			local mo = Instance.new("Motor6D")
			mo.Name = "Joint"
			local top = p1.Name ~= "Torso"
			local jointWorld = top and (p1.CFrame * CFrame.new(0, p1.Size.Y / 2, 0)) or p1.CFrame
			mo.Part0 = p0
			mo.Part1 = p1
			mo.C0 = p0.CFrame:ToObjectSpace(jointWorld)
			mo.C1 = top and CFrame.new(0, p1.Size.Y / 2, 0) or CFrame.new()
			mo.Parent = p1
		end
		for _, p in ipairs(m:GetChildren()) do
			if p:IsA("BasePart") and p ~= root then
				if p.Name == "Torso" or p.Name == "LeftLeg" or p.Name == "RightLeg" then
					motor(root, p)
				elseif p.Name == "LeftArm" or p.Name == "RightArm" then
					motor(torso, p)
				else
					local attach = m:FindFirstChild(p:GetAttribute("Att") or "Torso") or torso
					local w = Instance.new("WeldConstraint")
					w.Part0 = attach
					w.Part1 = p
					w.Parent = p
				end
			end
		end
	end

	-- ---------- HUB ----------
	local hubSpinners = {}
	local hubSpawnCF
	local function buildHub()
		HubFolder:ClearAllChildren()
		table.clear(hubSpinners)
		local f = HubFolder
		local hx, hy, hz = CFG.HUB_POS.X, CFG.HUB_POS.Y, CFG.HUB_POS.Z
		local AMBER, ICE = RGB(255, 150, 60), RGB(110, 190, 230)
		local CONC, DCONC, STEEL, DSTEEL = RGB(96, 96, 98), RGB(66, 66, 70), RGB(62, 66, 74), RGB(34, 37, 43)
		local HAZ, BLK = RGB(225, 165, 30), RGB(24, 24, 26)
		local function spin(part, speed) part:SetAttribute("Spin", speed) end
		local function barrel(x, z, col)
			local b = Instance.new("Part")
			b.Shape = Enum.PartType.Cylinder
			b.Anchored = true
			b.CanTouch = false
			b.Size = Vector3.new(4, 3, 3)
			b.CFrame = CFrame.new(x, hy + 2, z) * CFrame.Angles(0, 0, math.pi / 2)
			b.Color = col
			b.Material = Enum.Material.Metal
			b.Name = "Barrel"
			b.Parent = f
		end
		-- floor + painted pad
		blk(f, hx, hy - 2, hz, 130, 2, 130, CONC, Enum.Material.Plastic, "HubFloor")
		blk(f, hx, hy, hz, 44, 0.15, 44, DCONC, Enum.Material.Concrete, "Pad")
		for i = 0, 9 do
			local c = (i % 2 == 0) and HAZ or BLK
			local o = -22 + i * 4.4 + 2.2
			blk(f, hx + o, hy + 0.15, hz - 22, 4.4, 0.05, 1, c, Enum.Material.SmoothPlastic, "Hazard")
			blk(f, hx + o, hy + 0.15, hz + 22, 4.4, 0.05, 1, c, Enum.Material.SmoothPlastic, "Hazard")
			blk(f, hx - 22, hy + 0.15, hz + o, 1, 0.05, 4.4, c, Enum.Material.SmoothPlastic, "Hazard")
			blk(f, hx + 22, hy + 0.15, hz + o, 1, 0.05, 4.4, c, Enum.Material.SmoothPlastic, "Hazard")
		end
		-- walls with ribs, ceiling with trusses
		local function wall(cx, cz, sx, sz) blk(f, cx, hy, cz, sx, 34, sz, DSTEEL, Enum.Material.Metal, "HubWall") end
		wall(hx, hz - 62, 130, 3)
		wall(hx - 62, hz, 3, 130)
		wall(hx + 62, hz, 3, 130)
		wall(hx, hz + 62, 130, 3)
		for k = -52, 52, 13 do
			blk(f, hx + k, hy, hz - 59.8, 1.2, 34, 1.4, STEEL, Enum.Material.Metal, "Rib")
			blk(f, hx + k, hy, hz + 59.8, 1.2, 34, 1.4, STEEL, Enum.Material.Metal, "Rib")
			blk(f, hx - 59.8, hy, hz + k, 1.4, 34, 1.2, STEEL, Enum.Material.Metal, "Rib")
			blk(f, hx + 59.8, hy, hz + k, 1.4, 34, 1.2, STEEL, Enum.Material.Metal, "Rib")
		end
		blk(f, hx, hy + 34, hz, 130, 2, 130, RGB(20, 22, 26), Enum.Material.Metal, "HubCeiling")
		for k = -48, 48, 16 do
			blk(f, hx, hy + 32, hz + k, 128, 1.5, 2, STEEL, Enum.Material.Metal, "Truss")
			blk(f, hx + k, hy + 30.5, hz, 2, 1.5, 128, STEEL, Enum.Material.Metal, "Truss")
		end
		-- warm work lights (dim, industrial)
		for _, lx in ipairs({ -32, 0, 32 }) do
			for _, lz in ipairs({ -32, 0, 32 }) do
				local fx = blk(f, hx + lx, hy + 29, hz + lz, 5, 0.5, 5, RGB(235, 225, 200), Enum.Material.Neon, "Light")
				fx.CanCollide = false
				lamp(f, Vector3.new(hx + lx, hy + 26, hz + lz), RGB(255, 228, 190), 55, 1.4)
			end
		end
		-- hangar door + title
		blk(f, hx, hy, hz - 59.4, 38, 26, 0.8, RGB(46, 50, 56), Enum.Material.Metal, "HangarDoor")
		for k = -16, 16, 8 do blk(f, hx + k, hy, hz - 58.9, 0.6, 26, 0.4, DSTEEL, Enum.Material.Metal, "DoorSeam") end
		blk(f, hx, hy + 26, hz - 59.4, 42, 1.2, 1, HAZ, Enum.Material.SmoothPlastic, "DoorFrame")
		signboard(f, Vector3.new(hx, hy + 30, hz - 59.2), Vector3.new(50, 6, 0.4), Enum.NormalId.Back, "BLACKOUT", AMBER)
		signboard(f, Vector3.new(hx - 59.6, hy + 14, hz), Vector3.new(0.4, 3, 20), Enum.NormalId.Right, "ARMORY", AMBER)
		signboard(f, Vector3.new(hx + 59.6, hy + 14, hz), Vector3.new(0.4, 3, 28), Enum.NormalId.Left, "OPERATIONS", ICE)
		-- armoury wall: every weapon on a rack
		blk(f, hx - 60, hy, hz, 0.6, 13, 70, DSTEEL, Enum.Material.Metal, "RackBack")
		local racks = { PRIMARY_ORDER[1], PRIMARY_ORDER[2], PRIMARY_ORDER[3], PRIMARY_ORDER[4], PRIMARY_ORDER[5], PRIMARY_ORDER[6], PRIMARY_ORDER[7],
			SECONDARY_ORDER[1], SECONDARY_ORDER[2] }
		for row = 1, 3 do
			local y = 2.5 + (row - 1) * 3.8
			blk(f, hx - 59, hy + y, hz, 1.4, 0.3, 66, STEEL, Enum.Material.Metal, "Shelf")
			for col = 1, 3 do
				local id = racks[(row - 1) * 3 + col]
				local g = buildGunModel(WEAPONS[id], true)
				g.Parent = f
				g:PivotTo(CFrame.new(hx - 58.4, hy + y + 0.65, hz + (col - 2) * 20))
			end
		end
		-- crates, barrels, sandbags
		local function crates(x, z)
			blk(f, x, 0 + hy, z, 5, 5, 5, RGB(78, 74, 60), Enum.Material.Metal, "Crate")
			blk(f, x + 5.6, hy, z + 1, 5, 3.5, 5, RGB(66, 70, 58), Enum.Material.Metal, "Crate")
			blk(f, x + 1, hy + 5, z, 4, 3, 4, RGB(88, 80, 62), Enum.Material.Metal, "Crate")
		end
		crates(hx - 46, hz - 46)
		crates(hx + 40, hz - 4)
		crates(hx - 46, hz + 40)
		barrel(hx + 46, hz + 42, RGB(80, 40, 36))
		barrel(hx + 50, hz + 45, RGB(60, 62, 56))
		barrel(hx + 44, hz + 47, RGB(60, 62, 56))
		for _, sb in ipairs({ { -30, 42 }, { 30, 44 } }) do
			for i = 0, 3 do
				blk(f, hx + sb[1] + i * 3.6 - 5.4, hy, hz + sb[2], 3.6, 1.1, 1.6, RGB(112, 100, 74), Enum.Material.Fabric, "Sandbag")
				if i < 3 then blk(f, hx + sb[1] + i * 3.6 - 3.6, hy + 1.1, hz + sb[2], 3.6, 1.1, 1.6, RGB(104, 94, 70), Enum.Material.Fabric, "Sandbag") end
			end
		end
		-- parked dropship (rotor spins on the client)
		do
			local x, z = hx + 36, hz - 30
			blk(f, x, hy + 1.2, z, 8, 5, 18, RGB(44, 48, 50), Enum.Material.Metal, "Dropship")
			blk(f, x, hy + 2.2, z - 9.2, 6, 3, 2, RGB(30, 44, 58), Enum.Material.SmoothPlastic, "Dropship")
			blk(f, x, hy + 3.2, z + 13, 2, 2, 14, RGB(44, 48, 50), Enum.Material.Metal, "Dropship")
			blk(f, x, hy + 3.2, z + 19.5, 0.5, 5, 3, RGB(44, 48, 50), Enum.Material.Metal, "Dropship")
			blk(f, x - 3.2, hy, z, 0.5, 0.6, 15, DSTEEL, Enum.Material.Metal, "Skid")
			blk(f, x + 3.2, hy, z, 0.5, 0.6, 15, DSTEEL, Enum.Material.Metal, "Skid")
			blk(f, x, hy + 6.2, z, 1, 2, 1, DSTEEL, Enum.Material.Metal, "Mast")
			local b1 = blk(f, x, hy + 8, z, 30, 0.25, 1.6, RGB(26, 28, 30), Enum.Material.Metal, "Rotor")
			local b2 = blk(f, x, hy + 8, z, 1.6, 0.25, 30, RGB(26, 28, 30), Enum.Material.Metal, "Rotor")
			b1.CanCollide = false
			b2.CanCollide = false
			spin(b1, 7)
			spin(b2, 7)
		end
		-- holo table
		local base = Instance.new("Part")
		base.Shape = Enum.PartType.Cylinder
		base.Anchored = true
		base.CanTouch = false
		base.Size = Vector3.new(3, 20, 20)
		base.CFrame = CFrame.new(hx, hy + 1.5, hz) * CFrame.Angles(0, 0, math.pi / 2)
		base.Color = RGB(30, 33, 40)
		base.Material = Enum.Material.Metal
		base.Name = "HoloBase"
		base.Parent = f
		local ring = Instance.new("Part")
		ring.Shape = Enum.PartType.Cylinder
		ring.Anchored = true
		ring.CanCollide = false
		ring.CanTouch = false
		ring.Size = Vector3.new(0.3, 17, 17)
		ring.CFrame = CFrame.new(hx, hy + 3.2, hz) * CFrame.Angles(0, 0, math.pi / 2)
		ring.Color = ICE
		ring.Material = Enum.Material.Neon
		ring.Transparency = 0.55
		ring.Name = "HoloRing"
		ring.Parent = f
		local globe = Instance.new("Part")
		globe.Shape = Enum.PartType.Ball
		globe.Anchored = true
		globe.CanCollide = false
		globe.CanTouch = false
		globe.Size = Vector3.new(8, 8, 8)
		globe.Position = Vector3.new(hx, hy + 9, hz)
		globe.Color = ICE
		globe.Material = Enum.Material.ForceField
		globe.Name = "HoloGlobe"
		globe.Parent = f
		local core = Instance.new("Part")
		core.Anchored = true
		core.CanCollide = false
		core.CanTouch = false
		core.Size = Vector3.new(2.4, 2.4, 2.4)
		core.Position = Vector3.new(hx, hy + 9, hz)
		core.Color = AMBER
		core.Material = Enum.Material.Neon
		core.Name = "HoloCore"
		core.Parent = f
		spin(globe, 0.5)
		spin(core, -1.3)
		lamp(f, Vector3.new(hx, hy + 9, hz), ICE, 48, 1.6)
		-- operators around the table
		local unis = { RGB(52, 50, 38), RGB(34, 40, 50), RGB(30, 30, 34), RGB(52, 50, 38) }
		for i = 1, 4 do
			local a = math.rad(i * 90 + 45)
			local sx, sz = hx + math.cos(a) * 17, hz + math.sin(a) * 17
			local s2 = buildBotBody("Soldier" .. i, i % 2 == 0 and AMBER or ICE, true, unis[i])
			s2.Parent = f
			s2:PivotTo(CFrame.lookAt(Vector3.new(sx, hy + 3, sz), Vector3.new(hx, hy + 3, hz)))
			pcall(dressCharacter, s2, OPERATOR_ORDER[({ 1, 9, 13, 20 })[i] or 1] or "OP_STANDARD", true, i % 2 == 0 and AMBER or ICE)
			local g = buildGunModel(WEAPONS[PRIMARY_ORDER[i]], true)
			g.Parent = s2
			g:PivotTo(s2.PrimaryPart.CFrame * CFrame.new(0.25, 0.55, -2.1))
		end
		-- spawn
		local sp = Instance.new("SpawnLocation")
		sp.Name = "HubSpawn"
		sp.Anchored = true
		sp.Size = Vector3.new(14, 1, 14)
		sp.Position = Vector3.new(hx, hy + 0.5, hz + 32)
		sp.Transparency = 1
		sp.Neutral = true
		sp.Duration = 0
		sp.Parent = f
		hubSpawnCF = CFrame.lookAt(Vector3.new(hx, hy + 4, hz + 32), Vector3.new(hx, hy + 4, hz))
		local bound = blk(f, hx, hy - 12, hz, 140, 2, 140, RGB(0, 0, 0), Enum.Material.SmoothPlastic, "HubVoidCatch")
		bound.Transparency = 1
		-- the hangar floats high above the arenas: it must never throw sun shadows onto them (that made arena floors flash dark/bright)
		for _, d in ipairs(f:GetDescendants()) do
			if d:IsA("BasePart") then d.CastShadow = false end
		end
	end
	buildHub()

	-- ---------- MAPS (original layouts, dark gritty near-future war zones) ----------
	local MapBuilders = {}
	local MapRand = Random.new(1)
	local Spawns = {}
	local function R(a, b) return MapRand:NextNumber(a, b) end
	local function nearSpawn(x, z, r)
		for _, v in ipairs(Spawns) do
			if (v.X - x) ^ 2 + (v.Z - z) ^ 2 < r * r then return true end
		end
		return false
	end
	local function setSpawns(sp)
		Spawns = {}
		for _, list in pairs(sp) do
			for _, v in ipairs(list) do table.insert(Spawns, v) end
		end
	end
	local function place(f, cf, sx, sy, sz, col, mat, name)
		local p = blk(f, 0, 0, 0, sx, sy, sz, col, mat, name)
		p.CFrame = cf
		return p
	end
	local CAR_COLS = { RGB(52, 56, 48), RGB(44, 46, 52), RGB(96, 86, 66), RGB(70, 44, 40), RGB(30, 30, 32), RGB(80, 82, 84), RGB(58, 64, 70) }
	local function car(f, x, z, rotY, col, flip)
		if nearSpawn(x, z, 9) then return end
		col = col or CAR_COLS[MapRand:NextInteger(1, #CAR_COLS)]
		local base = CFrame.new(x, 0, z) * CFrame.Angles(0, math.rad(rotY), 0)
		if flip then base = CFrame.new(x, 2.7, z) * CFrame.Angles(0, math.rad(rotY), math.rad(90)) end
		place(f, base * CFrame.new(0, 1.5, 0), 5, 1.6, 11, col, Enum.Material.Metal, "Car")
		place(f, base * CFrame.new(0, 2.9, 0.6), 4.4, 1.4, 5.4, RGB(18, 22, 28), Enum.Material.SmoothPlastic, "CarCabin")
		place(f, base * CFrame.new(0, 0.8, 3.6), 5.4, 1.6, 1.6, RGB(20, 20, 22), Enum.Material.Metal, "Wheels")
		place(f, base * CFrame.new(0, 0.8, -3.6), 5.4, 1.6, 1.6, RGB(20, 20, 22), Enum.Material.Metal, "Wheels")
	end
	local function truck(f, x, z, rotY, col)
		if nearSpawn(x, z, 12) then return end
		local base = CFrame.new(x, 0, z) * CFrame.Angles(0, math.rad(rotY), 0)
		place(f, base * CFrame.new(0, 2.6, 5), 6.4, 4.4, 14, col, Enum.Material.Metal, "TruckBed")
		place(f, base * CFrame.new(0, 2.4, -5.2), 6.2, 4, 5, RGB(48, 52, 46), Enum.Material.Metal, "TruckCab")
		place(f, base * CFrame.new(0, 3.6, -7.2), 5.4, 1.6, 0.4, RGB(18, 22, 28), Enum.Material.SmoothPlastic, "TruckGlass")
		place(f, base * CFrame.new(0, 0.9, 5), 6.8, 1.8, 2, RGB(20, 20, 22), Enum.Material.Metal, "Wheels")
		place(f, base * CFrame.new(0, 0.9, -5), 6.8, 1.8, 2, RGB(20, 20, 22), Enum.Material.Metal, "Wheels")
	end
	local function pile(f, x, z, n)
		if nearSpawn(x, z, 8) then return end
		for _ = 1, n or 4 do
			local s = R(1.2, 3.6)
			place(f, CFrame.new(x + R(-2.5, 2.5), s * 0.4, z + R(-2.5, 2.5)) * CFrame.Angles(R(-0.5, 0.5), R(0, 3), R(-0.5, 0.5)),
				s * R(1, 1.8), s, s * R(1, 1.6), RGB(70 + MapRand:NextInteger(0, 24), 68 + MapRand:NextInteger(0, 20), 64 + MapRand:NextInteger(0, 18)),
				Enum.Material.Concrete, "Rubble")
		end
	end
	local function sandbags(f, x, z, rotY)
		if nearSpawn(x, z, 8) then return end
		local base = CFrame.new(x, 0, z) * CFrame.Angles(0, math.rad(rotY), 0)
		for layer = 0, 1 do
			for i = 0, 2 - layer do
				place(f, base * CFrame.new((i - 1 + layer * 0.5) * 3.7, 0.55 + layer * 1.1, 0), 3.6, 1.1, 1.7, RGB(112, 100, 74), Enum.Material.Fabric, "Sandbag")
			end
		end
	end
	local function jersey(f, x, z, rotY)
		if nearSpawn(x, z, 7) then return end
		place(f, CFrame.new(x, 1.5, z) * CFrame.Angles(0, math.rad(rotY), 0), 6, 3, 1.8, RGB(104, 102, 98), Enum.Material.Concrete, "Barrier")
	end
	local function pole(f, x, z, lit)
		blk(f, x, 0, z, 0.6, 14, 0.6, RGB(40, 42, 46), Enum.Material.Metal, "Pole")
		blk(f, x + 1.2, 13.6, z, 3, 0.4, 0.5, RGB(40, 42, 46), Enum.Material.Metal, "PoleArm")
		if lit then lamp(f, Vector3.new(x + 2, 13, z), RGB(255, 190, 120), 45, 1.1) end
	end
	local function container(f, x, z, rotY, level, col)
		if nearSpawn(x, z, 10) then return end
		place(f, CFrame.new(x, level * 6.1 + 3, z) * CFrame.Angles(0, math.rad(rotY), 0), 12, 6, 6, col, Enum.Material.CorrodedMetal, "Container")
	end

	-- =================================================================
	-- DOWNTOWN  (id NEON): a huge modern city (660 x 660). 4x4 city blocks around wide avenues with a central plaza, bank, hotel tower,
	-- multi-storey car park, park, construction site, industrial yard, gas station and apartment blocks. Most buildings are enterable with
	-- stairs to the roof, and an elevated overpass runs along the southern avenue. Alpha stages on the +Z edge, Bravo on the -Z edge.
	-- =================================================================
	MapBuilders.NEON = function(f)
		MapRand = Random.new(11)
		local M = Enum.Material
		local FH, SW = 12, 0.8 -- storey height, sidewalk top
		local CEN = { -240, -80, 80, 240 }
		local sp = { Alpha = {}, Bravo = {}, FFA = {} }
		for _, x in ipairs(CEN) do
			table.insert(sp.Alpha, Vector3.new(x, SW, 288))
			table.insert(sp.Bravo, Vector3.new(x, SW, -288))
		end
		for _, p in ipairs({ { -160, -80 }, { 160, -80 }, { -160, 80 }, { 160, 80 }, { -80, -160 }, { 80, -160 }, { -80, 169 }, { 80, 169 },
			{ 0, -90 }, { 0, 90 }, { -90, 0 }, { 90, 0 } }) do
			table.insert(sp.FFA, Vector3.new(p[1], 0.1, p[2]))
		end
		setSpawns(sp)

		local function B(cx, by, cz, sx, sy, sz, col, mat, name) return blk(f, cx, by, cz, sx, sy, sz, col, mat, name) end
		local function ghost(p) p.CanCollide = false; p.CanQuery = false; return p end
		local function ball(cx, cy, cz, d, col, mat, name)
			local p = B(cx, cy - d / 2, cz, d, d, d, col, mat, name)
			p.Shape = Enum.PartType.Ball
			return p
		end
		local function cyl(cx, by, cz, d, h, col, mat, name)
			local p = B(cx, by, cz, d, h, d, col, mat, name)
			p.Shape = Enum.PartType.Cylinder
			p.CFrame = CFrame.new(cx, by + h / 2, cz) * CFrame.Angles(0, 0, math.rad(90))
			p.Size = Vector3.new(h, d, d)
			return p
		end

		-- ---------- palettes ----------
		local STY = {
			{ M.Brick, { RGB(128, 74, 58), RGB(116, 68, 54), RGB(142, 88, 66), RGB(104, 62, 52) } },
			{ M.Concrete, { RGB(150, 148, 142), RGB(134, 134, 132), RGB(160, 154, 144), RGB(120, 122, 124) } },
			{ M.Sandstone, { RGB(176, 156, 124), RGB(166, 148, 120), RGB(150, 136, 112) } },
			{ M.Limestone, { RGB(184, 176, 158), RGB(172, 166, 150) } },
			{ M.Granite, { RGB(96, 100, 104), RGB(84, 88, 94) } },
		}
		local SLABC = RGB(146, 144, 140)
		local GLASSC = RGB(52, 74, 94)
		local ROOFC = RGB(70, 72, 76)

		-- ---------- structural helpers ----------
		local function wallRun(axis, fix, a0, a1, by, h, th, col, mat, ops, name)
			table.sort(ops, function(p, q) return p.c < q.c end)
			local function piece(u0, u1, y0, y1)
				if u1 - u0 < 0.05 or y1 - y0 < 0.05 then return end
				if axis == "x" then
					B((u0 + u1) / 2, by + y0, fix, u1 - u0, y1 - y0, th, col, mat, name or "Wall")
				else
					B(fix, by + y0, (u0 + u1) / 2, th, y1 - y0, u1 - u0, col, mat, name or "Wall")
				end
			end
			local cur = a0
			for _, o in ipairs(ops) do
				local lo, hi = o.c - o.w / 2, o.c + o.w / 2
				piece(cur, lo, 0, h)
				piece(lo, hi, 0, o.y0)
				piece(lo, hi, o.y1, h)
				if o.pane then
					local g
					if axis == "x" then g = B(o.c, by + o.y0, fix, o.w, o.y1 - o.y0, 0.25, RGB(70, 96, 118), M.Glass, "Pane")
					else g = B(fix, by + o.y0, o.c, 0.25, o.y1 - o.y0, o.w, RGB(70, 96, 118), M.Glass, "Pane") end
					g.Transparency = 0.45
					g.CanCollide = false
				end
				cur = hi
			end
			piece(cur, a1, 0, h)
		end
		-- walls' window / door openings along [a0, a1]
		local function openings(a0, a1, doors, ground, spacing, pane)
			local ops = {}
			for _, d in ipairs(doors or {}) do
				table.insert(ops, { c = d, w = 7, y0 = 0, y1 = 9.5 })
			end
			local c = a0 + 7
			while c < a1 - 6 do
				local ok = true
				if ground then
					for _, d in ipairs(doors or {}) do
						if math.abs(c - d) < 8 then ok = false end
					end
				end
				if ok then table.insert(ops, { c = c, w = 5, y0 = 3.2, y1 = 8.4, pane = pane }) end
				c += spacing
			end
			return ops
		end
		local function slabHole(top, xi0, xi1, zi0, zi1, strip, sx0, sx1, col, mat, th)
			th = th or 1
			local by = top - th
			local zs = zi1 - strip
			B((xi0 + xi1) / 2, by, (zi0 + zs) / 2, xi1 - xi0, th, zs - zi0, col, mat or M.Concrete, "Slab")
			if sx0 then
				if sx0 > xi0 + 0.05 then B((xi0 + sx0) / 2, by, (zs + zi1) / 2, sx0 - xi0, th, strip, col, mat or M.Concrete, "Slab") end
				if sx1 < xi1 - 0.05 then B((sx1 + xi1) / 2, by, (zs + zi1) / 2, xi1 - sx1, th, strip, col, mat or M.Concrete, "Slab") end
			else
				B((xi0 + xi1) / 2, by, (zs + zi1) / 2, xi1 - xi0, th, strip, col, mat or M.Concrete, "Slab")
			end
		end
		local function crate(x, by, z, sx, sy, sz, rot)
			return blkR(f, x, by, z, sx, sy, sz, rot or R(0, 90), RGB(92 + MapRand:NextInteger(0, 20), 80 + MapRand:NextInteger(0, 14), 60), M.Wood, "Crate")
		end
		local function roofProps(cx, top, cz, w, d)
			for _ = 1, 3 do
				local x, z = cx + R(-w / 2 + 8, w / 2 - 8), cz + R(-d / 2 + 8, d / 2 - 14)
				B(x, top, z, R(4, 7), R(2, 3.4), R(4, 7), RGB(112, 116, 120), M.Metal, "ACUnit")
			end
			if MapRand:NextNumber() < 0.6 then
				local x, z = cx + R(-w / 4, w / 4), cz + R(-d / 4, d / 6)
				cyl(x, top + 3, z, 6, 7, RGB(104, 78, 56), M.Wood, "WaterTank")
				for _, o in ipairs({ { -2, -2 }, { 2, -2 }, { -2, 2 }, { 2, 2 } }) do B(x + o[1], top, z + o[2], 0.5, 3, 0.5, RGB(60, 60, 64), M.Metal, "TankLeg") end
			end
			ghost(B(cx + R(-8, 8), top, cz + R(-6, 6), 0.4, 11, 0.4, RGB(50, 50, 54), M.Metal, "Antenna"))
		end

		-- ---------- an enterable building (hollow, several storeys, stairs and roof access) ----------
		local function building(cx, cz, w, d, n, sty, opt)
			opt = opt or {}
			local mat = opt.mat or sty[1]
			local cols = sty[2]
			local col = opt.col or cols[MapRand:NextInteger(1, #cols)]
			local th = 1.2
			local X0, X1, Z0, Z1 = cx - w / 2, cx + w / 2, cz - d / 2, cz + d / 2
			local xi0, xi1, zi0, zi1 = X0 + th, X1 - th, Z0 + th, Z1 - th
			local STRIP, RUN = 7, 22
			local doorsS = opt.doorsS or { cx - 12, cx + 12 }
			local doorsN = opt.doorsN or { cx + 12 }
			local doorsW = { cz - 8 }
			local doorsE = { cz - 8 }
			local pane = opt.pane
			local sp0 = opt.spacing or 10
			for fl = 0, n - 1 do
				local L = SW + fl * FH
				local g = fl == 0
				wallRun("x", Z0 + th / 2, X0, X1, L, FH, th, col, mat, openings(X0, X1, g and doorsS or nil, g, sp0, pane))
				wallRun("x", Z1 - th / 2, X0, X1, L, FH, th, col, mat, openings(X0, X1, g and doorsN or nil, g, sp0, pane))
				wallRun("z", X0 + th / 2, Z0 + th, Z1 - th, L, FH, th, col, mat, openings(Z0 + th, Z1 - th, g and doorsW or nil, g, 14, pane))
				wallRun("z", X1 - th / 2, Z0 + th, Z1 - th, L, FH, th, col, mat, openings(Z0 + th, Z1 - th, g and doorsE or nil, g, 14, pane))
				-- one interior partition with two door gaps, plus some furniture / cover
				local px = cx + R(-10, 6)
				local zf1 = zi1 - STRIP
				local half = (zf1 - zi0)
				wallRun("z", px, zi0, zf1, L, FH - 1, 0.8, RGB(168, 164, 154), M.Concrete,
					{ { c = zi0 + half * 0.28, w = 7, y0 = 0, y1 = 9 }, { c = zi0 + half * 0.74, w = 7, y0 = 0, y1 = 9 } }, "Partition")
				for _ = 1, 3 do
					local x = R(xi0 + 6, xi1 - 6)
					local z = R(zi0 + 6, zf1 - 5)
					if math.abs(x - px) > 4 then
						local kind = MapRand:NextInteger(1, 3)
						if kind == 1 then B(x, L, z, R(5, 8), R(2.9, 3.2), 2.6, RGB(96, 76, 56), M.WoodPlanks, "Desk")
						elseif kind == 2 then B(x, L, z, 2.4, R(4.3, 4.6), R(5, 8), RGB(78, 80, 84), M.Metal, "Shelf")
						else crate(x, L, z, 3.2, 3.2, 3.2) end
					end
				end
			end
			-- slabs + stairs
			for k = 1, n do
				local top = SW + k * FH
				local dir = ((k - 1) % 2 == 0) and 1 or -1
				local sx0, sx1
				if dir == 1 then sx0 = xi0 + 3; sx1 = sx0 + RUN else sx1 = xi1 - 3; sx0 = sx1 - RUN end
				slabHole(top, xi0, xi1, zi0, zi1, STRIP, sx0, sx1, SLABC, M.Concrete, 1)
				local zc = zi1 - STRIP / 2
				local L = SW + (k - 1) * FH
				local a = Vector3.new(dir == 1 and sx0 or sx1, L, zc)
				local b = Vector3.new(dir == 1 and sx1 or sx0, top, zc)
				ramp(f, a, b, 5.6, RGB(96, 96, 98), M.Concrete)
				if k == n then
					-- roof: parapet all round
					local ph = 1.5
					B(cx, top, Z0 + th / 2, w, ph, th, col, mat, "Parapet")
					B(cx, top, Z1 - th / 2, w, ph, th, col, mat, "Parapet")
					B(X0 + th / 2, top, cz, th, ph, d - 2 * th, col, mat, "Parapet")
					B(X1 - th / 2, top, cz, th, ph, d - 2 * th, col, mat, "Parapet")
					roofProps(cx, top, cz, w, d)
				end
			end
			-- cornice band + awning over the ground-floor doors
			B(cx, SW + FH - 0.2, Z0 - 0.35, w + 0.8, 0.5, 0.7, RGB(100, 98, 94), M.Concrete, "Cornice")
			for _, dx in ipairs(doorsS) do
				B(dx, SW + 9.8, Z0 - 1.6, 9, 0.4, 3.2, RGB(52, 60, 70), M.Metal, "Awning")
			end
			return { X0 = X0, X1 = X1, Z0 = Z0, Z1 = Z1, top = SW + n * FH }
		end

		-- solid (non-enterable) block with window bands, for skyline mass and cover
		local function solid(cx, cz, w, d, h, sty)
			local col = sty[2][MapRand:NextInteger(1, #sty[2])]
			B(cx, SW, cz, w, h, d, col, sty[1], "Building")
			B(cx, SW + h, cz, w + 1, 1.2, d + 1, RGB(96, 94, 90), M.Concrete, "Cornice")
			for fl = 0, math.floor(h / FH) - 1 do
				local y = SW + fl * FH + 4
				ghost(B(cx, y, cz - d / 2 - 0.05, w - 4, 4.4, 0.3, RGB(34, 44, 56), M.Glass, "Window"))
				ghost(B(cx, y, cz + d / 2 + 0.05, w - 4, 4.4, 0.3, RGB(34, 44, 56), M.Glass, "Window"))
				ghost(B(cx - w / 2 - 0.05, y, cz, 0.3, 4.4, d - 4, RGB(34, 44, 56), M.Glass, "Window"))
				ghost(B(cx + w / 2 + 0.05, y, cz, 0.3, 4.4, d - 4, RGB(34, 44, 56), M.Glass, "Window"))
			end
			roofProps(cx, SW + h + 1.2, cz, w, d)
		end

		-- a car on an elevated deck (y = deck height)
		local function carD(x, y, z, rotY, col)
			if nearSpawn(x, z, 9) then return end
			col = col or CAR_COLS[MapRand:NextInteger(1, #CAR_COLS)]
			local base = CFrame.new(x, y, z) * CFrame.Angles(0, math.rad(rotY), 0)
			place(f, base * CFrame.new(0, 1.5, 0), 5, 1.6, 11, col, M.Metal, "DeckCar")
			place(f, base * CFrame.new(0, 2.9, 0.6), 4.4, 1.4, 5.4, RGB(18, 22, 28), M.SmoothPlastic, "DeckCabin")
			place(f, base * CFrame.new(0, 0.8, 3.6), 5.4, 1.6, 1.6, RGB(20, 20, 22), M.Metal, "DeckWheels")
			place(f, base * CFrame.new(0, 0.8, -3.6), 5.4, 1.6, 1.6, RGB(20, 20, 22), M.Metal, "DeckWheels")
		end

		-- ---------- street props ----------
		local function tree(x, z, s)
			s = s or R(0.9, 1.3)
			B(x, SW, z, 1.3, 5.5 * s, 1.3, RGB(86, 64, 44), M.Wood, "Trunk")
			local c = ball(x, SW + 5 * s + 4.4 * s, z, 9 * s, RGB(54 + MapRand:NextInteger(0, 16), 86 + MapRand:NextInteger(0, 20), 46), M.Grass, "Leaves")
			c.CanCollide = false
		end
		local function bench(x, z, rot)
			if nearSpawn(x, z, 6) then return end
			local c = CFrame.new(x, SW, z) * CFrame.Angles(0, math.rad(rot), 0)
			place(f, c * CFrame.new(0, 1.3, 0), 5, 0.4, 1.6, RGB(110, 80, 52), M.WoodPlanks, "Bench")
			place(f, c * CFrame.new(0, 2.1, -0.7), 5, 1.2, 0.3, RGB(110, 80, 52), M.WoodPlanks, "BenchBack")
			place(f, c * CFrame.new(-2.2, 0.6, 0), 0.4, 1.2, 1.4, RGB(50, 52, 56), M.Metal, "BenchLeg")
			place(f, c * CFrame.new(2.2, 0.6, 0), 0.4, 1.2, 1.4, RGB(50, 52, 56), M.Metal, "BenchLeg")
		end
		local function dumpster(x, z, rot)
			if nearSpawn(x, z, 8) then return end
			place(f, CFrame.new(x, SW + 1.8, z) * CFrame.Angles(0, math.rad(rot), 0), 6, 3.6, 3.2, RGB(46, 84, 62), M.Metal, "Dumpster")
		end
		local function hydrant(x, z)
			B(x, SW, z, 1, 2.4, 1, RGB(170, 40, 34), M.Metal, "Hydrant")
		end
		local function bus(x, z, rot, col)
			if nearSpawn(x, z, 14) then return end
			local c = CFrame.new(x, 0.1, z) * CFrame.Angles(0, math.rad(rot), 0)
			place(f, c * CFrame.new(0, 2.9, 0), 7, 4.6, 22, col or RGB(46, 74, 112), M.Metal, "Bus")
			place(f, c * CFrame.new(0, 3.6, 0), 7.2, 1.6, 20, RGB(24, 30, 38), M.Glass, "BusWindows")
			place(f, c * CFrame.new(0, 1, 7), 7.2, 2, 3, RGB(20, 20, 22), M.Metal, "BusWheels")
			place(f, c * CFrame.new(0, 1, -7), 7.2, 2, 3, RGB(20, 20, 22), M.Metal, "BusWheels")
		end
		local function fence(x0, z0, x1, z1)
			local len = math.sqrt((x1 - x0) ^ 2 + (z1 - z0) ^ 2)
			local mx, mz = (x0 + x1) / 2, (z0 + z1) / 2
			local ang = math.atan2(x1 - x0, z1 - z0)
			local p = place(f, CFrame.new(mx, SW + 3, mz) * CFrame.Angles(0, ang, 0), 0.3, 6, len, RGB(110, 114, 118), M.Metal, "Fence")
			p.Transparency = 0.55
			for i = 0, math.floor(len / 12) do
				local t = i * 12 / len
				B(x0 + (x1 - x0) * t, SW, z0 + (z1 - z0) * t, 0.6, 6.4, 0.6, RGB(70, 72, 76), M.Metal, "FencePost")
			end
		end
		local function streetLight(x, z, lit)
			pole(f, x, z, lit)
		end
		local function billboard(x, z, rotY, text, col)
			B(x - 5, 0.1, z, 0.9, 16, 0.9, RGB(60, 62, 66), M.Metal, "BillPost")
			B(x + 5, 0.1, z, 0.9, 16, 0.9, RGB(60, 62, 66), M.Metal, "BillPost")
			local p = Instance.new("Part")
			p.Anchored = true
			p.Size = Vector3.new(22, 8, 0.8)
			p.CFrame = CFrame.new(x, 20, z) * CFrame.Angles(0, math.rad(rotY), 0)
			p.Color = RGB(20, 22, 26)
			p.Material = M.SmoothPlastic
			p.Name = "Billboard"
			p.Parent = f
			for _, face in ipairs({ Enum.NormalId.Front, Enum.NormalId.Back }) do
				local g = Instance.new("SurfaceGui")
				g.Face = face
				g.SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud
				g.PixelsPerStud = 30
				g.LightInfluence = 0.2
				g.Parent = p
				local t = Instance.new("TextLabel")
				t.Size = UDim2.new(1, 0, 1, 0)
				t.BackgroundTransparency = 1
				t.Text = text
				t.TextScaled = true
				t.Font = Enum.Font.GothamBlack
				t.TextColor3 = col
				t.Parent = g
			end
		end

		-- ---------- ground, roads, sidewalks ----------
		B(0, -3.2, 0, 1500, 3, 1500, RGB(70, 68, 62), M.Ground, "Outskirts") -- far ground, 0.2 below the city ground
		B(0, -1.9, 0, 664, 2, 664, RGB(54, 56, 60), M.Asphalt, "Ground")
		for _, bx in ipairs(CEN) do
			for _, bz in ipairs(CEN) do
				B(bx, 0.1, bz, 130, 0.7, 130, RGB(146, 144, 138), M.Pavement, "Sidewalk")
			end
		end
		-- lane markings
		local YEL, WHT = RGB(206, 170, 50), RGB(196, 194, 186)
		local function mark(cx, cz, sx, sz, col) B(cx, 0.1, cz, sx, 0.08, sz, col, M.SmoothPlastic, "Marking").CanCollide = false end
		for _, rc in ipairs({ -160, 0, 160 }) do
			for t = -300, 300, 20 do
				if math.abs(t - -160) > 22 and math.abs(t) > 22 and math.abs(t - 160) > 22 then
					mark(rc, t, 0.5, 8, YEL)
					mark(t, rc, 8, 0.5, YEL)
				end
			end
			-- edge lines
			for _, s in ipairs({ -13, 13 }) do
				for _, seg in ipairs({ { -300, -178 }, { -142, -22 }, { 22, 142 }, { 178, 300 } }) do
					local mid, len = (seg[1] + seg[2]) / 2, seg[2] - seg[1]
					mark(rc + s, mid, 0.3, len, WHT)
					mark(mid, rc + s, len, 0.3, WHT)
				end
			end
		end
		-- zebra crossings at every intersection
		for _, ix in ipairs({ -160, 0, 160 }) do
			for _, iz in ipairs({ -160, 0, 160 }) do
				for k = -5, 5 do
					local o = k * 2.4
					mark(ix + o, iz - 19, 1.2, 5, WHT)
					mark(ix + o, iz + 19, 1.2, 5, WHT)
					mark(ix - 19, iz + o, 5, 1.2, WHT)
					mark(ix + 19, iz + o, 5, 1.2, WHT)
				end
			end
		end

		-- ---------- central plaza (replaces the main intersection) ----------
		B(0, 0.1, 0, 52, 0.5, 52, RGB(150, 144, 132), M.Cobblestone, "PlazaStone")
		B(0, 0.58, 0, 40, 0.2, 40, RGB(166, 160, 148), M.Pavement, "PlazaInner")
		cyl(0, 0.75, 0, 20, 1.4, RGB(120, 118, 114), M.Granite, "FountainWall")
		local water = cyl(0, 1.3, 0, 17.6, 1.1, RGB(70, 118, 132), M.Glass, "FountainWater")
		water.CanCollide = false
		water.Transparency = 0.3
		B(0, 2.1, 0, 7, 1.4, 7, RGB(124, 122, 118), M.Granite, "MonumentBase")
		B(0, 3.5, 0, 4.2, 26, 4.2, RGB(132, 130, 126), M.Marble, "Monument")
		B(0, 29.5, 0, 2.6, 3, 2.6, RGB(108, 106, 104), M.Granite, "MonumentCap")
		for _, o in ipairs({ { -19, -19 }, { 19, -19 }, { -19, 19 }, { 19, 19 } }) do
			B(o[1], 0.75, o[2], 3.6, 1.1, 3.6, RGB(112, 110, 104), M.Concrete, "Planter")
			ball(o[1], 3.4, o[2], 3.4, RGB(60, 92, 50), M.Grass, "Shrub")
		end
		for _, o in ipairs({ { 0, -15, 0 }, { 0, 15, 180 }, { -15, 0, 270 }, { 15, 0, 90 } }) do bench(o[1], o[2], o[3]) end
		for _, c in ipairs({ { -32, -10, 90 }, { -32, 10, 90 }, { 32, -10, 90 }, { 32, 10, 90 }, { -10, -32, 0 }, { 10, -32, 0 }, { -10, 32, 0 }, { 10, 32, 0 } }) do
			jersey(f, c[1], c[2], c[3])
		end

		-- ---------- generic city block generator ----------
		local function parkingLot(cx, cz, w, d)
			B(cx, SW, cz, w, 0.12, d, RGB(60, 62, 64), M.Asphalt, "LotAsphalt")
			for i = -3, 3 do
				B(cx + i * 7, SW + 0.12, cz - d / 2 + 8, 0.3, 0.05, 11, WHT, M.SmoothPlastic, "Stall").CanCollide = false
				B(cx + i * 7, SW + 0.12, cz + d / 2 - 8, 0.3, 0.05, 11, WHT, M.SmoothPlastic, "Stall").CanCollide = false
			end
			for i = -3, 2 do
				if MapRand:NextNumber() < 0.7 then car(f, cx + i * 7 + 3.5, cz - d / 2 + 8, 90 + 90, nil, false) end
				if MapRand:NextNumber() < 0.7 then car(f, cx + i * 7 + 3.5, cz + d / 2 - 8, 90 + 90, nil, false) end
			end
			streetLight(cx - w / 2 + 3, cz, false)
		end
		local function staging(cx, cz, out)
			-- open staging area at the edge of the map: sandbags, barriers, pallets, trucks
			for i = -2, 2 do
				sandbags(f, cx + i * 22, cz - out * 22, 0)
			end
			jersey(f, cx - 40, cz - out * 8, 90)
			jersey(f, cx + 40, cz - out * 8, 90)
			truck(f, cx - 28, cz + out * 6, 90, RGB(72, 76, 58))
			truck(f, cx + 28, cz + out * 12, 90, RGB(62, 66, 56))
			for _ = 1, 3 do container(f, cx + R(-24, 24), cz + out * R(16, 24), 90, 0, ({ RGB(110, 54, 44), RGB(52, 76, 98), RGB(70, 88, 62), RGB(150, 120, 52) })[MapRand:NextInteger(1, 4)]) end
			fence(cx - 55, cz + out * 28, cx + 55, cz + out * 28)
		end

		local function genericBlock(bx, bz, out, gi, gj)
			local lots = {}
			for _, qx in ipairs({ -1, 1 }) do
				for _, qz in ipairs({ -1, 1 }) do
					table.insert(lots, { x = bx + qx * 30, z = bz + qz * 30, qz = qz })
				end
			end
			local first = true
			for _, lot in ipairs(lots) do
				if out ~= 0 and lot.qz == out then
					staging(lot.x, lot.z, out)
				else
					local r = MapRand:NextNumber()
					local sty = STY[MapRand:NextInteger(1, #STY)]
					if first or r < 0.58 then
						first = false
						local n = MapRand:NextInteger(2, 4)
						building(lot.x, lot.z, 52 + MapRand:NextInteger(0, 2) * 2, 50 + MapRand:NextInteger(0, 2) * 2, n, sty, {})
					elseif r < 0.76 then
						solid(lot.x, lot.z, 50, 50, MapRand:NextInteger(3, 6) * FH, sty)
					elseif r < 0.9 then
						parkingLot(lot.x, lot.z, 52, 52)
					else
						building(lot.x, lot.z, 54, 30, 1, sty, {})
						bus(lot.x, lot.z + 25 * lot.qz, 0, nil)
					end
				end
			end
			-- street dressing around the block
			for t = -50, 50, 25 do
				tree(bx + t, bz - 60.5, nil)
				tree(bx + t, bz + 60.5, nil)
				tree(bx - 60.5, bz + t, nil)
				tree(bx + 60.5, bz + t, nil)
			end
			dumpster(bx - 3, bz + 4, 90)
			hydrant(bx + 62, bz + 26)
			hydrant(bx - 62, bz - 26)
			streetLight(bx - 63, bz - 63, false)
			streetLight(bx + 63, bz + 63, false)
		end

		-- ---------- landmark: the bank ----------
		local function bank(bx, bz)
			local b = building(bx, bz, 78, 66, 3, STY[4], { doorsS = { bx - 14, bx + 14 }, doorsN = { bx + 16 }, pane = true, spacing = 11 })
			-- portico: columns + stepped entrance
			for i = -3, 3 do B(bx + i * 10, SW, b.Z0 - 8, 2.4, 10, 2.4, RGB(196, 190, 176), M.Marble, "Column") end
			B(bx, SW + 10, b.Z0 - 6.6, 76, 1.6, 10, RGB(176, 170, 156), M.Limestone, "Portico")
			B(bx, SW, b.Z0 - 8.6, 76, 0.5, 14, RGB(170, 166, 156), M.Marble, "BankSteps")
			signboard(f, Vector3.new(bx, SW + 13.4, b.Z0 - 1.9), Vector3.new(36, 3.2, 0.3), Enum.NormalId.Front, "FIRST NATIONAL BANK", RGB(222, 196, 120))
			-- armoured cars + barriers out front
			truck(f, bx - 40, b.Z0 - 22, 90, RGB(60, 64, 66))
			jersey(f, bx + 28, b.Z0 - 20, 0)
			jersey(f, bx + 40, b.Z0 - 20, 0)
			tree(bx + 60, bz - 20, nil)
			tree(bx - 60, bz + 20, nil)
			for _, o in ipairs({ { -52, 44 }, { 52, 44 } }) do tree(bx + o[1], bz + o[2], nil) end
		end

		-- ---------- landmark: hotel tower ----------
		local function hotel(bx, bz)
			local b = building(bx, bz + 4, 64, 58, 6, STY[2], { col = RGB(122, 128, 134), pane = true, spacing = 11, doorsS = { bx - 14, bx + 14 }, doorsN = { bx + 16 } })
			-- porte-cochere canopy
			B(bx, SW + 11, b.Z0 - 11, 28, 0.8, 22, RGB(60, 66, 74), M.Metal, "Canopy")
			for _, o in ipairs({ { -12, -19 }, { 12, -19 } }) do B(bx + o[1], SW, b.Z0 + o[2] + 8, 1.2, 10.5, 1.2, RGB(150, 146, 140), M.Concrete, "CanopyPost") end
			signboard(f, Vector3.new(bx, SW + 5 * FH + 5, b.Z0 - 0.45), Vector3.new(30, 4, 0.3), Enum.NormalId.Front, "GRAND HOTEL", RGB(230, 206, 150))
			-- drive + cars + front plaza
			B(bx, SW, b.Z0 - 20, 44, 0.12, 34, RGB(70, 70, 72), M.Asphalt, "Driveway")
			car(f, bx - 14, b.Z0 - 24, 0, RGB(30, 30, 34), false)
			car(f, bx + 14, b.Z0 - 30, 20, RGB(90, 94, 98), false)
			bench(bx - 40, b.Z0 - 8, 0)
			bench(bx + 40, b.Z0 - 8, 0)
			for t = -50, 50, 25 do tree(bx + t, bz - 56, nil) end
			parkingLot(bx - 48, b.Z1 + 18, 28, 26)
			parkingLot(bx + 48, b.Z1 + 18, 28, 26)
		end

		-- ---------- landmark: multi-storey car park ----------
		local function garage(bx, bz)
			local w, d = 108, 100
			local X0, X1, Z0, Z1 = bx - w / 2, bx + w / 2, bz - d / 2, bz + d / 2
			local GH = 10
			local STRIP, RUN = 10, 26
			local col = RGB(142, 140, 136)
			for k = 1, 3 do
				local top = SW + k * GH
				local dir = ((k - 1) % 2 == 0) and 1 or -1
				local sx0, sx1
				if dir == 1 then sx0 = X0 + 4; sx1 = sx0 + RUN else sx1 = X1 - 4; sx0 = sx1 - RUN end
				slabHole(top, X0, X1, Z0, Z1, STRIP, sx0, sx1, col, M.Concrete, 1.4)
				local zc = Z1 - STRIP / 2
				local L = SW + (k - 1) * GH
				ramp(f, Vector3.new(dir == 1 and sx0 or sx1, L, zc), Vector3.new(dir == 1 and sx1 or sx0, top, zc), 8, RGB(104, 104, 106), M.Concrete)
				-- parapet (open sides) + roof rail
				local ph = 1.3
				B(bx, top, Z0 + 0.5, w, ph, 1, RGB(120, 118, 114), M.Concrete, "Parapet")
				B(bx, top, Z1 - 0.5, w, ph, 1, RGB(120, 118, 114), M.Concrete, "Parapet")
				B(X0 + 0.5, top, bz, 1, ph, d - 2, RGB(120, 118, 114), M.Concrete, "Parapet")
				if k == 1 then
					B(X1 - 0.5, top, (Z0 + 1 + (bz - 24)) / 2, 1, ph, (bz - 24) - (Z0 + 1), RGB(120, 118, 114), M.Concrete, "Parapet")
					B(X1 - 0.5, top, ((bz - 12) + Z1 - 1) / 2, 1, ph, (Z1 - 1) - (bz - 12), RGB(120, 118, 114), M.Concrete, "Parapet")
				else
					B(X1 - 0.5, top, bz, 1, ph, d - 2, RGB(120, 118, 114), M.Concrete, "Parapet")
				end
				-- parked cars on every deck
				local zrow = Z0 + 14
				for i = -4, 4 do
					local x = bx + i * 10
					if MapRand:NextNumber() < 0.55 then carD(x, top, zrow, 90, nil) end
				end
				-- cover pillars on the deck and wall stubs
				for _, x in ipairs({ -36, -12, 12, 36 }) do
					B(bx + x, top, bz - 10, 2.4, GH - 1.4, 2.4, RGB(120, 118, 116), M.Concrete, "Column")
				end
				B(bx, top, bz + 12, 14, 3, 1, RGB(120, 118, 114), M.Concrete, "DeckWall")
			end
			-- ground level: columns (ground deck is the sidewalk) + cars
			for _, x in ipairs({ -36, -12, 12, 36 }) do
				for _, z in ipairs({ -24, 8 }) do B(bx + x, SW, bz + z, 2.4, GH - 0.4, 2.4, RGB(120, 118, 116), M.Concrete, "Column") end
			end
			for i = -4, 4 do
				if i ~= 0 then car(f, bx + i * 10, Z0 + 14, 90, nil, false) end
			end
			-- roof props + sign + light
			for k = 1, 3 do lamp(f, Vector3.new(bx, SW + k * GH - 2, bz), RGB(255, 230, 190), 50, 0.7) end
			signboard(f, Vector3.new(bx, SW + 3 * GH + 3.4, Z0 - 0.2), Vector3.new(30, 4, 0.4), Enum.NormalId.Front, "PARKING", RGB(236, 220, 160))
			B(bx - 36, SW + 3 * GH, bz + 16, 8, 3, 6, RGB(112, 116, 120), M.Metal, "ACUnit")
			-- exterior ramp from the street onto the first deck (east side, wide)
			ramp(f, Vector3.new(X1 + 36, 0.1, bz - 18), Vector3.new(X1 + 1, SW + GH, bz - 18), 8, RGB(104, 104, 106), M.Concrete)
			jersey(f, X1 + 20, bz - 28, 0)
			jersey(f, X1 + 20, bz - 8, 0)
		end

		-- ---------- landmark: park ----------
		local function park(bx, bz)
			B(bx, SW, bz, 124, 0.12, 124, RGB(70, 104, 56), M.Grass, "Lawn")
			-- paths
			B(bx, SW + 0.12, bz, 6, 0.06, 124, RGB(168, 160, 142), M.Pavement, "Path")
			B(bx, SW + 0.12, bz, 124, 0.04, 6, RGB(168, 160, 142), M.Pavement, "Path")
			-- pond
			local pond = cyl(bx - 28, SW + 0.12, bz + 28, 30, 0.5, RGB(58, 104, 120), M.Glass, "Pond")
			pond.CanCollide = false
			pond.Transparency = 0.25
			for a = 0, 330, 30 do
				local rx, rz = math.cos(math.rad(a)) * 15.8, math.sin(math.rad(a)) * 15.8
				ball(bx - 28 + rx, SW + 1.2, bz + 28 + rz, 2.6, RGB(112, 108, 100), M.Rock, "PondRock")
			end
			-- pavilion (bandstand)
			cyl(bx + 28, SW, bz - 28, 22, 0.9, RGB(150, 146, 136), M.Concrete, "BandstandBase")
			for a = 0, 315, 45 do
				local rx, rz = math.cos(math.rad(a)) * 9, math.sin(math.rad(a)) * 9
				B(bx + 28 + rx, SW + 0.9, bz - 28 + rz, 0.9, 9, 0.9, RGB(126, 122, 112), M.Concrete, "Post")
			end
			cyl(bx + 28, SW + 9.9, bz - 28, 22, 0.8, RGB(66, 76, 70), M.Slate, "BandstandRoof")
			-- trees
			for i = 1, 40 do
				local x, z = bx + R(-56, 56), bz + R(-56, 56)
				if math.abs(x - bx) > 5 and math.abs(z - bz) > 5 and ((x - bx + 28) ^ 2 + (z - bz - 28) ^ 2) > 20 ^ 2 and ((x - bx - 28) ^ 2 + (z - bz + 28) ^ 2) > 16 ^ 2 then
					tree(x, z, nil)
				end
			end
			for _, o in ipairs({ { 10, 3, 0 }, { -10, 3, 0 }, { 10, -3, 180 }, { -10, -3, 180 }, { 3, 40, 90 }, { 3, -40, 90 } }) do bench(bx + o[1], bz + o[2], o[3]) end
			-- low stone walls as cover
			for _, o in ipairs({ { -44, -20, 0 }, { 44, 20, 0 }, { -20, 46, 90 }, { 20, -46, 90 } }) do
				blkR(f, bx + o[1], SW, bz + o[2], 14, 2.2, 1.6, o[3], RGB(120, 116, 106), M.Cobblestone, "StoneWall")
			end
			-- a statue plinth
			B(bx, SW + 0.2, bz, 5, 2, 5, RGB(126, 124, 120), M.Granite, "Plinth")
			B(bx, SW + 2.2, bz, 1.6, 7, 1.6, RGB(80, 98, 90), M.Metal, "Statue")
		end

		-- ---------- landmark: construction site ----------
		local function construction(bx, bz, out)
			local X0, X1 = bx - 36, bx + 36
			local Z0, Z1 = bz - 28, bz + 28
			local CH = 11
			local STRIP, RUN = 8, 22
			-- concrete core + columns + open floors
			for k = 1, 4 do
				local top = SW + k * CH
				local dir = ((k - 1) % 2 == 0) and 1 or -1
				local sx0, sx1
				if dir == 1 then sx0 = X0 + 3; sx1 = sx0 + RUN else sx1 = X1 - 3; sx0 = sx1 - RUN end
				slabHole(top, X0, X1, Z0, Z1, STRIP, sx0, sx1, RGB(130, 128, 124), M.Concrete, 1.2)
				local zc = Z1 - STRIP / 2
				ramp(f, Vector3.new(dir == 1 and sx0 or sx1, SW + (k - 1) * CH, zc), Vector3.new(dir == 1 and sx1 or sx0, top, zc), 5.4, RGB(96, 96, 98), M.Concrete)
				for _, x in ipairs({ -30, -10, 10, 30 }) do
					for _, z in ipairs({ -22, 0 }) do
						B(bx + x, top - CH + 0.0, bz + z, 1.8, CH - 1.2, 1.8, RGB(120, 118, 114), M.Concrete, "Column")
					end
				end
				-- temporary safety rail on the edge (front)
				B(bx, top, Z0 + 0.3, X1 - X0, 1.2, 0.3, RGB(226, 150, 40), M.Metal, "SafetyRail")
			end
			-- scaffolding on the west face
			for k = 0, 3 do
				B(X0 - 3.5, SW + k * CH, bz, 7, 0.4, 56, RGB(110, 112, 116), M.DiamondPlate, "Scaffold")
				for _, z in ipairs({ -26, -13, 0, 13, 26 }) do B(X0 - 6.6, SW + k * CH, bz + z, 0.5, CH, 0.5, RGB(120, 122, 126), M.Metal, "ScaffoldPost") end
			end
			ramp(f, Vector3.new(X0 - 30, SW, bz), Vector3.new(X0 - 6, SW + CH + 0.4, bz), 5, RGB(104, 100, 90), M.WoodPlanks)
			-- tower crane
			local cx, cz = bx + 46, bz + 20
			B(cx, SW, cz, 3, 62, 3, RGB(214, 170, 44), M.Metal, "CraneMast")
			B(cx - 18, SW + 62, cz, 54, 2, 2.4, RGB(214, 170, 44), M.Metal, "CraneJib")
			B(cx + 14, SW + 62, cz, 14, 1.6, 2, RGB(214, 170, 44), M.Metal, "CraneCounter")
			B(cx + 12, SW + 56, cz, 5, 5, 3, RGB(90, 90, 92), M.Concrete, "CraneWeight")
			ghost(B(cx - 34, SW + 40, cz, 0.3, 22, 0.3, RGB(40, 40, 44), M.Metal, "CraneCable"))
			-- materials, mixers, pallets and fences
			for i = 0, 3 do B(bx - 30 + i * 6.5, SW, bz - 44, 5, 5, 5, RGB(150, 142, 126), M.Concrete, "ConcretePipe") end
			for i = 0, 2 do crate(bx + 18 + i * 5, SW, bz - 44, 4, 3, 4, 0) end
			cyl(bx - 12, SW + 1.8, bz + 44, 4.4, 4.6, RGB(214, 170, 44), M.Metal, "Mixer")
			container(f, bx + 34, bz - 40, 0, 0, RGB(52, 76, 98))
			container(f, bx - 52, bz + 36, 90, 0, RGB(150, 120, 52))
			pile(f, bx - 40, bz - 40, 5)
			pile(f, bx + 46, bz - 18, 4)
			jersey(f, bx - 20, bz - 52, 0)
			jersey(f, bx + 20, bz - 52, 0)
			jersey(f, bx + 56, bz + 4, 90)
			fence(bx - 62, bz - 62, bx - 62, bz + 62)
			fence(bx - 62, bz - 62, bx - 20, bz - 62)
		end

		-- ---------- landmark: industrial yard ----------
		local function industrial(bx, bz, out)
			-- big warehouse with a mezzanine catwalk
			local W, D = 80, 48
			local wx, wz = bx - 10, bz + 22
			local X0, X1, Z0, Z1 = wx - W / 2, wx + W / 2, wz - D / 2, wz + D / 2
			local th = 1.2
			local H = 18
			local col, mat = RGB(98, 108, 112), M.CorrodedMetal
			wallRun("x", Z0 + th / 2, X0, X1, SW, H, th, col, mat, { { c = wx - 14, w = 14, y0 = 0, y1 = 12 }, { c = wx + 16, w = 14, y0 = 0, y1 = 12 } })
			wallRun("x", Z1 - th / 2, X0, X1, SW, H, th, col, mat, { { c = wx, w = 12, y0 = 0, y1 = 11 } })
			wallRun("z", X0 + th / 2, Z0 + th, Z1 - th, SW, H, th, col, mat, { { c = wz, w = 10, y0 = 0, y1 = 10 } })
			wallRun("z", X1 - th / 2, Z0 + th, Z1 - th, SW, H, th, col, mat, { { c = wz, w = 10, y0 = 0, y1 = 10 } })
			B(wx, SW + H, wz, W + 1, 1.2, D + 1, RGB(66, 70, 72), M.Metal, "WarehouseRoof")
			-- high clerestory windows
			for i = -3, 3 do
				ghost(B(wx + i * 11, SW + 13, Z0 - 0.05, 6, 3, 0.3, RGB(40, 52, 62), M.Glass, "Window"))
			end
			-- mezzanine along the back with ramp
			B(wx + 8, SW + 7, Z1 - 7, W - 22, 0.8, 12, RGB(104, 106, 108), M.DiamondPlate, "Mezzanine")
			ramp(f, Vector3.new(wx - 37, SW, Z1 - 7), Vector3.new(wx - 21, SW + 7.8, Z1 - 7), 5, RGB(100, 102, 104), M.DiamondPlate)
			B(wx + 8, SW + 7.8, Z1 - 1.5, W - 22, 1.1, 0.3, RGB(214, 170, 44), M.Metal, "Rail")
			-- interior: racks and crates
			for _, x in ipairs({ -22, -4, 14, 32 }) do
				B(wx + x, SW, wz - 8, 2.4, 8, 14, RGB(70, 90, 120), M.Metal, "Rack")
			end
			for _ = 1, 6 do crate(wx + R(-34, 34), SW, wz + R(-18, 4), R(3, 4.5), R(3, 4.5), R(3, 4.5)) end
			signboard(f, Vector3.new(wx, SW + 15, Z0 - 0.2), Vector3.new(26, 3, 0.4), Enum.NormalId.Front, "WAREHOUSE 7", RGB(226, 190, 90))
			-- container yard (stacks reachable by ramps)
			local ccols = { RGB(110, 54, 44), RGB(52, 76, 98), RGB(70, 88, 62), RGB(150, 120, 52), RGB(88, 88, 92) }
			for row = 0, 2 do
				for i = 0, 3 do
					local x, z = bx + 22 + i * 13, bz - 30 + row * 8
					local lvl = ((row + i) % 3 == 0) and 1 or 0
					container(f, x, z, 0, 0, ccols[MapRand:NextInteger(1, #ccols)])
					if lvl == 1 then container(f, x, z, 0, 1, ccols[MapRand:NextInteger(1, #ccols)]) end
				end
			end
			ramp(f, Vector3.new(bx + 22 - 6 - 14.5, SW, bz - 14), Vector3.new(bx + 22 - 6.2, SW + 6.1, bz - 14), 5, RGB(100, 90, 70), M.WoodPlanks)
			-- gantry crane
			for _, x in ipairs({ bx - 52, bx - 22 }) do B(x, SW, bz - 34, 2, 22, 2, RGB(214, 170, 44), M.Metal, "GantryLeg") end
			B(bx - 37, SW + 22, bz - 34, 34, 2, 2.6, RGB(214, 170, 44), M.Metal, "GantryBeam")
			-- fuel tanks
			for i = 0, 1 do
				cyl(bx - 50 + i * 12, SW, bz + 54, 9, 12, RGB(120, 124, 126), M.Metal, "Tank")
			end
			fence(bx - 62, bz - 62, bx + 62, bz - 62)
			jersey(f, bx - 38, bz - 8, 0)
			jersey(f, bx - 30, bz - 8, 0)
			truck(f, bx + 50, bz + 10, 0, RGB(70, 74, 60))
			pile(f, bx - 56, bz - 10, 4)
		end

		-- ---------- landmark: gas station, diner and shops ----------
		local function gasStation(bx, bz, out)
			-- canopy on four pillars
			local cx, cz = bx - 22, bz - 14
			B(cx, SW + 10, cz, 40, 1.2, 24, RGB(190, 40, 36), M.Metal, "Canopy")
			B(cx, SW + 11.2, cz, 40.4, 0.4, 24.4, RGB(220, 218, 210), M.Metal, "CanopyTrim")
			for _, o in ipairs({ { -16, -8 }, { 16, -8 }, { -16, 8 }, { 16, 8 } }) do B(cx + o[1], SW, cz + o[2], 1.4, 10, 1.4, RGB(180, 176, 168), M.Metal, "CanopyPillar") end
			for _, o in ipairs({ { -8, -3 }, { -8, 3 }, { 8, -3 }, { 8, 3 } }) do
				B(cx + o[1], SW + 0.1, cz + o[2], 1.6, 4.6, 1.2, RGB(70, 74, 80), M.Metal, "Pump")
				B(cx + o[1], SW, cz + o[2], 6, 0.5, 3.5, RGB(120, 118, 112), M.Concrete, "PumpIsland")
			end
			lamp(f, Vector3.new(cx, SW + 8.6, cz), RGB(255, 236, 200), 45, 0.8)
			signboard(f, Vector3.new(cx, SW + 14, cz - 12), Vector3.new(16, 3, 0.4), Enum.NormalId.Front, "GAS  24H", RGB(240, 220, 150))
			B(cx, SW, cz - 12.5, 0.8, 14, 0.8, RGB(70, 72, 76), M.Metal, "SignPost")
			-- convenience store
			building(bx + 34, bz - 20, 52, 30, 1, STY[2], { col = RGB(172, 166, 148), doorsS = { bx + 26, bx + 44 }, doorsN = { bx + 40 } })
			-- diner (two storeys, brick)
			building(bx + 34, bz + 22, 56, 34, 2, STY[1], { doorsS = { bx + 22, bx + 44 }, doorsN = { bx + 46 } })
			-- parked and wrecked-looking cars, dumpsters
			car(f, bx - 40, bz + 12, 90, RGB(58, 64, 70), false)
			car(f, bx - 20, bz + 22, 0, RGB(120, 40, 36), false)
			car(f, bx - 52, bz - 20, 90, nil, false)
			dumpster(bx + 4, bz + 6, 0)
			dumpster(bx + 54, bz - 6, 90)
			bench(bx + 2, bz - 30, 0)
			hydrant(bx - 4, bz - 56)
			for t = -50, 20, 24 do tree(bx + t, bz + 52, nil) end
		end

		-- ---------- landmark: apartment row ----------
		local function apartments(bx, bz, out)
			building(bx - 30, bz - 28, 54, 50, 4, STY[1], { col = RGB(122, 72, 56) })
			building(bx + 32, bz - 28, 54, 50, 3, STY[1], { col = RGB(108, 64, 52) })
			building(bx - 30, bz + 24, 54, 40, 3, STY[3], {})
			building(bx + 32, bz + 24, 54, 40, 3, STY[3], {})
			for _, o in ipairs({ { -30, 4 }, { 30, 4 }, { 0, -2 }, { 0, 60 } }) do tree(bx + o[1], bz + o[2], nil) end
			dumpster(bx + 2, bz - 5, 0)
			car(f, bx - 4, bz + 40, 90, nil, false)
		end

		-- ---------- assemble the 16 blocks ----------
		for i = 1, 4 do
			for j = 1, 4 do
				local bx, bz = CEN[i], CEN[j]
				local out = (j == 1) and -1 or ((j == 4) and 1 or 0)
				if i == 1 and j == 1 then industrial(bx, bz, -1)
				elseif i == 4 and j == 1 then construction(bx, bz, -1)
				elseif i == 1 and j == 4 then gasStation(bx, bz, 1)
				elseif i == 4 and j == 4 then apartments(bx, bz, 1)
				elseif i == 2 and j == 2 then bank(bx, bz)
				elseif i == 3 and j == 2 then hotel(bx, bz)
				elseif i == 2 and j == 3 then garage(bx, bz)
				elseif i == 3 and j == 3 then park(bx, bz)
				else genericBlock(bx, bz, out, i, j) end
			end
		end
		-- staging rows for the corner landmarks too (open space around the spawns)
		for _, x in ipairs({ -240, 240 }) do
			for _, z in ipairs({ -1, 1 }) do
				sandbags(f, x - 30, z * 292, 0)
				sandbags(f, x + 30, z * 292, 0)
				jersey(f, x - 14, z * 276, 90)
				jersey(f, x + 14, z * 276, 90)
			end
		end

		-- ---------- elevated overpass along the southern avenue (z = 160) ----------
		local OZ = 160
		B(0, 12.5, OZ, 340, 1.5, 14, RGB(112, 112, 112), M.Concrete, "OverpassDeck")
		B(0, 14, OZ - 6.6, 340, 1.2, 0.8, RGB(96, 96, 98), M.Concrete, "OverpassRail")
		B(0, 14, OZ + 6.6, 340, 1.2, 0.8, RGB(96, 96, 98), M.Concrete, "OverpassRail")
		for _, px in ipairs({ -136, -102, -68, -34, 34, 68, 102, 136 }) do
			B(px, 0.1, OZ - 5, 3, 12.5, 3, RGB(120, 120, 118), M.Concrete, "OverpassPier")
			B(px, 0.1, OZ + 5, 3, 12.5, 3, RGB(120, 120, 118), M.Concrete, "OverpassPier")
			B(px, 10.8, OZ, 3.6, 1.7, 14, RGB(118, 118, 116), M.Concrete, "PierCap")
		end
		ramp(f, Vector3.new(-222, 0.1, OZ), Vector3.new(-170, 14, OZ), 14, RGB(112, 112, 112), M.Concrete)
		ramp(f, Vector3.new(222, 0.1, OZ), Vector3.new(170, 14, OZ), 14, RGB(112, 112, 112), M.Concrete)
		jersey(f, -168, OZ - 5, 0)
		jersey(f, 168, OZ + 5, 0)
		for _, x in ipairs({ -120, -60, 60, 120 }) do
			carD(x, 14, OZ - 3, 90, nil)
		end
		-- road traffic: abandoned cars, buses, trucks and barriers on the avenues
		local function roadCars(isNS, fixed)
			for t = -290, 290, 26 do
				if math.abs(t + 160) > 24 and math.abs(t) > 30 and math.abs(t - 160) > 24 and MapRand:NextNumber() < 0.5 then
					local side = (MapRand:NextNumber() < 0.5) and -6 or 6
					if isNS then car(f, fixed + side + R(-1, 1), t, (side < 0 and 0 or 180) + R(-6, 6), nil, MapRand:NextNumber() < 0.04)
					else car(f, t, fixed + side + R(-1, 1), 90 + (side < 0 and 0 or 180) + R(-6, 6), nil, MapRand:NextNumber() < 0.04) end
				end
			end
		end
		for _, rc in ipairs({ -160, 0, 160 }) do
			roadCars(true, rc)
			if rc ~= 160 then roadCars(false, rc) end
		end
		bus(-160, -60, 0, RGB(46, 74, 112))
		bus(100, 0, 90, RGB(130, 100, 40))
		bus(-60, 160, 90, RGB(120, 54, 48))
		truck(f, -100, -160, 90, RGB(70, 76, 58))
		truck(f, 160, 40, 0, RGB(62, 66, 58))
		-- roadblocks / checkpoints on the avenues
		for _, c in ipairs({ { -160, -120 }, { 160, 120 }, { 120, -160 }, { -120, 0 } }) do
			jersey(f, c[1] - 8, c[2], 90)
			jersey(f, c[1] + 8, c[2], 90)
			sandbags(f, c[1], c[2] + 4, 0)
		end

		-- ---------- billboards at the plaza approaches ----------
		billboard(0, -34, 0, "BLACKOUT", RGB(236, 220, 170))
		billboard(0, 34, 180, "DOWNTOWN", RGB(236, 220, 170))

		-- ---------- skyline backdrop outside the play area ----------
		local skyRng = Random.new(5)
		for t = -380, 380, 50 do
			for _, side in ipairs({ { 1, 0 }, { -1, 0 }, { 0, 1 }, { 0, -1 } }) do
				local dist = 360 + skyRng:NextNumber(0, 40)
				local w, d = skyRng:NextNumber(34, 60), skyRng:NextNumber(34, 60)
				local h = skyRng:NextNumber(70, 230)
				local x = side[1] ~= 0 and side[1] * dist or t
				local z = side[2] ~= 0 and side[2] * dist or t
				local sty = STY[skyRng:NextInteger(1, #STY)]
				local tower = B(x, -2, z, w, h + 2, d, GLASSC:Lerp(sty[2][1], 0.45), (skyRng:NextNumber() < 0.5) and M.Concrete or M.Glass, "Skyline")
				tower.CastShadow = false
				for k = 1, 3 do
					local band = B(x, 12 + k * (h / 4), z, w + 0.4, 2.2, d + 0.4, RGB(36, 46, 58), M.Glass, "SkylineBand")
					band.CanCollide = false
					band.CanQuery = false
				end
			end
		end

		-- helpers place vehicles / barriers at y = 0: lift the ones standing on a raised sidewalk
		local LIFT = { Car = 1, CarCabin = 1, Wheels = 1, TruckBed = 1, TruckCab = 1, TruckGlass = 1, Barrier = 1, Sandbag = 1, Rubble = 1,
			Container = 1, Pole = 1, PoleArm = 1, Bus = 0.7, BusWindows = 0.7, BusWheels = 0.7 }
		for _, d in ipairs(f:GetChildren()) do
			if LIFT[d.Name] then
				local p = d.Position
				local inX, inZ = false, false
				for _, c in ipairs(CEN) do
					if math.abs(p.X - c) <= 65 then inX = true end
					if math.abs(p.Z - c) <= 65 then inZ = true end
				end
				if inX and inZ then d.Position = p + Vector3.new(0, (LIFT[d.Name] == 1) and SW or LIFT[d.Name], 0) end
			end
		end

		return { spawns = sp, half = 320 }
	end

	-- =================================================================
	-- LUXURY BOAT  (id YACHT): two superyachts moored side by side at sunset, joined by gangways and an upper bridge. 4v4 but built at 6v6 scale
	-- (130 x 340). Each yacht: pointed bow, helipad / hot-tub foredeck, tiered white superstructure with blue glass, teak aft deck with pool,
	-- an engine-room flank below. Alpha spawns on both sterns, Bravo on both bows. Long axis = Z.
	-- =================================================================
	MapBuilders.YACHT = function(f)
		MapRand = Random.new(21)
		-- the template Baseplate (and its grey checker) would cover the sea and the engine room: drop it well below the water
		for _, inst in ipairs(game:GetService("Workspace"):GetChildren()) do
			if inst:IsA("BasePart") and inst.Name == "Baseplate" and inst.Position.Y > -60 then
				inst.Position = Vector3.new(inst.Position.X, -80, inst.Position.Z)
			end
		end
		-- =====================================================================================================
		-- HIJACKED-STYLE SUPERYACHT (day): 300 studs long, 68 wide, four levels.
		--   z: stern (+) ... bow (-)   x: starboard (+) ... port (-)
		--   y:  -9 engine floor | 0 stern + bow decks | 4 centre deck podium | 14 upper deck | 24 flybridge roof
		-- =====================================================================================================
		local SM, MET, WOOD, PLANK = Enum.Material.SmoothPlastic, Enum.Material.Metal, Enum.Material.Wood, Enum.Material.WoodPlanks
		local FAB, MARB, GRAN, DIA, CONC, GLS, LEA, SLATE = Enum.Material.Fabric, Enum.Material.Marble, Enum.Material.Granite, Enum.Material.DiamondPlate,
			Enum.Material.Concrete, Enum.Material.Glass, Enum.Material.Leather, Enum.Material.Slate
		local HULL, TRIM, NAVY, BLUE = RGB(206, 204, 196), RGB(222, 220, 212), RGB(28, 54, 98), RGB(44, 102, 172)
		local TEAK, DTEAK = RGB(150, 98, 60), RGB(96, 60, 38)
		local CREAM, SAND, CHROME, STEEL, DARK = RGB(216, 202, 178), RGB(188, 170, 142), RGB(176, 180, 188), RGB(86, 92, 100), RGB(34, 36, 42)
		local GLASSC, TURQ, WARM = RGB(118, 158, 178), RGB(64, 168, 178), RGB(255, 226, 180)
		local ORANGE, PURPLE, WOODP, WOODL = RGB(255, 140, 50), RGB(150, 100, 255), RGB(132, 80, 48), RGB(176, 124, 80)
		local SEAY = -14
		local Y0, Y1, Y2, Y3, YE = 0, 4, 14, 24, -9
		local pools = {}

		local function B(cx, by, cz, sx, sy, sz, col, mat, name) return blk(f, cx, by, cz, sx, sy, sz, col, mat, name) end
		local function slab(x0, x1, z0, z1, by, sy, col, mat, name) return blk(f, (x0 + x1) / 2, by, (z0 + z1) / 2, x1 - x0, sy, z1 - z0, col, mat, name) end
		local function ghost(p) p.CanCollide, p.CanQuery = false, false return p end
		local function shoot(p) p.CanQuery = false return p end
		local function deco(p) p.CanCollide = false return p end
		local function glass(cx, by, cz, sx, sy, sz, tr)
			local p = B(cx, by, cz, sx, sy, sz, GLASSC, GLS, "Glass")
			p.Transparency, p.Reflectance, p.CastShadow = tr or 0.55, 0.06, false
			p.CanQuery = false
			return p
		end
		local function cyl(cx, by, cz, h, d, col, mat, name) -- vertical cylinder
			local p = B(cx, by, cz, h, d, d, col, mat, name)
			p.Shape = Enum.PartType.Cylinder
			p.CFrame = CFrame.new(cx, by + h / 2, cz) * CFrame.Angles(0, 0, math.pi / 2)
			return p
		end
		local function ball(cx, cy, cz, sx, sy, sz, col, mat, name)
			local p = place(f, CFrame.new(cx, cy, cz), sx, sy, sz, col, mat, name)
			p.Shape = Enum.PartType.Ball
			return p
		end
		local function tube(a, b, d, col, mat, name) -- cylinder between two points
			local mid, L = (a + b) / 2, (b - a).Magnitude
			local p = place(f, CFrame.lookAt(mid, b) * CFrame.Angles(0, math.pi / 2, 0), L, d, d, col, mat or SM, name or "Tube")
			p.Shape = Enum.PartType.Cylinder
			return p
		end
		local function seg(x0, z0, x1, z1, by, h, th, col, mat, name) -- wall piece between two plan points
			local mx, mz = (x0 + x1) / 2, (z0 + z1) / 2
			local L = math.sqrt((x1 - x0) ^ 2 + (z1 - z0) ^ 2) + 0.2
			return place(f, CFrame.lookAt(Vector3.new(mx, by + h / 2, mz), Vector3.new(x1, by + h / 2, z1)), th, h, L, col, mat, name)
		end
		local function arc(cx, cz, r, a0, a1, by, h, th, col, mat, name, step)
			local n = math.max(2, math.ceil(math.rad(math.abs(a1 - a0)) * r / (step or 3.2)))
			local out = {}
			for i = 0, n - 1 do
				local t0, t1 = math.rad(a0 + (a1 - a0) * i / n), math.rad(a0 + (a1 - a0) * (i + 1) / n)
				table.insert(out, seg(cx + r * math.cos(t0), cz + r * math.sin(t0), cx + r * math.cos(t1), cz + r * math.sin(t1), by, h, th, col, mat, name))
			end
			return out
		end
		-- straight wall with doors (full height up to 7.6) and windows (sill..head, glass pane); axis "x": runs along x at z=fix; axis "z": along z at x=fix
		local function wall(axis, fix, a0, a1, by, h, col, mat, name, doors, wins, th)
			th = th or 0.8
			local ops = {}
			for _, d in ipairs(doors or {}) do table.insert(ops, { d[1], d[2], 0, d[3] or 7.6, false }) end
			for _, w in ipairs(wins or {}) do table.insert(ops, { w[1], w[2], w[3] or 2.8, w[4] or 7.4, true }) end
			table.sort(ops, function(p, q) return p[1] < q[1] end)
			local function piece(u0, u1, y0, y1)
				if u1 - u0 < 0.05 or y1 - y0 < 0.05 then return end
				if axis == "x" then B((u0 + u1) / 2, by + y0, fix, u1 - u0, y1 - y0, th, col, mat, name)
				else B(fix, by + y0, (u0 + u1) / 2, th, y1 - y0, u1 - u0, col, mat, name) end
			end
			local cur = a0
			for _, o in ipairs(ops) do
				piece(cur, o[1], 0, h)
				if o[3] > 0 then piece(o[1], o[2], 0, o[3]) end
				piece(o[1], o[2], o[4], h)
				if o[5] then
					if axis == "x" then glass((o[1] + o[2]) / 2, by + o[3], fix, o[2] - o[1], o[4] - o[3], 0.25)
					else glass(fix, by + o[3], (o[1] + o[2]) / 2, 0.25, o[4] - o[3], o[2] - o[1]) end
				end
				cur = o[2]
			end
			piece(cur, a1, 0, h)
		end
		-- navy pinstripe pair on an exterior wall (above door lintels so doors stay clear)
		local function pins(axis, fix, a0, a1, y, off)
			for _, dy in ipairs({ 0, 0.7 }) do
				if axis == "x" then ghost(B((a0 + a1) / 2, y + dy, fix, a1 - a0, 0.32, 1.0 + (off or 0), BLUE, SM, "Pin"))
				else ghost(B(fix, y + dy, (a0 + a1) / 2, 1.0 + (off or 0), 0.32, a1 - a0, BLUE, SM, "Pin")) end
			end
		end
		-- walkable ramp (invisible) + visual steps
		local function stair(a, b, width, riserCol, treadCol, treadMat)
			local rise = b.Y - a.Y
			local flat = Vector3.new(b.X - a.X, 0, b.Z - a.Z)
			local run = flat.Magnitude
			local dir = flat.Unit
			local n = math.max(3, math.ceil(math.abs(rise) / 0.62))
			local r = ramp(f, a, b, width - 0.2, STEEL, MET)
			r.Transparency = 1
			local vw = width - 0.5 -- visual steps stay clear of the side walls (no flickering shared faces)
			local low = math.min(a.Y, b.Y) - 1
			for i = 1, n do
				local t = (i - 0.5) / n
				local top = a.Y + rise * t
				local c = Vector3.new(a.X, 0, a.Z) + dir * (run * t)
				local cf = CFrame.lookAt(Vector3.new(c.X, (top + low) / 2, c.Z), Vector3.new(c.X, (top + low) / 2, c.Z) + dir)
				deco(place(f, cf, vw, top - low, run / n + 0.04, riserCol or CREAM, SM, "StairStep"))
				deco(place(f, CFrame.lookAt(Vector3.new(c.X, top + 0.1, c.Z), Vector3.new(c.X, top + 0.1, c.Z) + dir), vw, 0.2, run / n + 0.3, treadCol or DTEAK, treadMat or PLANK, "Tread"))
			end
			if width <= 9 then -- handrails + posts on narrow stairs
				local side = Vector3.new(-dir.Z, 0, dir.X)
				for _, sg in ipairs({ -1, 1 }) do
					local off = side * (sg * (vw / 2 - 0.2))
					deco(tube(a + off + Vector3.new(0, 3.1, 0), b + off + Vector3.new(0, 3.1, 0), 0.28, CHROME, MET, "HandRail"))
					for k = 0, math.floor(run / 5) do
						local t = math.min(1, k * 5 / run)
						local pp = a + (b - a) * t + off
						deco(B(pp.X, pp.Y, pp.Z, 0.22, 3.1, 0.22, CHROME, MET, "HandRailPost"))
					end
				end
			end
			return r
		end
		local function lampc(x, y, z, col, range, br) lamp(f, Vector3.new(x, y, z), col or RGB(255, 236, 205), range or 34, (br or 0.8) * 0.4) end
		local function led(x, y, z, sx, sy, sz, col) -- glowing strip (no collision)
			local p = ghost(B(x, y, z, sx, sy, sz, col, Enum.Material.Neon, "LED"))
			p.CastShadow = false
			return p
		end

		-- ----- hull outline (half beam by z) -----
		local HP = { { -152, 0 }, { -146, 6 }, { -132, 16 }, { -116, 26 }, { -100, 32 }, { -84, 34 }, { 72, 34 }, { 92, 33 }, { 106, 29 }, { 116, 22 }, { 122, 12 }, { 124, 0 } }
		local function hw(z)
			if z <= HP[1][1] then return 0 end
			for i = 2, #HP do
				if z <= HP[i][1] then
					local a, b = HP[i - 1], HP[i]
					return a[2] + (b[2] - a[2]) * (z - a[1]) / (b[1] - a[1])
				end
			end
			return 0
		end
		local function deckY(z) if z >= 56 then return Y0 elseif z >= -70 then return Y1 end return Y0 end

		-- ----- ocean (tiles of 2000; top at SEAY) -----
		for _, ox in ipairs({ -2000, 0, 2000 }) do
			for _, oz in ipairs({ -2000, 0, 2000 }) do
				local s = ghost(slab(ox - 1000, ox + 1000, oz - 1000, oz + 1000, SEAY - 2, 2, RGB(16, 58, 98), SM, "Ocean"))
				s.Reflectance, s.CastShadow = 0.12, false
			end
		end

		-- ----- hull, bulwark, rails, barrier -----
		do
			local z = -152
			while z < 124 do
				local z1 = math.min(z + 5, 124)
				local zm = (z + z1) / 2
				local top = deckY(zm)
				for _, sx in ipairs({ -1, 1 }) do
					local x0, x1 = sx * hw(z), sx * hw(z1)
					seg(x0, z, x1, z1, SEAY, top - SEAY, 1.4, HULL, SM, "Hull")
					ghost(seg(x0 * 1.0, z, x1, z1, SEAY, 1.9, 1.7, NAVY, SM, "HullStripe"))
					ghost(seg(x0, z, x1, z1, SEAY, 0.25, 3.6, TRIM, SM, "Foam")).Transparency = 0.35
					ghost(seg(x0, z, x1, z1, top - 3.6, 0.34, 1.6, BLUE, SM, "HullPin"))
					ghost(seg(x0, z, x1, z1, top - 4.3, 0.34, 1.6, BLUE, SM, "HullPin"))
					-- invisible wall so nobody can leave the ship
					local bw = seg(x0, z, x1, z1, top, 16, 1.0, HULL, SM, "Barrier")
					bw.Transparency, bw.CanQuery, bw.CastShadow = 1, false, false
					if zm >= 56 then
						-- stern bulwark: tall white wing wall with a teak cap
						seg(x0 * 0.985, z, x1 * 0.985, z1, top, 4.6, 1.2, HULL, SM, "Bulwark")
						seg(x0 * 0.985, z, x1 * 0.985, z1, top + 4.6, 0.4, 1.7, TEAK, WOOD, "BulwarkCap")
						shoot(deco(seg(x0 * 0.985, z, x1 * 0.985, z1, top + 5.0, 0.35, 0.4, CHROME, MET, "Rail")))
					else
						local pn = shoot(seg(x0 * 0.99, z, x1 * 0.99, z1, top + 0.6, 3.2, 0.2, GLASSC, GLS, "RailGlass"))
						pn.Transparency, pn.CanCollide = 0.6, false
						deco(seg(x0 * 0.99, z, x1 * 0.99, z1, top + 3.8, 0.35, 0.45, CHROME, MET, "Rail"))
						deco(B(x1 * 0.99, top, z1, 0.4, 4.1, 0.4, CHROME, MET, "RailPost"))
					end
				end
				z = z1
			end
			-- portholes along the hull
			for _, sx in ipairs({ -1, 1 }) do
				for pz = -130, 100, 11 do
					local px = sx * (hw(pz) + 0.75)
					local ph = ghost(B(px, deckY(pz) - 7.5, pz, 0.3, 1.5, 1.5, RGB(30, 44, 60), GLS, "Porthole"))
					ph.Shape = Enum.PartType.Cylinder
					ph.Size = Vector3.new(0.3, 1.5, 1.5)
				end
			end
		end

		-- ----- decks -----
		local function strips(z0, z1, y, col, mat, hole)
			local z = z0
			while z < z1 do
				local w = hw(z + 1) - 0.5
				if w > 0.3 then
					if hole and z + 1 > hole.z0 and z + 1 < hole.z1 then
						slab(-w, hole.x0, z, z + 2, y - 1.5, 1.5, col, mat, "Deck")
						slab(hole.x1, w, z, z + 2, y - 1.5, 1.5, col, mat, "Deck")
					else
						slab(-w, w, z, z + 2, y - 1.5, 1.5, col, mat, "Deck")
					end
				end
				z += 2
			end
		end
		strips(56, 124, Y0, TEAK, PLANK, { x0 = -4.7, x1 = 4.7, z0 = 60, z1 = 78 })
		strips(-152, -70, Y0, TEAK, PLANK, nil)
		-- podium (centre deck) y = 4; the engine slot (x 8..14, z -18..8) is left open
		slab(-34, 8, -22, 24, Y1 - 1.5, 1.5, TEAK, PLANK, "Deck")
		slab(14, 34, -22, 24, Y1 - 1.5, 1.5, TEAK, PLANK, "Deck")
		slab(8, 14, 8, 24, Y1 - 1.5, 1.5, TEAK, PLANK, "Deck")
		slab(8, 14, -22, -18, Y1 - 1.5, 1.5, TEAK, PLANK, "Deck")
		slab(-34, 34, -70, -66, Y1 - 1.5, 1.5, TEAK, PLANK, "Deck") -- bow ledge
		slab(-26, -10, 24, 56, Y1 - 1.5, 1.5, SAND, SM, "HouseFloor")
		slab(10, 26, 24, 56, Y1 - 1.5, 1.5, SAND, SM, "HouseFloor")
		slab(-10, 10, 24, 36, Y1 - 1.5, 1.5, TEAK, PLANK, "Deck")
		slab(-26, 26, -66, -22, Y1 - 1.5, 1.5, SAND, SM, "HouseFloor")
		-- side lanes ("viewing decks"): grey non-slip plate with glass skylight panels
		for _, sx in ipairs({ -1, 1 }) do
			slab(math.min(sx * 26, sx * 34), math.max(sx * 26, sx * 34), 24, 56, Y1 - 1.5, 1.5, RGB(120, 124, 130), DIA, "Lane")
			slab(math.min(sx * 26, sx * 34), math.max(sx * 26, sx * 34), -66, -22, Y1 - 1.5, 1.5, RGB(120, 124, 130), DIA, "Lane")
			-- lane skylights
			for _, zc in ipairs({ 30, 46, -34, -52 }) do
				local sl = B(sx * 30, Y1, zc, 4.4, 0.12, 6.4, RGB(26, 34, 44), GLS, "Skylight")
				sl.Transparency, sl.CanCollide = 0.25, false
				deco(B(sx * 30, Y1, zc, 5.2, 0.1, 7.2, DARK, MET, "SkylightFrame"))
			end
		end
		-- lane ramps down to the stern deck and the bow deck
		for _, sx in ipairs({ -1, 1 }) do
			local r1 = ramp(f, Vector3.new(sx * 30, Y0, 66), Vector3.new(sx * 30, Y1, 56), 8, RGB(120, 124, 130), DIA)
			local r2 = ramp(f, Vector3.new(sx * 30, Y1, -70), Vector3.new(sx * 30, Y0, -80), 8, RGB(120, 124, 130), DIA)
			r1.Name, r2.Name = "LaneRamp", "LaneRamp"
		end
		-- podium front/back faces (4 high) where there are no stairs
		for _, w in ipairs({ { -70, -26, -12.5 }, { -70, 12.5, 26 } }) do wall("x", w[1], w[2], w[3], Y0, 4.0, HULL, SM, "PodiumWall", nil, nil, 0.8) end
		-- stairs: stern deck -> centre deck (wide, between the two aft blocks); bow deck -> bow ledge
		stair(Vector3.new(0, Y0, 58), Vector3.new(0, Y1, 36), 20, CREAM, DTEAK, PLANK)
		stair(Vector3.new(0, Y0, -84), Vector3.new(0, Y1, -70), 24, CREAM, DTEAK, PLANK)
		for _, sx in ipairs({ -1, 1 }) do
			wall("z", sx * 12.6, -84, -70, Y0, 5.0, HULL, SM, "StairWall", nil, nil, 1.0)
		end

		-- little shared props
		local function plant(x, y, z, s)
			s = s or 1
			B(x, y, z, 2 * s, 1.1 * s, 2 * s, HULL, SM, "Pot")
			local lf = RGB(58, 104, 52)
			ball(x, y + 2.0 * s, z, 2.6 * s, 2.4 * s, 2.6 * s, lf, Enum.Material.Grass, "Leaves").CanCollide = false
			ball(x + 0.5 * s, y + 2.9 * s, z - 0.3 * s, 1.8 * s, 1.8 * s, 1.8 * s, RGB(72, 124, 62), Enum.Material.Grass, "Leaves").CanCollide = false
		end
		local function lounger(x, z, rot)
			local cf = CFrame.new(x, 0, z) * CFrame.Angles(0, rot or 0, 0)
			place(f, cf * CFrame.new(0, 0.7, 0), 3.0, 1.4, 6.6, HULL, SM, "Lounger")
			deco(place(f, cf * CFrame.new(0, 1.5, 0.3), 2.8, 0.34, 6.0, TURQ, FAB, "Cushion"))
			deco(place(f, cf * CFrame.new(0, 2.4, 2.5) * CFrame.Angles(math.rad(-32), 0, 0), 2.8, 0.34, 2.8, TURQ, FAB, "Cushion"))
			deco(place(f, cf * CFrame.new(0, 1.9, -2.2), 1.6, 0.55, 1.0, TRIM, FAB, "Pillow"))
		end
		local function sofa(x, y, z, rot, w, d, col)
			local cf = CFrame.new(x, y, z) * CFrame.Angles(0, rot or 0, 0)
			place(f, cf * CFrame.new(0, 0.75, 0), w, 1.5, d, col or TRIM, FAB, "Sofa")
			place(f, cf * CFrame.new(0, 2.1, d / 2 - 0.45), w, 2.2, 0.9, col or TRIM, FAB, "SofaBack")
			for _, sx in ipairs({ -1, 1 }) do place(f, cf * CFrame.new(sx * (w / 2 - 0.4), 1.7, 0), 0.8, 1.2, d, col or TRIM, FAB, "SofaArm") end
		end
		local function crate(x, y, z, rot, s)
			s = s or 3.4
			place(f, CFrame.new(x, y + s / 2, z) * CFrame.Angles(0, math.rad(rot or 0), 0), s, s, s, RGB(176, 150, 104), PLANK, "Crate")
		end

		-- =====================================================================================================
		-- ENGINE ROOM (y -9): cream machinery hall under the centre deck, service tunnel to the stern, grand ramp to the centre deck
		-- =====================================================================================================
		do
			local EW, EF = RGB(206, 200, 186), RGB(96, 100, 106)
			slab(-16, 16, -22, 24, YE - 1.5, 1.5, EF, CONC, "EngFloor")
			slab(-5.4, 5.4, 24, 62, YE - 1.5, 1.5, EF, CONC, "EngFloor")
			slab(-16, 8, -22, 24, -1.5, 1, EW, SM, "EngCeil")
			slab(14, 16, -22, 24, -1.5, 1, EW, SM, "EngCeil")
			slab(8, 14, 8, 24, -1.5, 1, EW, SM, "EngCeil")
			slab(-5.4, 5.4, 24, 60, -1.5, 1, EW, SM, "EngCeil")
			wall("z", -16, -22, 24, YE, 7.5, EW, SM, "EngWall")
			wall("z", 16, -22, 24, YE, 7.5, EW, SM, "EngWall")
			wall("x", -22, -16, 16, YE, 7.5, EW, SM, "EngWall")
			wall("x", 24, -16, 16, YE, 7.5, EW, SM, "EngWall", { { -5, 5, 7.5 } })
			for _, sx in ipairs({ -1, 1 }) do
				wall("z", sx * 5, 24, 60, YE, 7.5, EW, SM, "EngWall", nil, nil, 1.0)
				wall("z", sx * 5, 60, 78, YE, 9.9, EW, SM, "EngWall", nil, nil, 1.0)
			end
			-- the slot where the grand ramp climbs through to the centre deck
			wall("z", 7.6, -18, 8, -1.5, 6.9, HULL, SM, "Coaming", nil, nil, 0.8)
			wall("z", 14.4, -18, 8, -1.5, 6.9, HULL, SM, "Coaming", nil, nil, 0.8)
			stair(Vector3.new(11, Y1, -18), Vector3.new(11, YE, 8), 6, STEEL, RGB(120, 124, 130), DIA)
			stair(Vector3.new(0, YE, 60), Vector3.new(0, Y0, 78), 9, STEEL, RGB(120, 124, 130), DIA)
			-- machinery
			for _, z in ipairs({ -14, 1, 16 }) do
				B(-9.5, YE, z, 8, 1.4, 11.6, STEEL, MET, "EngineBase")
				B(-9.5, YE + 1.4, z, 6.2, 4.4, 9.6, RGB(224, 222, 214), SM, "Engine")
				tube(Vector3.new(-9.5, YE + 4.2, z - 5), Vector3.new(-9.5, YE + 4.2, z + 5), 3.6, RGB(214, 214, 208), SM, "Cylinder")
				tube(Vector3.new(-12.5, YE + 3.0, z - 4), Vector3.new(-12.5, YE + 3.0, z + 4), 1.4, RGB(176, 106, 68), MET, "Pipe")
				ghost(B(-6.2, YE + 3.2, z, 0.3, 0.5, 5, ORANGE, Enum.Material.Neon, "EngLight"))
			end
			for _, z in ipairs({ 13, 21 }) do
				B(11, YE, z, 6, 4.6, 6.4, RGB(62, 78, 96), MET, "Generator")
				ghost(B(8.0, YE + 3.4, z, 0.2, 0.5, 4.4, RGB(255, 170, 50), Enum.Material.Neon, "GenLight"))
				tube(Vector3.new(11, YE + 4.6, z - 3), Vector3.new(11, YE + 6.6, z - 3), 1, STEEL, MET, "Pipe")
			end
			-- control console with three monitors
			B(-1, YE, -20.6, 12, 2.4, 2.4, DARK, SM, "Console")
			for i = -1, 1 do
				local sc = ghost(B(-1 + i * 3.6, YE + 3.0, -21.2, 3.2, 2.0, 0.2, RGB(70, 220, 240), Enum.Material.Neon, "Screen"))
				sc.CFrame = sc.CFrame * CFrame.Angles(math.rad(-8), 0, 0)
			end
			-- overhead pipes
			for _, x in ipairs({ -13.5, -2.5, 2.5, 13.5 }) do tube(Vector3.new(x, -3.4, -21), Vector3.new(x, -3.4, 23), 0.8, RGB(176, 106, 68), MET, "Pipe") end
			for z = -16, 20, 9 do ghost(B(-13.5, -4.3, z, 0.7, 1.8, 0.7, RGB(200, 50, 44), MET, "Valve")) end
			-- hazard stripes + crates
			for _, x in ipairs({ -4, 4 }) do ghost(B(x, YE, -20, 0.4, 0.04, 42, RGB(236, 196, 40), SM, "Hazard")) end
			crate(5, YE, -17, 12, 3) crate(-2, YE, 20, 0, 2.8)
			-- lights: cool white in the hall, blue in the tunnel
			for _, p in ipairs({ { 0, -16 }, { 0, 0 }, { 0, 14 }, { 12, 18 } }) do lampc(p[1], -4, p[2], RGB(214, 226, 255), 40, 0.9) end
			for z = 32, 56, 12 do lampc(0, -4, z, RGB(120, 160, 255), 26, 0.9) end
			for _, sx in ipairs({ -1, 1 }) do led(sx * 4.1, -3.2, 42, 0.18, 0.2, 36, RGB(70, 140, 255)) end
			signboard(f, Vector3.new(0, -3.0, 23.3), Vector3.new(3.4, 1.1, 0.3), Enum.NormalId.Front, "EXIT", RGB(60, 255, 120))
			signboard(f, Vector3.new(0, -3.0, 24.7), Vector3.new(3.4, 1.1, 0.3), Enum.NormalId.Back, "EXIT", RGB(60, 255, 120))
		end

		-- =====================================================================================================
		-- STERN DECK (z 56..124, y 0): open teak plaza, curved bar, canopy, loungers, skylight, hatch to the engine room
		-- =====================================================================================================
		do
			-- curved bar facing the stairs
			arc(0, 104, 9, 205, 335, Y0, 3.4, 2.2, HULL, SM, "Bar", 2.6)
			arc(0, 104, 9.1, 205, 335, Y0 + 3.4, 0.4, 3.0, RGB(40, 42, 48), GRAN, "BarTop", 2.6)
			arc(0, 104, 9.0, 205, 335, Y0, 1.0, 2.5, DTEAK, PLANK, "BarBase", 2.6)
			for _, a in ipairs({ 218, 238, 258, 278, 298, 318 }) do
				local sx, sz = 12.4 * math.cos(math.rad(a)), 104 + 12.4 * math.sin(math.rad(a))
				cyl(sx, Y0, sz, 2.4, 0.5, CHROME, MET, "StoolPost")
				cyl(sx, Y0 + 2.4, sz, 0.5, 2.0, TURQ, FAB, "Stool")
			end
			for _, a in ipairs({ 230, 270, 310 }) do
				cyl(9.1 * math.cos(math.rad(a)), Y0 + 3.8, 104 + 9.1 * math.sin(math.rad(a)), 1.1, 0.5, RGB(40, 120, 60), GLS, "Bottle")
			end
			-- tall angled vent block at the back, with louvres
			B(0, Y0, 116, 12, 7.2, 6, HULL, SM, "VentBlock")
			for i = 0, 5 do ghost(B(0, Y0 + 1.2 + i * 0.95, 112.9, 9.6, 0.3, 0.3, RGB(60, 64, 72), SM, "Louvre")) end
			ghost(B(0, Y0 + 7.2, 116, 12.6, 0.5, 6.6, TRIM, SM, "VentCap"))
			-- sculptural canopy wing (tilted slab with timber soffit) on two pillars
			do
				local cf = CFrame.new(14, 11.4, 106) * CFrame.Angles(math.rad(-3), math.rad(6), math.rad(3))
				place(f, cf, 20, 0.9, 14, HULL, SM, "Canopy")
				deco(place(f, cf * CFrame.new(0, -0.55, 0), 19.4, 0.2, 13.4, DTEAK, PLANK, "CanopyUnder"))
				ghost(place(f, cf * CFrame.new(0, 0.5, 6.9), 20.2, 0.25, 0.4, BLUE, SM, "CanopyStripe"))
				cyl(22.2, Y0, 112.5, 11, 1.9, HULL, SM, "Pillar")
				cyl(5.8, Y0, 112.5, 11, 1.9, HULL, SM, "Pillar")
				tube(Vector3.new(22.2, 11, 112.5), Vector3.new(26, 14.2, 104), 1.1, HULL, SM, "Arch")
			end
			lounger(10, 106, 0) lounger(18, 106, 0) lounger(-17, 112, 0.2) lounger(-24, 104, 0.35)
			lounger(26.5, 72, math.pi / 2) lounger(-26.5, 96, -math.pi / 2)
			-- glass floor panels (skylight over the lower rooms) on the port side
			for ix = 0, 2 do
				for iz = 0, 3 do
					local x, z = -29 + ix * 5.2, 66.5 + iz * 6.2
					local g2 = B(x, Y0, z, 4.8, 0.1, 5.8, RGB(24, 32, 44), GLS, "GlassFloor")
					g2.Transparency, g2.CanCollide = 0.2, false
					deco(B(x, Y0, z, 5.4, 0.08, 6.4, DARK, MET, "GlassFrame"))
				end
			end
			-- cover: big white lockers, planters, crates
			B(-25, Y0, 78, 6, 4.2, 9, HULL, SM, "Locker") ghost(B(-25, Y0 + 4.2, 78, 6.4, 0.3, 9.4, CHROME, MET, "LockerRail")).CanCollide = false
			B(25, Y0, 80, 6, 4.2, 9, HULL, SM, "Locker")
			B(-8, Y0, 70, 5, 3.4, 6, HULL, SM, "Locker") B(9, Y0, 90, 3.6, 3, 3.6, HULL, SM, "Locker")
			plant(12, Y0, 64, 1.2) plant(-12, Y0, 64, 1.2) plant(-17, Y0, 90, 1) plant(19, Y0, 94, 1)
			crate(-15, Y0, 99, 18) crate(-19.5, Y0, 97.5, 40, 2.6)
			-- hex portholes in the bulwark, string of globe lights over the plaza
			for _, sx in ipairs({ -1, 1 }) do
				for z = 66, 112, 9 do
					local ph = ghost(B(sx * (hw(z) * 0.985 - 0.66), Y0 + 2.2, z, 0.2, 1.8, 1.8, RGB(40, 46, 56), SM, "Hex"))
					ph.Shape = Enum.PartType.Cylinder
					ph.Size = Vector3.new(0.2, 1.8, 1.8)
				end
			end
			for _, p in ipairs({ { -20, 62 }, { 20, 62 }, { -14, 84 }, { 14, 84 }, { 0, 76 }, { -10, 110 }, { 24, 98 } }) do
				lampc(p[1], 9, p[2], RGB(255, 242, 220), 48, 0.5)
			end
		end

		-- =====================================================================================================
		-- BOW DECK (z -70..-152, y 0): helipad + helicopter, two tenders, bow lounges, anchor gear
		-- =====================================================================================================
		do
			-- helipad
			cyl(0, Y0, -128, 0.35, 23.6, RGB(214, 178, 44), SM, "PadRing")
			cyl(0, Y0 + 0.35, -128, 0.3, 22, RGB(66, 70, 78), CONC, "Helipad")
			for _, b in ipairs({ { -2.8, 0, 1.3, 9 }, { 2.8, 0, 1.3, 9 }, { 0, 0, 6.4, 1.3 } }) do ghost(B(b[1], Y0 + 0.65, -128 + b[2], b[3], 0.05, b[4], TRIM, SM, "PadH")) end
			-- helicopter (nose toward the bow)
			do
				local hy, hz = Y0 + 0.65, -128
				ball(0, hy + 3.8, hz, 4.4, 4.6, 10, TRIM, SM, "HeliBody").CanCollide = true
				ball(0, hy + 4.2, hz - 3.4, 3.6, 3.2, 4, GLASSC, GLS, "HeliGlass").Transparency = 0.35
				ghost(B(0, hy + 3.4, hz, 4.5, 0.5, 9.2, BLUE, SM, "HeliStripe"))
				deco(B(0, hy + 3.8, hz + 8.5, 0.9, 0.9, 8.5, TRIM, SM, "HeliTail"))
				deco(B(0, hy + 4.2, hz + 12.4, 0.3, 3.2, 1.6, TRIM, SM, "HeliFin"))
				cyl(0, hy + 5.6, hz - 0.5, 1.2, 0.5, DARK, MET, "HeliMast")
				ghost(B(0, hy + 6.8, hz - 0.5, 17, 0.12, 0.9, DARK, SM, "Rotor"))
				ghost(B(0, hy + 6.8, hz - 0.5, 0.9, 0.12, 17, DARK, SM, "Rotor"))
				for _, sx in ipairs({ -1, 1 }) do
					tube(Vector3.new(sx * 2.4, hy + 0.6, hz - 4), Vector3.new(sx * 2.4, hy + 0.6, hz + 4), 0.5, DARK, MET, "Skid")
					deco(B(sx * 2.4, hy + 0.6, hz - 2, 0.3, 1.8, 0.3, DARK, MET, "SkidLeg"))
					deco(B(sx * 2.4, hy + 0.6, hz + 2.4, 0.3, 1.8, 0.3, DARK, MET, "SkidLeg"))
				end
			end
			-- tenders on cradles
			local function tender(x, z)
				local cf = CFrame.new(x, 0, z)
				place(f, cf * CFrame.new(0, 0.5, 0), 3, 1, 11, STEEL, MET, "Cradle")
				place(f, cf * CFrame.new(0, 2.1, 1), 5.6, 2.6, 10, TRIM, SM, "Tender")
				for _, sx in ipairs({ -1, 1 }) do
					place(f, cf * CFrame.new(sx * 1.2, 2.1, -5.7) * CFrame.Angles(0, sx * math.rad(-25), 0), 3.2, 2.6, 3.2, TRIM, SM, "TenderBow")
				end
				ghost(place(f, cf * CFrame.new(0, 1.5, 1), 5.8, 0.5, 10.2, NAVY, SM, "TenderStripe"))
				place(f, cf * CFrame.new(0, 4.2, -0.4) * CFrame.Angles(math.rad(-18), 0, 0), 4.2, 2.0, 0.3, GLASSC, GLS, "TenderGlass").Transparency = 0.45
				place(f, cf * CFrame.new(0, 3.9, 2.4), 4, 1.2, 3, DARK, LEA, "TenderSeat")
				place(f, cf * CFrame.new(0, 3.2, 6.4), 1.6, 3, 1.2, DARK, MET, "Outboard")
			end
			tender(-13.5, -100) tender(13.5, -100)
			-- white deck box with life ring, hatch
			B(0, Y0, -96, 5.4, 3.6, 4.4, HULL, SM, "DeckBox")
			cyl(0, Y0 + 1.6, -98.25, 0.3, 2.6, RGB(236, 112, 40), SM, "LifeRing").CFrame = CFrame.new(0, Y0 + 1.8, -98.3) * CFrame.Angles(0, math.pi / 2, math.pi / 2)
			ghost(B(-9, Y0, -84.5, 6, 0.1, 6, RGB(60, 64, 70), DIA, "Hatch"))
			-- bow lounge cabins with sliding doors
			for _, sx in ipairs({ -1, 1 }) do
				local x0, x1 = sx * 20.5, sx * 28
				local xa, xb = math.min(x0, x1), math.max(x0, x1)
				wall("z", x0, -108, -90, Y0, 8.4, HULL, SM, "BowCabin", { { -102, -95 } }, { { -107, -104 }, { -93, -91 } })
				wall("z", x1, -108, -90, Y0, 8.4, HULL, SM, "BowCabin", nil, { { -106, -98 }, { -96, -92 } })
				wall("x", -108, xa, xb, Y0, 8.4, HULL, SM, "BowCabin")
				wall("x", -90, xa, xb, Y0, 8.4, HULL, SM, "BowCabin", nil, { { xa + 1.5, xb - 1.5 } })
				slab(xa - 0.5, xb + 0.5, -108.5, -89.5, Y0 + 8.4, 0.9, HULL, SM, "BowCabinRoof")
				ghost(B(sx * 24.2, Y0 + 9.3, -99, 8.6, 0.25, 19.4, BLUE, SM, "RoofStripe"))
				pins("z", x0, -108, -90, Y0 + 8.0, 0.1) pins("z", x1, -108, -90, Y0 + 8.0, 0.1)
				B(sx * 26.2, Y0, -99, 3, 1.5, 12, TRIM, FAB, "Banquette")
				place(f, CFrame.new(sx * 24, 1.2, -99), 2.4, 1.2, 2.6, DTEAK, PLANK, "CoffeeTable")
				lampc(sx * 24.2, 7, -99, WARM, 28, 0.7)
			end
			-- anchors, bollards, flag at the tip
			for _, sx in ipairs({ -1, 1 }) do
				B(sx * 8, Y0, -139, 3, 2.2, 3, STEEL, MET, "Winch")
				ball(sx * 8, Y0 + 2.6, -139, 2.4, 2.4, 2.4, CHROME, MET, "WinchDrum")
			end
			cyl(0, Y0, -146, 15, 0.5, CHROME, MET, "Flagpole")
			deco(B(0.1, Y0 + 11, -147.2, 0.1, 2.6, 4, RGB(214, 40, 40), FAB, "Flag"))
			for _, p in ipairs({ { -17, -76 }, { 17, -76 }, { -22, -118 }, { 22, -118 }, { -12, -134 }, { 12, -134 } }) do
				local bo = B(p[1], Y0, p[2], 1.2, 1.5, 1.2, STEEL, MET, "Bollard")
				bo.Shape = Enum.PartType.Cylinder
				bo.CFrame = CFrame.new(p[1], Y0 + 0.75, p[2]) * CFrame.Angles(0, 0, math.pi / 2)
			end
			-- loungers, parasol, crates for cover
			lounger(-19, -122, 0.6) lounger(19, -122, -0.6) lounger(-20, -78.5, 0) lounger(20, -78.5, 0)
			cyl(-24, Y0, -76, 9, 0.35, CHROME, MET, "ParasolPole")
			cyl(-24, Y0 + 9, -76, 0.3, 9, RGB(36, 74, 140), FAB, "Parasol").CanCollide = false
			crate(-22, Y0, -130, 10) crate(22, Y0, -132, -20) crate(-8, Y0, -112, 0, 2.6) crate(8, Y0, -86, 30, 2.6)
			plant(-6, Y0, -78, 1) plant(6, Y0, -78, 1)
			for _, p in ipairs({ { -16, -88 }, { 16, -88 }, { 0, -112 }, { 0, -138 }, { -14, -124 }, { 14, -124 } }) do lampc(p[1], 9, p[2], RGB(255, 242, 220), 46, 0.5) end
		end

		local function xr(sg, a, b) local p, q = sg * a, sg * b if p > q then p, q = q, p end return p, q end
		local function ledRing(x0, x1, z0, z1, y, col) -- LED cove around a ceiling
			led((x0 + x1) / 2, y, z0, x1 - x0, 0.22, 0.22, col) led((x0 + x1) / 2, y, z1, x1 - x0, 0.22, 0.22, col)
			led(x0, y, (z0 + z1) / 2, 0.22, 0.22, z1 - z0, col) led(x1, y, (z0 + z1) / 2, 0.22, 0.22, z1 - z0, col)
		end
		local function tv(x, y, z, w, h, axis) -- flat screen on a wall
			if axis == "x" then B(x, y, z, 0.25, h, w, DARK, SM, "TV") else B(x, y, z, w, h, 0.25, DARK, SM, "TV") end
		end

		-- =====================================================================================================
		-- MID HOUSE (x -26..26, z -66..-22): L1 pantry + kitchen (port) and a grand dining hall; L2 bridge (bow end) + office + open balcony;
		-- roof deck with radar. Dining stairs rise along the starboard wall to the balcony; a roof stair climbs from the bridge.
		-- =====================================================================================================
		do
			local X0, X1, Z0, Z1 = -26, 26, -66, -22
			wall("x", Z0, X0, X1, Y1, 9, HULL, SM, "MidWall", { { -22, -16 }, { -2, 4 } }, nil, 0.9)
			wall("x", Z1, X0, X1, Y1, 9, HULL, SM, "MidWall", { { -22, -16 }, { 0, 8 } }, { { -13, -5 }, { 11, 22 } }, 0.9)
			wall("z", X1, Z0, Z1, Y1, 9, HULL, SM, "MidWall", nil, { { -64, -57 }, { -53, -46 }, { -42, -35 }, { -31, -24 } }, 0.9)
			wall("z", X0, Z0, Z1, Y1, 9, HULL, SM, "MidWall", nil, { { -64, -57 }, { -47, -40 }, { -34, -25 } }, 0.9)
			wall("z", -12, Z0, Z1, Y1, 9, CREAM, SM, "Partition", { { -42, -31 }, { -63, -57 } }, nil, 0.8)
			wall("x", -50, X0, -12, Y1, 9, CREAM, SM, "Partition", { { -21, -16 } }, nil, 0.8)
			-- L2 floor (stair slot on the starboard side)
			slab(-26, 21, -66, -22, Y2 - 1, 1, CREAM, SM, "UpperFloor")
			slab(21, 26, -66, -42, Y2 - 1, 1, CREAM, SM, "UpperFloor")
			slab(21, 26, -24, -22, Y2 - 1, 1, CREAM, SM, "UpperFloor")
			stair(Vector3.new(23.5, Y1, -24), Vector3.new(23.5, Y2, -42), 5, CREAM, DTEAK, PLANK)
			glass(20.9, Y1, -33, 0.25, 3.4, 18, 0.6)
			-- L1 floors + finishes
			slab(-12, 26, -66, -22, Y1, 0.12, RGB(148, 98, 62), PLANK, "DiningFloor")
			slab(-26, -12, -50, -22, Y1, 0.12, RGB(206, 200, 190), MARB, "KitchenFloor")
			slab(-26, -12, -66, -50, Y1, 0.12, RGB(118, 80, 52), PLANK, "PantryFloor")
			pins("x", Z0 - 0.0, X0, X1, Y1 + 8.2) pins("x", Z1, X0, X1, Y1 + 8.2) pins("z", X1, Z0, Z1, Y1 + 8.2) pins("z", X0, Z0, Z1, Y1 + 8.2)

			-- DINING: marble table, chairs, mural wall, spiral ceiling light, purple cove lights
			do
				slab(4.5, 11.5, -56, -33, Y1 + 2.6, 0.6, RGB(214, 208, 196), MARB, "DiningTable")
				for _, z in ipairs({ -53, -36 }) do B(8, Y1, z, 2.4, 2.6, 1.2, DARK, SM, "TableLeg") end
				for i = 0, 5 do
					for _, sx in ipairs({ -1, 1 }) do
						local cx, cz = 8 + sx * 5.2, -54 + i * 3.8
						place(f, CFrame.new(cx, Y1 + 1.7, cz), 1.9, 0.5, 1.9, DARK, LEA, "Chair")
						place(f, CFrame.new(cx + sx * 0.8, Y1 + 3.0, cz), 0.4, 2.3, 1.9, DARK, LEA, "ChairBack")
						B(cx, Y1, cz, 0.4, 1.5, 0.4, DARK, MET, "ChairLeg")
					end
				end
				slab(-1, 17, -58, -30, Y1, 0.05, RGB(70, 56, 70), FAB, "Rug")
				-- mural wall (stylised mountains)
				B(8, Y1 + 2, -65.4, 22, 6.4, 0.2, RGB(86, 66, 112), SM, "Mural")
				for _, m in ipairs({ { 0, 5, 9, 6 }, { 8, 5.6, 11, 7.6 }, { 15, 5, 9, 5.8 } }) do ghost(ball(m[1], Y1 + 2 + m[2] * 0.6, -65.25, m[3], m[4], 0.2, RGB(150, 120, 176), SM, "MuralHill")) end
				for _, p in ipairs({ { 24, -62 }, { 24, -26 }, { -9, -25 }, { -9, -63 } }) do plant(p[1], Y1, p[2], 1.2) end
				-- ceiling: spiral light disc + cove LEDs
				cyl(8, Y1 + 7.2, -44, 0.7, 11, TRIM, SM, "CeilingDisc").CanCollide = false
				ghost(cyl(8, Y1 + 7.0, -44, 0.2, 8.6, PURPLE, Enum.Material.Neon, "CeilingRing"))
				ledRing(-11, 25, -65.4, -22.6, Y1 + 8.4, PURPLE)
				for _, p in ipairs({ { 8, -58 }, { 8, -44 }, { 8, -30 }, { -4, -44 }, { 18, -44 } }) do lampc(p[1], Y1 + 7.6, p[2], RGB(210, 170, 255), 34, 0.55) end
				for _, p in ipairs({ { 8, -44 }, { 0, -30 } }) do lampc(p[1], Y1 + 6, p[2], RGB(255, 236, 205), 36, 0.5) end
			end
			-- KITCHEN + PANTRY
			do
				B(-24.8, Y1, -36, 1.8, 3.4, 26, WOODL, WOOD, "Cabinets")
				ghost(B(-24.6, Y1 + 3.4, -36, 2.4, 0.3, 26.4, RGB(40, 42, 48), GRAN, "Counter")).CanCollide = true
				B(-24.6, Y1 + 6.4, -36, 1.4, 2.4, 22, WOODL, WOOD, "WallCabinet")
				B(-24.4, Y1, -24.6, 2.2, 6.8, 2.8, CHROME, MET, "Fridge")
				B(-24.4, Y1 + 3.4, -41, 2.2, 0.3, 3.4, RGB(22, 22, 26), SM, "Hob")
				B(-17.5, Y1, -36, 5, 3.4, 12, WOODL, WOOD, "Island")
				B(-17.5, Y1 + 3.4, -36, 5.6, 0.4, 12.6, RGB(236, 232, 222), MARB, "IslandTop")
				for _, z in ipairs({ -40, -36, -32 }) do cyl(-14.2, Y1, z, 2.6, 1.3, DARK, LEA, "BarStool") end
				for _, z in ipairs({ -64, -56 }) do
					B(-24.5, Y1, z, 1.8, 8, 3, WOODP, WOOD, "Shelf")
					B(-20, Y1, z - 0.5 * (z < -60 and -1 or 1), 4, 1.2, 1.6, RGB(160, 130, 90), FAB, "Sacks")
				end
				crate(-16, Y1, -62, 14, 2.6) crate(-19.5, Y1, -58, 0, 2.2)
				lampc(-19, Y1 + 7.2, -36, RGB(255, 244, 226), 34, 0.8) lampc(-19, Y1 + 7.2, -58, RGB(255, 236, 205), 28, 0.6)
				ledRing(-25.2, -12.8, -49.5, -22.8, Y1 + 8.4, WARM)
			end
			-- L2 shell
			wall("x", Z0, X0, X1, Y2, 9, HULL, SM, "MidWall2", nil, { { -24, -13 }, { -12, -1 }, { 0, 12 }, { 13, 24 } }, 0.9)
			wall("z", X1, Z0, -44, Y2, 9, HULL, SM, "MidWall2", nil, { { -64, -46 } }, 0.9)
			wall("z", X0, Z0, -44, Y2, 9, HULL, SM, "MidWall2", nil, { { -64, -46 } }, 0.9)
			wall("x", -44, X0, X1, Y2, 9, HULL, SM, "MidWall2", { { -16, -10 }, { 12, 18 } }, nil, 0.8)
			wall("x", Z1, X0, 6, Y2, 9, HULL, SM, "MidWall2", nil, { { -24, -10 }, { -8, 4 } }, 0.9)
			wall("z", X0, -44, Z1, Y2, 9, HULL, SM, "MidWall2", nil, { { -42, -24 } }, 0.9)
			wall("z", 6, -44, Z1, Y2, 9, HULL, SM, "MidWall2", { { -37, -30 } }, nil, 0.9)
			-- roof (hole for the roof stair)
			slab(-26, -24, -66, -44, Y3 - 1, 1, HULL, SM, "Roof") slab(-20, 26, -66, -44, Y3 - 1, 1, HULL, SM, "Roof")
			slab(-24, -20, -66, -64, Y3 - 1, 1, HULL, SM, "Roof") slab(-24, -20, -46, -44, Y3 - 1, 1, HULL, SM, "Roof")
			slab(-26, 6, -44, -22, Y3 - 1, 1, HULL, SM, "Roof")
			stair(Vector3.new(-22, Y2, -64), Vector3.new(-22, Y3, -46), 4, CREAM, DTEAK, PLANK)
			-- L2 floors
			slab(-26, 26, -66, -44, Y2, 0.12, RGB(112, 72, 44), PLANK, "BridgeFloor")
			slab(-26, 6, -44, -22, Y2, 0.12, RGB(142, 140, 152), FAB, "OfficeFloor")
			slab(6, 21, -44, -22, Y2, 0.12, TEAK, PLANK, "BalconyDeck") slab(21, 26, -44, -42, Y2, 0.12, TEAK, PLANK, "BalconyDeck") slab(21, 26, -24, -22, Y2, 0.12, TEAK, PLANK, "BalconyDeck")
			pins("x", Z0, X0, X1, Y2 + 8.2) pins("z", X1, Z0, -44, Y2 + 8.2) pins("z", X0, Z0, Z1, Y2 + 8.2) pins("x", Z1, X0, 6, Y2 + 8.2)
			-- balcony balustrade + soffit style fascia
			glass(26, Y2, -33, 0.25, 3.4, 22, 0.6) glass(16, Y2, -22, 20, 3.4, 0.25, 0.6)
			deco(B(26, Y2 + 3.4, -33, 0.5, 0.35, 22.4, CHROME, MET, "Rail")) deco(B(16, Y2 + 3.4, -22, 20.4, 0.35, 0.5, CHROME, MET, "Rail"))
			ghost(B(16, Y2 - 1.6, -21.7, 20, 0.5, 0.5, BLUE, SM, "FasciaStripe"))
			sofa(15, Y2, -39, 0, 9, 3.2, TRIM) sofa(10, Y2, -26, math.pi, 6, 3, TRIM)
			B(14, Y2, -30.5, 4, 1.4, 4, DTEAK, PLANK, "BalconyTable")
			-- BRIDGE: curved helm, big chairs, curtains, purple cove
			do
				arc(0, -42, 13, 232, 308, Y2, 3.2, 2.4, WOODP, WOOD, "Helm", 3)
				arc(0, -42, 13, 232, 308, Y2 + 3.2, 0.3, 3.0, RGB(40, 44, 52), GRAN, "HelmTop", 3)
				for i = -2, 2 do
					local sc = ghost(B(i * 4.2, Y2 + 3.6, -54.6 + math.abs(i) * 0.5, 3.4, 2, 0.2, RGB(60, 210, 235), Enum.Material.Neon, "Screen"))
					sc.CFrame = sc.CFrame * CFrame.Angles(math.rad(-22), math.rad(i * -12), 0)
				end
				for _, x in ipairs({ -5, 5 }) do
					cyl(x, Y2, -50.5, 2.2, 1.0, CHROME, MET, "ChairPost")
					place(f, CFrame.new(x, Y2 + 2.6, -50.5), 2.4, 0.6, 2.4, RGB(70, 50, 40), LEA, "CaptainChair")
					place(f, CFrame.new(x, Y2 + 4.2, -49.4), 2.4, 2.8, 0.6, RGB(70, 50, 40), LEA, "CaptainChairBack")
				end
				for _, x in ipairs({ -24, 24 }) do B(x, Y2, -60, 1.0, 8.2, 4, RGB(90, 58, 56), FAB, "Curtain").CanCollide = false end
				sofa(-15, Y2, -62, math.rad(180), 8, 3, RGB(86, 64, 54))
				ledRing(-25, 25, -65.4, -44.6, Y2 + 8.4, PURPLE)
				lampc(0, Y2 + 7.4, -56, RGB(214, 180, 255), 44, 0.6) lampc(-14, Y2 + 7.4, -52, RGB(255, 236, 205), 32, 0.4) lampc(14, Y2 + 7.4, -52, RGB(255, 236, 205), 32, 0.4)
			end
			-- OFFICE: rounded island desk, shelves + TVs, big wall screen
			do
				B(-12, Y2, -33, 12, 3.2, 5, TRIM, SM, "Island")
				for _, sx in ipairs({ -1, 1 }) do cyl(-12 + sx * 6, Y2, -33, 3.2, 5, TRIM, SM, "IslandEnd") end
				B(-12, Y2 + 3.2, -33, 12.4, 0.3, 5.4, RGB(176, 128, 82), WOOD, "IslandTop")
				for i = 0, 3 do B(-24.8, Y2 + 2 + i * 1.8, -33, 0.8, 0.25, 12, WOODL, WOOD, "ShelfBoard") end
				tv(-24.6, Y2 + 3.5, -37, 4, 2.4, "x") tv(-24.6, Y2 + 3.5, -29, 4, 2.4, "x")
				B(-10, Y2 + 1.8, -43.5, 16, 5.6, 0.2, RGB(30, 150, 160), Enum.Material.Neon, "WallScreen").Transparency = 0.25
				sofa(-4, Y2, -26, math.pi, 7, 3, RGB(96, 90, 130))
				plant(-23, Y2, -25, 1.1) plant(-23, Y2, -41, 1.1)
				ledRing(-25, 5.4, -43.4, -22.6, Y2 + 8.4, PURPLE)
				lampc(-10, Y2 + 7.4, -33, RGB(190, 160, 255), 38, 0.7) lampc(-18, Y2 + 7.4, -26, RGB(255, 236, 205), 30, 0.4)
			end
			-- roof deck: parapet, radar mast, domes, vents, solar panels
			do
				for _, w in ipairs({ { "x", -66, -26, 26 }, { "z", 26, -66, -44 }, { "z", -26, -66, -22 }, { "x", -22, -26, 6 }, { "z", 6, -44, -22 }, { "x", -44, 6, 26 } }) do
					wall(w[1], w[2], w[3], w[4], Y3, 1.5, HULL, SM, "Parapet", nil, nil, 0.7)
				end
				cyl(0, Y3, -58, 11, 0.8, CHROME, MET, "Mast")
				B(0, Y3 + 8.5, -58, 7, 0.3, 0.8, CHROME, MET, "Radar") B(0, Y3 + 9.4, -58, 0.8, 0.3, 4, CHROME, MET, "Radar")
				for _, x in ipairs({ -12, 12 }) do ball(x, Y3 + 2.4, -54, 4.4, 4.4, 4.4, TRIM, SM, "Dome") end
				B(-16, Y3, -34, 6, 3, 8, HULL, SM, "RoofVent") B(14, Y3, -60, 5, 2.4, 5, HULL, SM, "RoofVent")
				for _, p in ipairs({ { -6, -34 }, { 2, -34 } }) do ghost(B(p[1], Y3, p[2], 6, 0.2, 10, RGB(26, 44, 86), GLS, "Solar")) end
				lampc(0, Y3 + 10, -44, RGB(255, 246, 232), 70, 0.35)
			end
			signboard(f, Vector3.new(0, Y3 + 0.75, -66.45), Vector3.new(14, 1.2, 0.3), Enum.NormalId.Front, "AURORA", RGB(240, 232, 214))
		end

		-- =====================================================================================================
		-- AFT BLOCKS (x -26..-10 and 10..26, z 24..56) either side of the wide stern stairs:
		-- port: laundry + gym (L1) / dressing room (L2); starboard: living room + TV room (L1) / bedroom + balcony (L2).
		-- An enclosed link corridor bridges the stairs on L2.
		-- =====================================================================================================
		local function aft(sg)
			local xo, xi = sg * 26, sg * 10
			local a, b = xr(sg, 16, 22)
			local fa, fb = xr(sg, 14, 20)
			-- L1 shell
			wall("z", xo, 24, 56, Y1, 9, HULL, SM, "AftWall", nil, { { 27, 35 }, { 47, 54 } }, 0.9)
			wall("z", xi, 24, 56, Y1, 9, HULL, SM, "AftWall", { { 26, 32 } }, { { 38, 43 }, { 47, 53 } }, 0.9)
			wall("x", 24, math.min(xo, xi), math.max(xo, xi), Y1, 9, HULL, SM, "AftWall", { { fa, fb } }, nil, 0.9)
			wall("x", 56, math.min(xo, xi), math.max(xo, xi), Y1, 9, HULL, SM, "AftWall", { { a, b } }, nil, 0.9)
			wall("x", 44, math.min(xo, xi), math.max(xo, xi), Y1, 9, CREAM, SM, "Partition", { { xr(sg, 14, 18) } }, nil, 0.8)
			wall("x", 56, math.min(xo, xi), math.max(xo, xi), Y0, 4, HULL, SM, "PodiumWall", nil, nil, 0.9)
			pins("z", xo, 24, 56, Y1 + 8.2) pins("x", 56, math.min(xo, xi), math.max(xo, xi), Y1 + 8.2) pins("x", 24, math.min(xo, xi), math.max(xo, xi), Y1 + 8.2)
			-- L2 floor with the stair slot, stair, roof
			local c0, c1 = xr(sg, 10, 21)
			local d0, d1 = xr(sg, 21, 26)
			slab(c0, c1, 24, 56, Y2 - 1, 1, CREAM, SM, "UpperFloor")
			slab(d0, d1, 24, 26, Y2 - 1, 1, CREAM, SM, "UpperFloor")
			slab(d0, d1, 42, 56, Y2 - 1, 1, CREAM, SM, "UpperFloor")
			stair(Vector3.new(sg * 23.5, Y1, 42), Vector3.new(sg * 23.5, Y2, 26), 4.8, CREAM, DTEAK, PLANK)
			-- L2 shell
			wall("z", xo, 24, 56, Y2, 9, HULL, SM, "AftWall2", nil, { { 27, 36 }, { 40, 54 } }, 0.9)
			wall("z", xi, 24, 56, Y2, 9, HULL, SM, "AftWall2", { { 29, 35 } }, { { 40, 54 } }, 0.9)
			wall("x", 24, math.min(xo, xi), math.max(xo, xi), Y2, 9, HULL, SM, "AftWall2", (sg > 0) and { { fa, fb } } or nil, (sg < 0) and { { -25, -12 } } or nil, 0.9)
			wall("x", 56, math.min(xo, xi), math.max(xo, xi), Y2, 9, HULL, SM, "AftWall2", nil, { { xr(sg, 12, 24) } }, 0.9)
			pins("z", xo, 24, 56, Y2 + 8.2) pins("x", 56, math.min(xo, xi), math.max(xo, xi), Y2 + 8.2)
			slab(math.min(xo, xi) - 0.6, math.max(xo, xi) + 0.6, 23.4, 56.6, Y3 - 1, 1, HULL, SM, "Roof")
			ghost(B(sg * 18, Y3, 40, 14, 0.18, 20, RGB(26, 44, 86), GLS, "Solar"))
			ghost(B(sg * 18, Y3 + 0.9, 56.6, 16.4, 0.3, 0.6, BLUE, SM, "RoofStripe"))
			-- curved corner towers on the stern corners
			cyl(sg * 25.4, Y0, 55.4, 23, 2.6, HULL, SM, "CornerTower")
			cyl(sg * 10.6, Y0, 55.4, 23, 2.6, HULL, SM, "CornerTower")
			return a
		end
		aft(-1) aft(1)

		-- port block interiors: laundry (z 44..56), gym (z 24..44), dressing room on L2
		do
			slab(-26, -10, 44, 56, Y1, 0.12, RGB(126, 84, 54), PLANK, "LaundryFloor")
			slab(-26, -10, 24, 44, Y1, 0.12, RGB(110, 114, 122), SLATE, "GymFloor")
			for _, z in ipairs({ 46.5, 51 }) do
				for k = 0, 1 do
					B(-24.7, Y1 + k * 3.2, z, 1.9, 3.1, 3.4, TRIM, MET, "Washer")
					local dr = ghost(B(-23.7, Y1 + 0.8 + k * 3.2, z, 0.3, 1.7, 1.7, RGB(60, 70, 80), GLS, "WasherDoor"))
					dr.Shape = Enum.PartType.Cylinder
					dr.Size = Vector3.new(0.3, 1.7, 1.7)
				end
				B(-24.7, Y1 + 6.6, z, 1.9, 2.6, 3.4, WOODL, WOOD, "LaundryCabinet")
			end
			tube(Vector3.new(-14, Y1 + 6.6, 46), Vector3.new(-14, Y1 + 6.6, 55), 0.3, CHROME, MET, "ClothesRail")
			for i = 0, 3 do ghost(B(-14, Y1 + 3.6, 46.8 + i * 2.1, 0.5, 3, 1.3, ({ RGB(220, 214, 200), RGB(60, 84, 110), RGB(200, 196, 188), RGB(130, 40, 40) })[i + 1], FAB, "Garment")) end
			B(-12.4, Y1, 55, 5, 3.2, 1.6, WOODL, WOOD, "Counter") B(-12.4, Y1 + 3.2, 55, 5.4, 0.3, 2, RGB(210, 206, 198), GRAN, "CounterTop")
			place(f, CFrame.new(-20.5, Y1 + 2.8, 48) * CFrame.Angles(0, math.rad(20), math.rad(80)), 5.6, 0.3, 1.6, RGB(90, 120, 170), FAB, "IroningBoard")
			B(-21, Y1, 53.2, 2.4, 1.6, 2, RGB(120, 150, 130), SM, "Basket")
			ledRing(-25.2, -10.6, 44.6, 55.4, Y1 + 8.4, ORANGE)
			lampc(-18, Y1 + 7.2, 50, RGB(255, 190, 120), 30, 0.8)
			-- gym
			for _, z in ipairs({ 29, 34.5 }) do
				B(-14, Y1, z, 2.6, 1.0, 6.4, DARK, SM, "TreadmillBase")
				B(-14, Y1 + 1.0, z - 0.5, 2.0, 0.3, 4.8, RGB(40, 42, 46), SM, "TreadmillBelt")
				B(-14, Y1 + 1.0, z - 3.1, 2.4, 3.2, 0.6, DARK, SM, "TreadmillConsole")
			end
			B(-13, Y1, 40, 2, 1.6, 5, DARK, LEA, "Bench") B(-20, Y1, 41, 6, 6, 0.6, STEEL, MET, "Rack")
			for _, p in ipairs({ { -19, 27, 1.0, 0.1 }, { -16.8, 31 } }) do
				B(p[1], Y1, p[2], 4.4, 0.2, 2, RGB(60, 110, 150), FAB, "Mat")
			end
			for _, p in ipairs({ { -22, 32 }, { -23, 35.5 }, { -22, 38 } }) do ball(p[1], Y1 + 1.1, p[2], 2.2, 2.2, 2.2, (p[2] > 34) and TURQ or RGB(200, 90, 60), SM, "GymBall") end
			sofa(-11.8, Y1, 42, -math.pi / 2, 3, 2.8, RGB(94, 80, 70))
			glass(-10.4, Y1 + 0.6, 34, 0.15, 6, 8, 0.55).Reflectance = 0.25
			ledRing(-25.2, -10.6, 24.6, 43.4, Y1 + 8.4, ORANGE)
			lampc(-16, Y1 + 7.2, 33, RGB(255, 200, 140), 36, 0.8)
			-- L2 dressing room
			slab(-21, -10, 24, 56, Y2, 0.12, RGB(126, 84, 54), PLANK, "DressingFloor")
			slab(-26, -21, 24, 26, Y2, 0.12, RGB(126, 84, 54), PLANK, "DressingFloor")
			slab(-26, -21, 43.5, 56, Y2, 0.12, RGB(126, 84, 54), PLANK, "DressingFloor")
			glass(-21, Y2, 34.75, 0.15, 3.4, 17.5, 0.5)
			B(-24.8, Y2, 36, 1.8, 7.4, 8, WOODL, WOOD, "Wardrobe") B(-24.8, Y2, 48, 1.8, 7.4, 8, WOODL, WOOD, "Wardrobe")
			B(-15, Y2, 40, 5, 2.6, 8, TRIM, FAB, "Island") B(-15, Y2 + 2.6, 40, 5.4, 0.3, 8.4, RGB(176, 128, 82), WOOD, "IslandTop")
			glass(-11, Y2 + 0.6, 50, 0.2, 6.4, 5, 0.4).Reflectance = 0.3
			plant(-12, Y2, 54, 1)
			ledRing(-25.2, -10.6, 24.6, 55.4, Y2 + 8.4, ORANGE)
			lampc(-18, Y2 + 7.2, 40, RGB(255, 200, 140), 34, 0.8)
		end

		-- starboard block interiors: living room (z 24..44), TV room (z 44..56), bedroom + balcony on L2
		do
			slab(10, 26, 24, 44, Y1, 0.12, RGB(150, 102, 66), PLANK, "LivingFloor")
			slab(10, 26, 44, 56, Y1, 0.12, RGB(110, 76, 50), PLANK, "TVFloor")
			sofa(15, Y1, 30, 0, 8, 3.2, RGB(210, 196, 172)) sofa(12.6, Y1, 36, -math.pi / 2, 7, 3.2, RGB(210, 196, 172))
			B(17, Y1, 34, 4, 1.2, 4, DTEAK, PLANK, "CoffeeTable")
			tv(25.4, Y1 + 3.6, 36, 8, 4.6, "x")
			for _, z in ipairs({ 30, 40 }) do ghost(B(25.3, Y1 + 5.4, z, 0.3, 3, 3, RGB(176, 120, 76), SM, "WallArt")).Shape = Enum.PartType.Cylinder end
			B(20, Y1, 43, 8, 2.6, 1.6, WOODL, WOOD, "LongDesk")
			plant(24, Y1, 26.5, 1.2) plant(11.6, Y1, 42, 1)
			B(12, Y1, 52, 3, 7, 1.4, WOODL, WOOD, "Bookshelf") B(20, Y1, 54.4, 8, 2.6, 2, WOODL, WOOD, "Desk")
			place(f, CFrame.new(20, Y1 + 1.8, 52), 2, 0.6, 2, DARK, LEA, "Chair")
			ledRing(10.6, 25.4, 24.6, 43.4, Y1 + 8.4, ORANGE) ledRing(10.6, 25.4, 44.6, 55.4, Y1 + 8.4, ORANGE)
			lampc(18, Y1 + 7.2, 33, RGB(255, 200, 140), 36, 0.8) lampc(18, Y1 + 7.2, 50, RGB(255, 200, 140), 30, 0.7)
			-- bedroom
			slab(10, 21, 24, 56, Y2, 0.12, RGB(126, 84, 54), PLANK, "BedroomFloor")
			slab(21, 26, 24, 26, Y2, 0.12, RGB(126, 84, 54), PLANK, "BedroomFloor")
			slab(21, 26, 43.5, 56, Y2, 0.12, RGB(126, 84, 54), PLANK, "BedroomFloor")
			glass(21, Y2, 34.75, 0.15, 3.4, 17.5, 0.5)
			B(18, Y2, 48, 13, 1.0, 12, WOODL, WOOD, "BedPlatform")
			B(18, Y2 + 1.0, 48, 11, 1.5, 10.6, TRIM, FAB, "Mattress")
			B(18, Y2 + 1.0, 53.4, 11.6, 4.4, 1.0, WOODP, WOOD, "Headboard")
			for _, x in ipairs({ 14.6, 21.4 }) do B(x, Y2 + 2.5, 51.8, 3, 0.9, 1.8, RGB(96, 70, 60), FAB, "Pillow") end
			B(18, Y2 + 2.5, 43.6, 11, 0.4, 3.4, RGB(110, 70, 56), FAB, "Throw")
			for _, x in ipairs({ 10.8, 25.2 }) do B(x, Y2, 52, 1.2, 2, 2, WOODL, WOOD, "NightStand") end
			tv(18, Y2 + 3.4, 24.5, 9, 4.8, "z")
			B(25.3, Y2 + 0.2, 34, 0.4, 7.8, 9, RGB(60, 150, 150), FAB, "TealPanel")
			sofa(12.6, Y2, 30, -math.pi / 2, 6, 3, RGB(176, 150, 120))
			ledRing(10.6, 25.4, 24.6, 55.4, Y2 + 8.4, ORANGE)
			lampc(18, Y2 + 7.2, 36, RGB(255, 200, 140), 36, 0.8) lampc(18, Y2 + 7.2, 50, RGB(255, 200, 140), 32, 0.7)
			-- balcony over the centre deck
			slab(10, 26, 10, 24, Y2 - 1, 1, CREAM, SM, "BalconyFloor")
			slab(10, 26, 10, 24, Y2, 0.12, TEAK, PLANK, "BalconyDeck")
			slab(10, 26, 10, 24.4, Y3 - 1, 1, HULL, SM, "BalconyRoof")
			slab(10, 26, 10, 24.4, Y3 - 1.3, 0.3, DTEAK, PLANK, "Soffit")
			ghost(B(18, Y3 + 0.9, 10, 16.4, 0.3, 0.6, BLUE, SM, "RoofStripe"))
			for _, p in ipairs({ { 11, 11 }, { 25, 11 } }) do cyl(p[1], Y1, p[2], Y2 - Y1 - 1, 1.6, HULL, SM, "BalconyColumn") end
			glass(26, Y2, 17, 0.25, 3.4, 14, 0.6) glass(10, Y2, 15, 0.25, 3.4, 10, 0.6) glass(18, Y2, 10, 16, 3.4, 0.25, 0.6)
			deco(B(26, Y2 + 3.4, 17, 0.5, 0.35, 14.4, CHROME, MET, "Rail")) deco(B(18, Y2 + 3.4, 10, 16.4, 0.35, 0.5, CHROME, MET, "Rail")) deco(B(10, Y2 + 3.4, 15, 0.5, 0.35, 10.4, CHROME, MET, "Rail"))
			ghost(B(18, Y2 - 1.9, 9.7, 16, 0.5, 0.5, BLUE, SM, "FasciaStripe"))
			sofa(14, Y2, 13, 0, 7, 3, TRIM) place(f, CFrame.new(22, Y2 + 0.9, 17), 3.4, 1.8, 3.4, DTEAK, PLANK, "BalconyTable")
		end

		-- link corridor over the stern stairs (L2)
		do
			slab(-10, 10, 28, 36, Y2 - 1, 1, CREAM, SM, "UpperFloor")
			slab(-10, 10, 28, 36, Y2, 0.12, RGB(126, 84, 54), PLANK, "LinkFloor")
			wall("x", 28, -10, 10, Y2, 9, HULL, SM, "LinkWall", nil, { { -9, -1 }, { 1, 9 } }, 0.9)
			wall("x", 36, -10, 10, Y2, 9, HULL, SM, "LinkWall", nil, { { -9, -1 }, { 1, 9 } }, 0.9)
			slab(-10, 10, 26.4, 37.6, Y3 - 1, 1, HULL, SM, "LinkRoof")
			pins("x", 28, -10, 10, Y2 + 8.2) pins("x", 36, -10, 10, Y2 + 8.2)
			ghost(B(0, Y3 + 0.9, 37.6, 20.4, 0.3, 0.6, BLUE, SM, "RoofStripe"))
			led(0, Y2 + 8.4, 32, 18, 0.22, 0.22, ORANGE)
			lampc(0, Y2 + 7, 32, RGB(255, 200, 140), 30, 0.8)
		end

		-- =====================================================================================================
		-- CENTRE DECK (z -22..24, y 4): hot tub with curved wall, white column, two cabanas, wood-panelled shower passage, planters
		-- =====================================================================================================
		do
			-- hot tub (port)
			cyl(-19, Y1, 5, 1.8, 13, HULL, SM, "HotTub")
			cyl(-19, Y1 + 1.8, 5, 0.25, 13.6, TEAK, WOOD, "TubRim").CanCollide = false
			local tub = ghost(cyl(-19, Y1 + 1.35, 5, 0.2, 11, TURQ, GLS, "HotTubWater"))
			tub.Transparency = 0.3
			table.insert(pools, tub)
			local fx = Instance.new("ParticleEmitter")
			fx.Texture = "rbxasset://textures/particles/smoke_main.dds"
			fx.Color = ColorSequence.new(RGB(240, 248, 255))
			fx.Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.8), NumberSequenceKeypoint.new(1, 1) })
			fx.Size = NumberSequence.new(1.2, 3.4)
			fx.Lifetime = NumberRange.new(2, 3)
			fx.Rate = 3
			fx.Speed = NumberRange.new(1, 2.4)
			fx.SpreadAngle = Vector2.new(20, 20)
			fx.Parent = tub
			arc(-19, 5, 9, 138, 222, Y1, 5.4, 1.4, HULL, SM, "TubWall", 3)
			arc(-19, 5, 9, 138, 222, Y1 + 5.4, 0.3, 1.9, TEAK, WOOD, "TubWallCap", 3)
			for _, a in ipairs({ 150, 165, 180, 195, 210 }) do
				local hx, hz = -19 + 8.25 * math.cos(math.rad(a)), 5 + 8.25 * math.sin(math.rad(a))
				local hv = ghost(B(hx, Y1 + 3.4, hz, 0.2, 1.7, 1.7, RGB(40, 46, 56), SM, "Hex"))
				hv.Shape = Enum.PartType.Cylinder
				hv.CFrame = CFrame.new(hx, Y1 + 4.25, hz) * CFrame.Angles(0, -math.rad(a), 0)
			end
			B(-14, Y1 + 1.8, 9, 1.4, 0.5, 2.2, TRIM, FAB, "Towels")
			cyl(-23, Y1 + 1.8, 9, 1.1, 0.7, RGB(220, 150, 40), GLS, "Bottle")
			lampc(-19, Y1 + 6, 5, RGB(255, 236, 205), 34, 0.5)
			-- white column (vent stack) near the stairs
			cyl(0, Y1, 8, 13, 8.4, HULL, SM, "Column")
			cyl(0, Y1 + 13, 8, 0.9, 9.6, TRIM, SM, "ColumnCap")
			ghost(cyl(0, Y1 + 10, 8, 0.5, 8.7, BLUE, SM, "ColumnStripe"))
			ghost(cyl(0, Y1 + 13.9, 8, 0.3, 6, DARK, MET, "ColumnHatch"))
			-- cabanas (open toward the middle)
			for _, sg in ipairs({ -1, 1 }) do
				local xa, xb = xr(sg, 22, 32)
				wall("x", -21, xa, xb, Y1, 7.4, HULL, SM, "Cabana", nil, { { xa + 1.5, xb - 1.5, 3, 6.4 } }, 0.7)
				wall("x", -10, xa, xb, Y1, 7.4, HULL, SM, "Cabana", nil, nil, 0.7)
				wall("z", sg * 31.7, -21, -10, Y1, 7.4, HULL, SM, "Cabana", nil, nil, 0.7)
				B(sg * 31.2, Y1 + 0.3, -17, 0.3, 6.4, 3.4, RGB(156, 92, 48), WOOD, "CabanaDoor")
				B(sg * 31.2, Y1 + 0.3, -13, 0.3, 6.4, 3.4, RGB(156, 92, 48), WOOD, "CabanaDoor")
				slab(math.min(xa, xb) - 0.8, math.max(xa, xb) + 0.8, -21.8, -9.2, Y1 + 7.4, 0.8, HULL, SM, "CabanaRoof")
				ghost(B(sg * 22, Y1 + 8.2, -15.5, 0.3, 0.3, 12.4, BLUE, SM, "RoofStripe"))
				sofa(sg * 29.4, Y1, -15.5, sg * math.pi / 2, 9, 3, TRIM)
				B(sg * 26, Y1, -15.5, 2.4, 1.4, 3.2, DTEAK, PLANK, "CoffeeTable")
				lounger(sg * 25.5, -4, sg * math.pi / 2 + math.pi / 2)
				lampc(sg * 27, Y1 + 6, -15.5, WARM, 26, 0.6)
			end
			-- shower passage between two wood-panelled walls (starboard)
			for _, x in ipairs({ 16, 23 }) do wall("z", x, -6.5, 6.5, Y1, 7.4, WOODL, WOOD, "ShowerWall", nil, nil, 1.0) end
			slab(15.4, 23.6, -7, 7, Y1 + 7.4, 0.8, HULL, SM, "ShowerRoof")
			slab(16.5, 22.5, -6.5, 6.5, Y1, 0.1, RGB(176, 124, 80), PLANK, "ShowerFloor")
			for _, p in ipairs({ { 16.7, -3 }, { 16.7, 3 }, { 22.3, -3 }, { 22.3, 3 } }) do
				B(p[1], Y1 + 3.6, p[2], 0.7, 0.28, 1.8, TRIM, SM, "Shelf")
				B(p[1] + (p[1] > 20 and -0.2 or 0.2), Y1 + 3.9, p[2], 0.4, 0.9, 0.5, RGB(64, 46, 34), GLS, "Bottle").CanCollide = false
				B(p[1], Y1 + 0.6, p[2] + 1.6, 0.3, 3, 1.3, TRIM, FAB, "Towel").CanCollide = false
			end
			lampc(19.5, Y1 + 6.4, 0, RGB(255, 228, 190), 22, 0.7)
			-- benches, planters and the odd crate
			B(-8, Y1, -17, 6, 1.8, 1.8, DTEAK, PLANK, "Bench") B(-8, Y1 + 1.8, -17.7, 6, 1.8, 0.4, DTEAK, PLANK, "BenchBack")
			plant(-3, Y1, -18, 1.2) plant(4, Y1, -18, 1.2) plant(-30, Y1, 14, 1.1) plant(30, Y1, 16, 1.1) plant(-5, Y1, 19, 1)
			crate(-28, Y1, -2, 14, 2.8)
			lampc(0, Y1 + 12, -4, RGB(255, 246, 232), 70, 0.45) lampc(0, Y1 + 12, 16, RGB(255, 246, 232), 60, 0.4)
			-- white arch tubes over the viewing decks
			for _, sg in ipairs({ -1, 1 }) do
				for _, z in ipairs({ 30, 48, -32, -52 }) do
					local prev
					for i = 0, 6 do
						local t = i / 6
						local p = Vector3.new(sg * (33.3 - 7 * t), Y1 + 9.4 * (1 - (1 - t) ^ 2), z)
						if prev then deco(tube(prev, p, 1.1, TRIM, SM, "Arch")) end
						prev = p
					end
				end
			end
		end

		-- =====================================================================================================
		-- SPAWNS
		-- =====================================================================================================
		local sp = { Alpha = {}, Bravo = {}, FFA = {} }
		for _, p in ipairs({ { -14, 86 }, { 14, 86 }, { -21, 90 }, { 21, 90 }, { -4, 84 }, { 4, 82 }, { -18, 80 }, { 18, 78 } }) do table.insert(sp.Alpha, Vector3.new(p[1], Y0, p[2])) end
		for _, p in ipairs({ { -6, -90 }, { 6, -90 }, { -8, -100 }, { 8, -100 }, { -5, -108 }, { 5, -108 }, { -6, -116 }, { 6, -116 } }) do table.insert(sp.Bravo, Vector3.new(p[1], Y0, p[2])) end
		for _, p in ipairs({ { -8, Y1, -14 }, { 0, Y1, 18 }, { 26, Y1, -2 }, { -30, Y1, 18 }, { 16, Y1, -44 }, { -19, Y1, -26 }, { -22, Y1, -58 },
			{ 0, Y2, -62 }, { -21, Y2, -27 }, { 20, Y2, -34 }, { -18, Y1, 28 }, { 14, Y1, 48 }, { 16, Y2, 34 }, { -16, Y2, 30 }, { 0, YE, 0 },
			{ -30, Y1, -40 }, { 30, Y1, 40 }, { 24, Y0, 70 }, { -10, Y0, -88 }, { -10, Y3, -34 } }) do table.insert(sp.FFA, Vector3.new(p[1], p[2], p[3])) end
		setSpawns(sp)

		-- =====================================================================================================
		-- WORLD: cable-stayed bridge, container quay + cranes, city skyline, moored yachts, sailboats (all non-solid)
		-- =====================================================================================================
		do
			local R2 = Random.new(77)
			local bd = Instance.new("Model")
			bd.Name = "Backdrop"
			bd.Parent = f
			local function G(cx, by, cz, sx, sy, sz, col, mat, name) return ghost(blk(bd, cx, by, cz, sx, sy, sz, col, mat, name)) end
			local PALE = RGB(222, 220, 226)
			-- bridge
			G(-700, 30, -430, 1400, 3, 34, RGB(196, 194, 200), SM, "BridgeDeck") G(700, 30, -430, 1400, 3, 34, RGB(196, 194, 200), SM, "BridgeDeck")
			G(0, 33, -430, 2800, 1.4, 1.2, PALE, SM, "BridgeRail")
			for _, px in ipairs({ -200, 330 }) do
				for _, s in ipairs({ -1, 1 }) do G(px + s * 7, SEAY, -430, 7, 250, 8, PALE, SM, "Pylon") end
				G(px, 190, -430, 20, 5, 8, PALE, SM, "PylonBeam")
				for k = 1, 10 do
					for _, s in ipairs({ -1, 1 }) do
						local top = Vector3.new(px, 236 - k * 3.2, -430)
						local bot = Vector3.new(px + s * (k * 30), 33, -430)
						ghost(tube(top, bot, 0.9, PALE, MET, "Stay"))
					end
				end
			end
			for k = -5, 5 do G(k * 260 + 40, SEAY, -430, 12, 44, 12, RGB(158, 158, 164), CONC, "BridgePier") end
			-- container quay + cranes
			G(0, SEAY, -650, 2400, 9, 90, RGB(150, 150, 154), CONC, "Quay")
			for i = 1, 70 do
				local x, z = R2:NextNumber(-900, 900), R2:NextNumber(-675, -625)
				local c = ({ RGB(176, 52, 44), RGB(44, 92, 150), RGB(206, 160, 44), RGB(60, 118, 80), RGB(190, 190, 196), RGB(150, 76, 40) })[R2:NextInteger(1, 6)]
				G(x, SEAY + 9, z, 24, 9, 9, c, MET, "Container")
				if R2:NextNumber() < 0.5 then G(x, SEAY + 18, z, 24, 9, 9, c, MET, "Container") end
			end
			for _, cx in ipairs({ -600, -250, 150, 520 }) do
				for _, lx in ipairs({ -14, 14 }) do for _, lz in ipairs({ -616, -664 }) do G(cx + lx, SEAY + 9, lz, 3, 70, 3, RGB(190, 60, 48), MET, "CraneLeg") end end
				G(cx, SEAY + 78, -640, 36, 4, 56, RGB(190, 60, 48), MET, "CraneTop")
				G(cx, SEAY + 80, -590, 6, 5, 140, RGB(200, 70, 52), MET, "CraneBoom")
			end
			-- skyline
			for i = 1, 90 do
				local w, h = R2:NextNumber(40, 110), R2:NextNumber(120, 420)
				local x = R2:NextNumber(-1500, 1500)
				local c = RGB(R2:NextInteger(150, 196), R2:NextInteger(166, 206), R2:NextInteger(180, 220))
				G(x, SEAY, R2:NextNumber(-1500, -950), w, h, w * R2:NextNumber(0.7, 1.2), c, (i % 3 == 0) and GLS or SM, "Tower")
			end
			-- moored yachts
			local function megaYacht(cx, cz, len, w)
				G(cx, SEAY, cz, w, 18, len, RGB(236, 234, 228), SM, "OtherHull")
				G(cx, SEAY + 5, cz, w + 0.2, 1.2, len, NAVY, SM, "OtherStripe")
				G(cx, SEAY + 18, cz, w - 3, 9, len * 0.66, RGB(240, 238, 232), SM, "OtherDeck1")
				G(cx, SEAY + 22, cz, w - 3 + 0.2, 2, len * 0.64, RGB(30, 40, 56), GLS, "OtherGlass")
				G(cx, SEAY + 27, cz - len * 0.05, w - 9, 8, len * 0.4, RGB(240, 238, 232), SM, "OtherDeck2")
				G(cx, SEAY + 35, cz - len * 0.05, w - 13, 5, len * 0.22, RGB(240, 238, 232), SM, "OtherDeck3")
				G(cx, SEAY + 40, cz - len * 0.05, 1, 12, 1, CHROME, MET, "OtherMast")
			end
			megaYacht(-98, -20, 150, 26) megaYacht(112, 70, 110, 22)
			-- sailboats
			for i = 1, 9 do
				local x, z = R2:NextNumber(-600, 600), R2:NextNumber(-380, -200)
				if math.abs(x) < 120 then x += 240 end
				local base = CFrame.new(x, SEAY + 1, z) * CFrame.Angles(0, math.rad(R2:NextNumber(0, 360)), 0)
				ghost(place(bd, base * CFrame.new(0, 1, 0), 5, 2.4, 16, TRIM, SM, "SailHull"))
				ghost(place(bd, base * CFrame.new(0, 10, 0), 0.4, 18, 0.4, CHROME, MET, "SailMast"))
				local w = Instance.new("WedgePart")
				w.Anchored, w.CanCollide, w.CanQuery, w.CanTouch = true, false, false, false
				w.Size = Vector3.new(0.2, 14, 9)
				w.CFrame = base * CFrame.new(0, 10, 1.6)
				w.Color, w.Material, w.Name = RGB(246, 240, 232), Enum.Material.Fabric, "Sail"
				w.Parent = bd
			end
		end
		task.spawn(function()
			local t = 0
			while pools[1] and pools[1].Parent do
				t += 0.25
				for i, w in ipairs(pools) do if w.Parent then w.Transparency = 0.3 + 0.07 * math.sin(t * 2.2 + i) end end
				task.wait(0.25)
			end
		end)
		-- ----- scale the whole ship up so it feels huge next to a player -----
		local K = 1.92
		local seaShift = SEAY * K - SEAY
		for _, d in ipairs(f:GetDescendants()) do
			if d:IsA("BasePart") then
				if d.Name == "Ocean" then
					d.Position = d.Position + Vector3.new(0, seaShift, 0)
				else
					local c = d.CFrame
					d.Size = d.Size * K
					d.CFrame = CFrame.new(c.Position * K) * c.Rotation
				end
			elseif d:IsA("PointLight") then
				d.Range = d.Range * K
			end
		end
		for _, list in pairs(sp) do
			for i, v in ipairs(list) do list[i] = v * K end
		end
		setSpawns(sp)
		return { spawns = sp, half = 180 * K, halfX = 44 * K }
	end

	-- =================================================================
	-- PROVING GROUNDS  (id GRID): military live-fire range. Tower, shoot-houses, vehicle yards.
	-- =================================================================
	-- =================================================================
	-- TERMINUS CENTRAL (id STATION): a massive Call-of-Duty style rail terminus (600 x 600). Alpha stages in the bus station / car park to the
	-- south, Bravo in the freight yard to the north. Between them: forecourt + fountain, a grand hall with mezzanines, a hotel and arcade,
	-- six tracks with four platforms, parked trains you can walk through, two glazed footbridges, a signal box tower, a warehouse and a depot.
	-- =================================================================
	MapBuilders.STATION = function(f)
		MapRand = Random.new(31)
		local M = Enum.Material
		local FH, SW = 12, 0.1
		local sp = { Alpha = {}, Bravo = {}, FFA = {} }
		for i = -4, 4 do
			table.insert(sp.Alpha, Vector3.new(i * 50, 0.1, 272))
			table.insert(sp.Bravo, Vector3.new(i * 50, 0.1, -272))
		end
		for _, p in ipairs({ { -80, 182, 0.1 }, { 80, 182, 0.1 }, { -160, 210, 0.1 }, { 160, 210, 0.1 }, { 0, 128, 0.1 }, { 0, 60, 0.1 }, { -62, 84, 0.1 }, { 62, 84, 0.1 },
			{ -60, 12, 0.1 }, { 60, 12, 0.1 }, { -100, -14, 1.8 }, { 100, -14, 1.8 }, { -60, -79, 1.8 }, { 60, -106, 1.8 }, { 0, -175, 0.1 }, { 0, -195, 0.1 },
			{ -160, -250, 0.1 }, { 160, -250, 0.1 }, { 0, -284, 0.1 } }) do
			table.insert(sp.FFA, Vector3.new(p[1], p[3], p[2]))
		end
		setSpawns(sp)
		local function B(cx, by, cz, sx, sy, sz, col, mat, name) return blk(f, cx, by, cz, sx, sy, sz, col, mat, name) end
		local function ghost(p) p.CanCollide = false; p.CanQuery = false; return p end
		local function ball(cx, cy, cz, d, col, mat, name)
			local p = B(cx, cy - d / 2, cz, d, d, d, col, mat, name)
			p.Shape = Enum.PartType.Ball
			return p
		end
		local function cyl(cx, by, cz, d, h, col, mat, name)
			local p = B(cx, by, cz, d, h, d, col, mat, name)
			p.Shape = Enum.PartType.Cylinder
			p.CFrame = CFrame.new(cx, by + h / 2, cz) * CFrame.Angles(0, 0, math.rad(90))
			p.Size = Vector3.new(h, d, d)
			return p
		end

		-- ---------- palettes ----------
		local STY = {
			{ M.Brick, { RGB(128, 74, 58), RGB(116, 68, 54), RGB(142, 88, 66), RGB(104, 62, 52) } },
			{ M.Concrete, { RGB(150, 148, 142), RGB(134, 134, 132), RGB(160, 154, 144), RGB(120, 122, 124) } },
			{ M.Sandstone, { RGB(176, 156, 124), RGB(166, 148, 120), RGB(150, 136, 112) } },
			{ M.Limestone, { RGB(184, 176, 158), RGB(172, 166, 150) } },
			{ M.Granite, { RGB(96, 100, 104), RGB(84, 88, 94) } },
		}
		local SLABC = RGB(146, 144, 140)
		local GLASSC = RGB(52, 74, 94)
		local ROOFC = RGB(70, 72, 76)

		-- ---------- structural helpers ----------
		local function wallRun(axis, fix, a0, a1, by, h, th, col, mat, ops, name)
			table.sort(ops, function(p, q) return p.c < q.c end)
			local function piece(u0, u1, y0, y1)
				if u1 - u0 < 0.05 or y1 - y0 < 0.05 then return end
				if axis == "x" then
					B((u0 + u1) / 2, by + y0, fix, u1 - u0, y1 - y0, th, col, mat, name or "Wall")
				else
					B(fix, by + y0, (u0 + u1) / 2, th, y1 - y0, u1 - u0, col, mat, name or "Wall")
				end
			end
			local cur = a0
			for _, o in ipairs(ops) do
				local lo, hi = o.c - o.w / 2, o.c + o.w / 2
				piece(cur, lo, 0, h)
				piece(lo, hi, 0, o.y0)
				piece(lo, hi, o.y1, h)
				if o.pane then
					local g
					if axis == "x" then g = B(o.c, by + o.y0, fix, o.w, o.y1 - o.y0, 0.25, RGB(70, 96, 118), M.Glass, "Pane")
					else g = B(fix, by + o.y0, o.c, 0.25, o.y1 - o.y0, o.w, RGB(70, 96, 118), M.Glass, "Pane") end
					g.Transparency = 0.45
					g.CanCollide = false
				end
				cur = hi
			end
			piece(cur, a1, 0, h)
		end
		-- walls' window / door openings along [a0, a1]
		local function openings(a0, a1, doors, ground, spacing, pane)
			local ops = {}
			for _, d in ipairs(doors or {}) do
				table.insert(ops, { c = d, w = 7, y0 = 0, y1 = 9.5 })
			end
			local c = a0 + 7
			while c < a1 - 6 do
				local ok = true
				if ground then
					for _, d in ipairs(doors or {}) do
						if math.abs(c - d) < 8 then ok = false end
					end
				end
				if ok then table.insert(ops, { c = c, w = 5, y0 = 3.2, y1 = 8.4, pane = pane }) end
				c += spacing
			end
			return ops
		end
		local function slabHole(top, xi0, xi1, zi0, zi1, strip, sx0, sx1, col, mat, th)
			th = th or 1
			local by = top - th
			local zs = zi1 - strip
			B((xi0 + xi1) / 2, by, (zi0 + zs) / 2, xi1 - xi0, th, zs - zi0, col, mat or M.Concrete, "Slab")
			if sx0 then
				if sx0 > xi0 + 0.05 then B((xi0 + sx0) / 2, by, (zs + zi1) / 2, sx0 - xi0, th, strip, col, mat or M.Concrete, "Slab") end
				if sx1 < xi1 - 0.05 then B((sx1 + xi1) / 2, by, (zs + zi1) / 2, xi1 - sx1, th, strip, col, mat or M.Concrete, "Slab") end
			else
				B((xi0 + xi1) / 2, by, (zs + zi1) / 2, xi1 - xi0, th, strip, col, mat or M.Concrete, "Slab")
			end
		end
		local function crate(x, by, z, sx, sy, sz, rot)
			return blkR(f, x, by, z, sx, sy, sz, rot or R(0, 90), RGB(92 + MapRand:NextInteger(0, 20), 80 + MapRand:NextInteger(0, 14), 60), M.Wood, "Crate")
		end
		local function roofProps(cx, top, cz, w, d)
			for _ = 1, 3 do
				local x, z = cx + R(-w / 2 + 8, w / 2 - 8), cz + R(-d / 2 + 8, d / 2 - 14)
				B(x, top, z, R(4, 7), R(2, 3.4), R(4, 7), RGB(112, 116, 120), M.Metal, "ACUnit")
			end
			if MapRand:NextNumber() < 0.6 then
				local x, z = cx + R(-w / 4, w / 4), cz + R(-d / 4, d / 6)
				cyl(x, top + 3, z, 6, 7, RGB(104, 78, 56), M.Wood, "WaterTank")
				for _, o in ipairs({ { -2, -2 }, { 2, -2 }, { -2, 2 }, { 2, 2 } }) do B(x + o[1], top, z + o[2], 0.5, 3, 0.5, RGB(60, 60, 64), M.Metal, "TankLeg") end
			end
			ghost(B(cx + R(-8, 8), top, cz + R(-6, 6), 0.4, 11, 0.4, RGB(50, 50, 54), M.Metal, "Antenna"))
		end

		-- ---------- an enterable building (hollow, several storeys, stairs and roof access) ----------
		local function building(cx, cz, w, d, n, sty, opt)
			opt = opt or {}
			local mat = opt.mat or sty[1]
			local cols = sty[2]
			local col = opt.col or cols[MapRand:NextInteger(1, #cols)]
			local th = 1.2
			local X0, X1, Z0, Z1 = cx - w / 2, cx + w / 2, cz - d / 2, cz + d / 2
			local xi0, xi1, zi0, zi1 = X0 + th, X1 - th, Z0 + th, Z1 - th
			local STRIP, RUN = 7, 22
			local doorsS = opt.doorsS or { cx - 12, cx + 12 }
			local doorsN = opt.doorsN or { cx + 12 }
			local doorsW = { cz - 8 }
			local doorsE = { cz - 8 }
			local pane = opt.pane
			local sp0 = opt.spacing or 10
			for fl = 0, n - 1 do
				local L = SW + fl * FH
				local g = fl == 0
				wallRun("x", Z0 + th / 2, X0, X1, L, FH, th, col, mat, openings(X0, X1, g and doorsS or nil, g, sp0, pane))
				wallRun("x", Z1 - th / 2, X0, X1, L, FH, th, col, mat, openings(X0, X1, g and doorsN or nil, g, sp0, pane))
				wallRun("z", X0 + th / 2, Z0 + th, Z1 - th, L, FH, th, col, mat, openings(Z0 + th, Z1 - th, g and doorsW or nil, g, 14, pane))
				wallRun("z", X1 - th / 2, Z0 + th, Z1 - th, L, FH, th, col, mat, openings(Z0 + th, Z1 - th, g and doorsE or nil, g, 14, pane))
				-- one interior partition with two door gaps, plus some furniture / cover
				local px = cx + R(-10, 6)
				local zf1 = zi1 - STRIP
				local half = (zf1 - zi0)
				wallRun("z", px, zi0, zf1, L, FH - 1, 0.8, RGB(168, 164, 154), M.Concrete,
					{ { c = zi0 + half * 0.28, w = 7, y0 = 0, y1 = 9 }, { c = zi0 + half * 0.74, w = 7, y0 = 0, y1 = 9 } }, "Partition")
				for _ = 1, 3 do
					local x = R(xi0 + 6, xi1 - 6)
					local z = R(zi0 + 6, zf1 - 5)
					if math.abs(x - px) > 4 then
						local kind = MapRand:NextInteger(1, 3)
						if kind == 1 then B(x, L, z, R(5, 8), R(2.9, 3.2), 2.6, RGB(96, 76, 56), M.WoodPlanks, "Desk")
						elseif kind == 2 then B(x, L, z, 2.4, R(4.3, 4.6), R(5, 8), RGB(78, 80, 84), M.Metal, "Shelf")
						else crate(x, L, z, 3.2, 3.2, 3.2) end
					end
				end
			end
			-- slabs + stairs
			for k = 1, n do
				local top = SW + k * FH
				local dir = ((k - 1) % 2 == 0) and 1 or -1
				local sx0, sx1
				if dir == 1 then sx0 = xi0 + 3; sx1 = sx0 + RUN else sx1 = xi1 - 3; sx0 = sx1 - RUN end
				slabHole(top, xi0, xi1, zi0, zi1, STRIP, sx0, sx1, SLABC, M.Concrete, 1)
				local zc = zi1 - STRIP / 2
				local L = SW + (k - 1) * FH
				local a = Vector3.new(dir == 1 and sx0 or sx1, L, zc)
				local b = Vector3.new(dir == 1 and sx1 or sx0, top, zc)
				ramp(f, a, b, 5.6, RGB(96, 96, 98), M.Concrete)
				if k == n then
					-- roof: parapet all round
					local ph = 1.5
					B(cx, top, Z0 + th / 2, w, ph, th, col, mat, "Parapet")
					B(cx, top, Z1 - th / 2, w, ph, th, col, mat, "Parapet")
					B(X0 + th / 2, top, cz, th, ph, d - 2 * th, col, mat, "Parapet")
					B(X1 - th / 2, top, cz, th, ph, d - 2 * th, col, mat, "Parapet")
					roofProps(cx, top, cz, w, d)
				end
			end
			-- cornice band + awning over the ground-floor doors
			B(cx, SW + FH - 0.2, Z0 - 0.35, w + 0.8, 0.5, 0.7, RGB(100, 98, 94), M.Concrete, "Cornice")
			for _, dx in ipairs(doorsS) do
				B(dx, SW + 9.8, Z0 - 1.6, 9, 0.4, 3.2, RGB(52, 60, 70), M.Metal, "Awning")
			end
			return { X0 = X0, X1 = X1, Z0 = Z0, Z1 = Z1, top = SW + n * FH }
		end

		-- solid (non-enterable) block with window bands, for skyline mass and cover
		local function solid(cx, cz, w, d, h, sty)
			local col = sty[2][MapRand:NextInteger(1, #sty[2])]
			B(cx, SW, cz, w, h, d, col, sty[1], "Building")
			B(cx, SW + h, cz, w + 1, 1.2, d + 1, RGB(96, 94, 90), M.Concrete, "Cornice")
			for fl = 0, math.floor(h / FH) - 1 do
				local y = SW + fl * FH + 4
				ghost(B(cx, y, cz - d / 2 - 0.05, w - 4, 4.4, 0.3, RGB(34, 44, 56), M.Glass, "Window"))
				ghost(B(cx, y, cz + d / 2 + 0.05, w - 4, 4.4, 0.3, RGB(34, 44, 56), M.Glass, "Window"))
				ghost(B(cx - w / 2 - 0.05, y, cz, 0.3, 4.4, d - 4, RGB(34, 44, 56), M.Glass, "Window"))
				ghost(B(cx + w / 2 + 0.05, y, cz, 0.3, 4.4, d - 4, RGB(34, 44, 56), M.Glass, "Window"))
			end
			roofProps(cx, SW + h + 1.2, cz, w, d)
		end

		-- a car on an elevated deck (y = deck height)
		local function carD(x, y, z, rotY, col)
			if nearSpawn(x, z, 9) then return end
			col = col or CAR_COLS[MapRand:NextInteger(1, #CAR_COLS)]
			local base = CFrame.new(x, y, z) * CFrame.Angles(0, math.rad(rotY), 0)
			place(f, base * CFrame.new(0, 1.5, 0), 5, 1.6, 11, col, M.Metal, "DeckCar")
			place(f, base * CFrame.new(0, 2.9, 0.6), 4.4, 1.4, 5.4, RGB(18, 22, 28), M.SmoothPlastic, "DeckCabin")
			place(f, base * CFrame.new(0, 0.8, 3.6), 5.4, 1.6, 1.6, RGB(20, 20, 22), M.Metal, "DeckWheels")
			place(f, base * CFrame.new(0, 0.8, -3.6), 5.4, 1.6, 1.6, RGB(20, 20, 22), M.Metal, "DeckWheels")
		end

		-- ---------- street props ----------
		local function tree(x, z, s)
			s = s or R(0.9, 1.3)
			B(x, SW, z, 1.3, 5.5 * s, 1.3, RGB(86, 64, 44), M.Wood, "Trunk")
			local c = ball(x, SW + 5 * s + 4.4 * s, z, 9 * s, RGB(54 + MapRand:NextInteger(0, 16), 86 + MapRand:NextInteger(0, 20), 46), M.Grass, "Leaves")
			c.CanCollide = false
		end
		local function bench(x, z, rot)
			if nearSpawn(x, z, 6) then return end
			local c = CFrame.new(x, SW, z) * CFrame.Angles(0, math.rad(rot), 0)
			place(f, c * CFrame.new(0, 1.3, 0), 5, 0.4, 1.6, RGB(110, 80, 52), M.WoodPlanks, "Bench")
			place(f, c * CFrame.new(0, 2.1, -0.7), 5, 1.2, 0.3, RGB(110, 80, 52), M.WoodPlanks, "BenchBack")
			place(f, c * CFrame.new(-2.2, 0.6, 0), 0.4, 1.2, 1.4, RGB(50, 52, 56), M.Metal, "BenchLeg")
			place(f, c * CFrame.new(2.2, 0.6, 0), 0.4, 1.2, 1.4, RGB(50, 52, 56), M.Metal, "BenchLeg")
		end
		local function dumpster(x, z, rot)
			if nearSpawn(x, z, 8) then return end
			place(f, CFrame.new(x, SW + 1.8, z) * CFrame.Angles(0, math.rad(rot), 0), 6, 3.6, 3.2, RGB(46, 84, 62), M.Metal, "Dumpster")
		end
		local function hydrant(x, z)
			B(x, SW, z, 1, 2.4, 1, RGB(170, 40, 34), M.Metal, "Hydrant")
		end
		local function bus(x, z, rot, col)
			if nearSpawn(x, z, 14) then return end
			local c = CFrame.new(x, 0.1, z) * CFrame.Angles(0, math.rad(rot), 0)
			place(f, c * CFrame.new(0, 2.9, 0), 7, 4.6, 22, col or RGB(46, 74, 112), M.Metal, "Bus")
			place(f, c * CFrame.new(0, 3.6, 0), 7.2, 1.6, 20, RGB(24, 30, 38), M.Glass, "BusWindows")
			place(f, c * CFrame.new(0, 1, 7), 7.2, 2, 3, RGB(20, 20, 22), M.Metal, "BusWheels")
			place(f, c * CFrame.new(0, 1, -7), 7.2, 2, 3, RGB(20, 20, 22), M.Metal, "BusWheels")
		end
		local function fence(x0, z0, x1, z1)
			local len = math.sqrt((x1 - x0) ^ 2 + (z1 - z0) ^ 2)
			local mx, mz = (x0 + x1) / 2, (z0 + z1) / 2
			local ang = math.atan2(x1 - x0, z1 - z0)
			local p = place(f, CFrame.new(mx, SW + 3, mz) * CFrame.Angles(0, ang, 0), 0.3, 6, len, RGB(110, 114, 118), M.Metal, "Fence")
			p.Transparency = 0.55
			for i = 0, math.floor(len / 12) do
				local t = i * 12 / len
				B(x0 + (x1 - x0) * t, SW, z0 + (z1 - z0) * t, 0.6, 6.4, 0.6, RGB(70, 72, 76), M.Metal, "FencePost")
			end
		end
		local function streetLight(x, z, lit)
			pole(f, x, z, lit)
		end
		local function billboard(x, z, rotY, text, col)
			B(x - 5, 0.1, z, 0.9, 16, 0.9, RGB(60, 62, 66), M.Metal, "BillPost")
			B(x + 5, 0.1, z, 0.9, 16, 0.9, RGB(60, 62, 66), M.Metal, "BillPost")
			local p = Instance.new("Part")
			p.Anchored = true
			p.Size = Vector3.new(22, 8, 0.8)
			p.CFrame = CFrame.new(x, 20, z) * CFrame.Angles(0, math.rad(rotY), 0)
			p.Color = RGB(20, 22, 26)
			p.Material = M.SmoothPlastic
			p.Name = "Billboard"
			p.Parent = f
			for _, face in ipairs({ Enum.NormalId.Front, Enum.NormalId.Back }) do
				local g = Instance.new("SurfaceGui")
				g.Face = face
				g.SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud
				g.PixelsPerStud = 30
				g.LightInfluence = 0.2
				g.Parent = p
				local t = Instance.new("TextLabel")
				t.Size = UDim2.new(1, 0, 1, 0)
				t.BackgroundTransparency = 1
				t.Text = text
				t.TextScaled = true
				t.Font = Enum.Font.GothamBlack
				t.TextColor3 = col
				t.Parent = g
			end
		end

		-- ============================ TERMINUS: palettes ============================
		local STONEM = { M.Limestone, { RGB(188, 178, 156), RGB(176, 168, 148), RGB(198, 188, 164) } }
		local BRICKM = STY[1]
		local STONEC, STONE2, MARB = RGB(190, 180, 158), RGB(160, 150, 130), RGB(206, 200, 186)
		local GREEN, DGREEN, CRM = RGB(44, 84, 70), RGB(26, 52, 44), RGB(214, 204, 176)
		local STEELC, DARK, RUSTC, YELC = RGB(98, 104, 112), RGB(34, 36, 40), RGB(124, 72, 50), RGB(232, 178, 36)
		local ASPH, PAVE, PLATC, GRAV = RGB(58, 60, 64), RGB(138, 132, 120), RGB(150, 148, 142), RGB(86, 80, 72)
		local WARM = RGB(255, 214, 150)
		local function patch(cx, cz, sx, sz, col, mat, by, hh)
			local p = B(cx, by or 0.1, cz, sx, hh or 0.08, sz, col, mat or M.Concrete, "Paint")
			p.CastShadow = false
			return p
		end
		local function paintLine(cx, cz, sx, sz, col)
			local p = B(cx, 0.17, cz, sx, 0.05, sz, col, M.SmoothPlastic, "Marking")
			p.CanCollide, p.CanQuery, p.CastShadow = false, false, false
		end
		local function glassPart(cx, by, cz, sx, sy, sz, tr, col)
			local p = B(cx, by, cz, sx, sy, sz, col or RGB(96, 140, 160), M.Glass, "Glass")
			p.Transparency, p.CanQuery, p.CastShadow = tr or 0.5, false, false
			return p
		end
		local function column(x, z, h, w, col, mat, by)
			B(x, by or SW, z, w, h, w, col, mat or M.Concrete, "Column")
			B(x, (by or SW) + h - 0.8, z, w + 0.8, 0.8, w + 0.8, col, mat or M.Concrete, "ColumnCap")
		end
		local function hang(x, y, z, col, rng, br)
			lamp(f, Vector3.new(x, y, z), col or WARM, rng or 50, br or 1)
		end
		local function kiosk(x, z, rot, col, label)
			if nearSpawn(x, z, 8) then return end
			local c = CFrame.new(x, SW, z) * CFrame.Angles(0, math.rad(rot or 0), 0)
			place(f, c * CFrame.new(0, 2.2, 0), 8, 4.4, 5, col or DGREEN, M.Metal, "Kiosk")
			place(f, c * CFrame.new(0, 4.6, 0.3), 9.4, 0.4, 6.4, STEELC, M.Metal, "KioskRoof")
			place(f, c * CFrame.new(0, 3.3, 2.55), 6.4, 1.4, 0.15, RGB(250, 220, 150), M.Neon, "KioskGlow").CanCollide = false
			place(f, c * CFrame.new(0, 1.2, 2.7), 7.2, 0.9, 0.7, STEELC, M.Metal, "KioskCounter")
		end
		local function trainBench(x, z, rot) bench(x, z, rot or 0) end
		local function shedWalls(x0, x1, z0, z1, h, col, mat, opsS, opsN, opsW, opsE)
			local th = 1.4
			wallRun("x", z0 + th / 2, x0, x1, SW, h, th, col, mat, opsS or {})
			wallRun("x", z1 - th / 2, x0, x1, SW, h, th, col, mat, opsN or {})
			wallRun("z", x0 + th / 2, z0 + th, z1 - th, SW, h, th, col, mat, opsW or {})
			wallRun("z", x1 - th / 2, z0 + th, z1 - th, SW, h, th, col, mat, opsE or {})
		end
		local function opsRow(a0, a1, doors, dw, dh, wy0, wy1, spacing, ww, pane)
			local ops = {}
			for _, d in ipairs(doors or {}) do table.insert(ops, { c = d, w = dw, y0 = 0, y1 = dh }) end
			local c = a0 + spacing / 2
			while c < a1 - ww do
				local ok = true
				for _, d in ipairs(doors or {}) do if math.abs(c - d) < dw / 2 + ww / 2 + 1.5 then ok = false end end
				if ok then table.insert(ops, { c = c, w = ww, y0 = wy0, y1 = wy1, pane = pane }) end
				c += spacing
			end
			return ops
		end
		-- stairs from the ground (a) up to a roof or deck (b), with low side rails
		local function stairs(a, b, w, col)
			ramp(f, a, b, w, col or RGB(98, 98, 100), M.Concrete)
			local d = (b - a)
			local dir = Vector3.new(d.X, 0, d.Z).Unit
			local side = Vector3.new(-dir.Z, 0, dir.X)
			local len = d.Magnitude
			for _, s in ipairs({ -1, 1 }) do
				local p = place(f, CFrame.lookAt((a + b) / 2 + side * s * (w / 2 + 0.2) + Vector3.new(0, 0.6, 0), b + side * s * (w / 2 + 0.2) + Vector3.new(0, 0.6, 0)),
					0.3, 1.3, len, STEELC, M.Metal, "StairRail")
				p.CanCollide = false
			end
		end

		-- ============================ ground + perimeter ============================
		B(0, -3.2, 0, 1600, 3, 1600, RGB(70, 68, 62), M.Ground, "Outskirts")
		B(0, -1.9, 0, 640, 2, 640, ASPH, M.Asphalt, "Ground")
		patch(0, 184, 600, 66, PAVE, M.Pavement)                 -- forecourt
		patch(0, 12, 520, 38, PAVE, M.Pavement)                  -- concourse
		patch(0, 90, 200, 120, MARB, M.Marble, 0.1, 0.12)        -- hall floor
		for k = -80, 80, 40 do patch(k, 90, 0.7, 118, STONE2, M.Marble, 0.1, 0.14) end
		patch(0, -172, 520, 74, GRAV, M.Pebble)                  -- freight yard
		patch(0, -258, 600, 80, RGB(70, 70, 72), M.Concrete)     -- bravo yard
		patch(0, 262, 600, 66, RGB(52, 54, 58), M.Asphalt, 0.1, 0.1) -- alpha car park
		-- perimeter wall (also the skyline edge)
		for _, s in ipairs({ -1, 1 }) do
			B(0, SW, s * 296, 596, 11, 2.4, RGB(110, 108, 102), M.Concrete, "PerimeterWall")
			B(s * 296, SW, 0, 2.4, 11, 596, RGB(110, 108, 102), M.Concrete, "PerimeterWall")
			B(0, SW + 11, s * 296, 598, 0.8, 3.4, RGB(78, 78, 80), M.Concrete, "PerimeterCap")
			B(s * 296, SW + 11, 0, 3.4, 0.8, 598, RGB(78, 78, 80), M.Concrete, "PerimeterCap")
		end

		-- ============================ ALPHA CAR PARK + BUS STATION (z 215 .. 295) ============================
		for _, row in ipairs({ { 244, 0 }, { 258, 180 } }) do
			for x = -240, 240, 12 do paintLine(x, row[1], 0.3, 11, RGB(220, 220, 214)) end
		end
		for x = -228, 228, 25 do
			if MapRand:NextNumber() < 0.7 then car(f, x + R(-2, 2), 244, 0 + R(-6, 6)) end
			if MapRand:NextNumber() < 0.6 then car(f, x + 12 + R(-2, 2), 258, 180 + R(-6, 6)) end
		end
		for _, x in ipairs({ -125, -75, -25, 25, 75, 125, -175, 175 }) do pole(f, x, 232, true) end
		for _, x in ipairs({ -150, -50, 50, 150 }) do jersey(f, x, 226, 0) jersey(f, x + 6, 226, 0) end
		bus(-215, 232, 90, RGB(150, 40, 36)) bus(215, 232, 90, RGB(40, 90, 140))
		kiosk(-100, 224, 0) kiosk(100, 224, 0)
		for _, p in ipairs({ { -270, 262 }, { 270, 262 } }) do
			solid(p[1], p[2], 44, 70, 38, BRICKM)
		end
		for _, x in ipairs({ -260, 260 }) do for z = 230, 290, 20 do hydrant(x, z) end end
		for x = -280, 280, 20 do paintLine(x, 219, 8, 0.4, YELC) end

		-- ============================ FORECOURT (z 150 .. 218) ============================
		cyl(0, 0.1, 182, 30, 2.4, STONEC, M.Concrete, "FountainRim")
		cyl(0, 2.4, 182, 26, 0.5, RGB(70, 130, 168), M.Glass, "Water").CanQuery = false
		cyl(0, 2.6, 182, 6, 5, STONE2, M.Concrete, "FountainCore")
		ball(0, 9.4, 182, 5, STONEC, M.Marble, "FountainBall")
		do -- clock tower
			local cz = 214
			B(0, SW, cz, 12, 4, 12, STONE2, M.Granite, "ClockBase")
			B(0, SW + 4, cz, 9, 32, 9, STONEC, M.Limestone, "ClockTower")
			B(0, SW + 36, cz, 12, 2, 12, STONE2, M.Granite, "ClockLedge")
			B(0, SW + 38, cz, 10, 3, 10, STONEC, M.Limestone, "ClockTop")
			B(0, SW + 41, cz, 6, 4, 6, GREEN, M.Metal, "ClockCap")
			B(0, SW + 45, cz, 2, 6, 2, DGREEN, M.Metal, "ClockSpire")
			for _, s in ipairs({ -1, 1 }) do
				local face = place(f, CFrame.new(0, SW + 30, cz + s * 4.7), 7.4, 7.4, 0.3, RGB(240, 238, 226), M.SmoothPlastic, "ClockFace")
				face.CanCollide = false
				local h1 = place(f, CFrame.new(0, SW + 31.2, cz + s * 4.95), 0.35, 3.0, 0.2, DARK, M.Metal, "ClockHand")
				local h2 = place(f, CFrame.new(1.0, SW + 30.1, cz + s * 4.95) * CFrame.Angles(0, 0, math.rad(-80)), 0.35, 2.4, 0.2, DARK, M.Metal, "ClockHand")
				h1.CanCollide, h2.CanCollide = false, false
			end
			hang(0, SW + 34, cz + 6, WARM, 40, 0.8)
		end
		for _, x in ipairs({ -60, -100, -140, 60, 100, 140 }) do
			B(x, SW, 162, 7, 0.9, 7, STONE2, M.Concrete, "Planter")
			tree(x, 162, 1.1)
			B(x, SW, 200, 7, 0.9, 7, STONE2, M.Concrete, "Planter")
			tree(x, 200, 1.0)
			bench(x, 171, 0)
			bench(x, 192, 180)
		end
		for _, x in ipairs({ -25, 25 }) do bench(x, 160, 180) pole(f, x, 168, true) end
		for _, x in ipairs({ -200, -170, 170, 200 }) do
			car(f, x, 190, 90, RGB(228, 178, 30))
			car(f, x, 176, 90, RGB(228, 178, 30))
		end
		kiosk(-60, 218, 180, RGB(120, 60, 40)) kiosk(60, 218, 180, RGB(60, 70, 110))
		for x = -92, 92, 8 do
			if math.abs(x) > 14 then B(x, SW, 153.5, 1.1, 2.6, 1.1, STEELC, M.Metal, "Bollard") end
		end
		B(0, SW, 152.4, 204, 0.5, 5, STONE2, M.Granite, "Steps")
		B(0, SW, 150.8, 204, 0.9, 3, STONE2, M.Granite, "Steps")
		-- flanking hotel + arcade (enterable, stairs to the roof)
		building(-212, 104, 156, 108, 3, STONEM, { doorsS = { -212 }, doorsN = { -250, -172 }, pane = true, spacing = 14 })
		building(212, 104, 156, 108, 3, BRICKM, { doorsS = { 212 }, doorsN = { 172, 250 }, pane = true, spacing = 14 })
		signboard(f, Vector3.new(-212, 39, 158.7), Vector3.new(30, 4, 0.4), Enum.NormalId.Back, "GRAND HOTEL", RGB(255, 214, 120))
		signboard(f, Vector3.new(212, 39, 158.7), Vector3.new(30, 4, 0.4), Enum.NormalId.Back, "POST & PARCELS", RGB(255, 120, 90))
		-- the two service lanes between the hall and the wings
		for _, s in ipairs({ -1, 1 }) do
			dumpster(s * 112, 120, 0) dumpster(s * 112, 70, 0)
			crate(s * 112, SW, 96, 4, 4, 4) crate(s * 112, SW + 4, 96, 3, 3, 3)
			pole(f, s * 112, 140, true) pole(f, s * 112, 50, true)
			truck(f, s * 120, 100, 0, RGB(64, 70, 58))
		end

		-- ============================ GRAND CONCOURSE (hall): x -100..100, z 30..150, 34 high ============================
		do
			local HH, HT = 34, 2
			local frontOps = {}
			for _, c in ipairs({ -60, 0, 60 }) do table.insert(frontOps, { c = c, w = 22, y0 = 0, y1 = 17 }) end
			for _, c in ipairs({ -90, -30, 30, 90 }) do table.insert(frontOps, { c = c, w = 6, y0 = 6, y1 = 24, pane = true }) end
			wallRun("x", 150 - HT / 2, -100, 100, SW, HH, HT, STONEC, M.Limestone, frontOps, "HallFront")
			local backOps = {}
			for _, c in ipairs({ -80, -40, 0, 40, 80 }) do table.insert(backOps, { c = c, w = 22, y0 = 0, y1 = 15 }) end
			wallRun("x", 30 + HT / 2, -100, 100, SW, HH, HT, STONEC, M.Limestone, backOps, "HallBack")
			for _, s in ipairs({ -1, 1 }) do
				local ops = { { c = 60, w = 8, y0 = 0, y1 = 10 }, { c = 122, w = 8, y0 = 0, y1 = 10 } }
				for _, c in ipairs({ 44, 78, 100, 140 }) do table.insert(ops, { c = c, w = 5, y0 = 8, y1 = 24, pane = true }) end
				wallRun("z", s * (100 - HT / 2), 32, 148, SW, HH, HT, STONEC, M.Limestone, ops, "HallSide")
			end
			-- roof: two flat decks with a glazed lantern strip down the middle
			B(-60, SW + HH, 90, 80, 1.6, 120, RGB(86, 92, 96), M.Metal, "HallRoof")
			B(60, SW + HH, 90, 80, 1.6, 120, RGB(86, 92, 96), M.Metal, "HallRoof")
			glassPart(0, SW + HH, 90, 40, 0.6, 120, 0.4, RGB(150, 190, 206))
			for z = 30, 150, 20 do B(0, SW + HH + 0.6, z, 40, 0.5, 1.2, DGREEN, M.Metal, "LanternRib") end
			B(-20, SW + HH + 0.6, 90, 1.2, 0.5, 120, DGREEN, M.Metal, "LanternRib")
			B(20, SW + HH + 0.6, 90, 1.2, 0.5, 120, DGREEN, M.Metal, "LanternRib")
			for z = 40, 140, 20 do B(0, SW + HH - 3, z, 200, 1.4, 1.6, GREEN, M.Metal, "RoofBeam") end
			-- pilasters on the facade
			for _, x in ipairs({ -100, -30, 30, 100 }) do B(x, SW, 151.4, 3, HH, 2, STONE2, M.Limestone, "Pilaster") end
			B(0, SW + 19, 151.6, 200, 2, 2.4, STONE2, M.Limestone, "FacadeBand")
			B(0, SW + HH, 151.2, 206, 2.2, 3.6, STONE2, M.Granite, "FacadeCornice")
			signboard(f, Vector3.new(0, SW + 27, 151.2), Vector3.new(46, 5.4, 0.5), Enum.NormalId.Back, "TERMINUS CENTRAL", RGB(255, 226, 150))
			signboard(f, Vector3.new(0, SW + 24, 31.2), Vector3.new(60, 6, 0.5), Enum.NormalId.Back, "DEPARTURES  >  PLATFORMS 1-4", RGB(255, 200, 80))
			-- two rows of great pillars, with cover blocks at their feet
			for _, x in ipairs({ -52, 52 }) do
				for z = 50, 130, 20 do
					column(x, z, HH - 1, 4, STONEC, M.Limestone)
					B(x, SW, z, 6, 1.2, 6, STONE2, M.Granite, "PillarBase")
				end
			end
			for _, x in ipairs({ -52, 52 }) do for z = 60, 120, 20 do hang(x, 24, z, WARM, 60, 0.7) end end
			hang(0, 28, 90, WARM, 80, 1)
			-- central ticket office (hollow, two doors, a counter wall with windows) + a big departures board column
			do
				local x0, x1, z0, z1 = -18, 18, 80, 112
				wallRun("x", z0 + 0.6, x0, x1, SW, 9, 1.2, RGB(120, 84, 56), M.WoodPlanks, { { c = 0, w = 6, y0 = 0, y1 = 7 } }, "Office")
				wallRun("x", z1 - 0.6, x0, x1, SW, 9, 1.2, RGB(120, 84, 56), M.WoodPlanks, { { c = -10, w = 5, y0 = 3.4, y1 = 7, pane = true }, { c = 0, w = 5, y0 = 3.4, y1 = 7, pane = true }, { c = 10, w = 5, y0 = 3.4, y1 = 7, pane = true } }, "Office")
				wallRun("z", x0 + 0.6, z0 + 1.2, z1 - 1.2, SW, 9, 1.2, RGB(120, 84, 56), M.WoodPlanks, { { c = 96, w = 5, y0 = 3.4, y1 = 7, pane = true } }, "Office")
				wallRun("z", x1 - 0.6, z0 + 1.2, z1 - 1.2, SW, 9, 1.2, RGB(120, 84, 56), M.WoodPlanks, { { c = 96, w = 5, y0 = 0, y1 = 7 } }, "Office")
				B(0, SW + 9, 96, 38, 1, 34, RGB(70, 52, 38), M.Wood, "OfficeRoof")
				B(0, SW + 10, 96, 10, 7, 1, STONEC, M.SmoothPlastic, "ClockHousing")
				for _, ox in ipairs({ -10, 10 }) do B(ox, SW, 98, 6, 3.2, 1.4, RGB(60, 44, 32), M.Wood, "OfficeDesk") end
				B(0, SW, 104, 20, 1.2, 1.2, RGB(60, 44, 32), M.Wood, "OfficeDesk")
				hang(0, 8, 96, WARM, 24, 0.9)
				signboard(f, Vector3.new(0, SW + 12, 112.6), Vector3.new(14, 4, 0.4), Enum.NormalId.Back, "TICKETS", RGB(255, 226, 150))
				signboard(f, Vector3.new(0, SW + 12, 79.4), Vector3.new(14, 4, 0.4), Enum.NormalId.Front, "TICKETS", RGB(255, 226, 150))
			end
			-- bench rows + luggage cover + flower stalls
			for _, x in ipairs({ -80, -68, 68, 80 }) do
				for z = 56, 124, 17 do bench(x, z, 90) end
			end
			for _, p in ipairs({ { -34, 62 }, { 34, 62 }, { -34, 130 }, { 34, 130 }, { -34, 90 }, { 34, 90 } }) do
				B(p[1], SW, p[2], 3, 2.2, 5, RGB(96, 60, 40), M.Wood, "LuggageCart")
				crate(p[1], SW + 2.2, p[2], 2.4, 2, 2.4)
			end
			-- mezzanine galleries along both side walls (z 84..146) reached by long stairs from the floor
			for _, s in ipairs({ -1, 1 }) do
				local x0, x1 = s == -1 and -100 or 80, s == -1 and -80 or 100
				B((x0 + x1) / 2, SW + 10, 115, 20, 1, 62, STONE2, M.Concrete, "Gallery")
				-- rail on the open edge
				B(s == -1 and -80.4 or 80.4, SW + 11, 115, 0.8, 1.4, 62, STONEC, M.Limestone, "GalleryRail")
				B(s * 90, SW + 11, 145.4, 20, 1.4, 0.8, STONEC, M.Limestone, "GalleryRail")
				B(s * 90, SW + 11, 84.4, 20, 1.4, 0.8, STONEC, M.Limestone, "GalleryRail")
				-- shop fronts under the gallery
				B(s * 90, SW, 142, 14, 4, 3, RGB(150, 100, 60), M.WoodPlanks, "ShopCounter")
				B(s * 90, SW, 100, 3, 3.4, 22, RGB(120, 84, 56), M.WoodPlanks, "ShopCounter")
				stairs(Vector3.new(s * 90, SW, 42), Vector3.new(s * 90, SW + 11, 84), 6, STONE2)
				hang(s * 90, 8, 100, WARM, 40, 0.8)
				hang(s * 90, 8, 130, WARM, 40, 0.8)
			end
			-- gate posts at each platform opening
			for _, c in ipairs({ -80, -40, 0, 40, 80 }) do
				B(c - 8.5, SW, 36, 1.6, 3.4, 5, STEELC, M.Metal, "GatePost")
				B(c + 8.5, SW, 36, 1.6, 3.4, 5, STEELC, M.Metal, "GatePost")
				glassPart(c - 4.2, SW, 36, 0.2, 3, 5, 0.6).CanCollide = true
				glassPart(c + 4.2, SW, 36, 0.2, 3, 5, 0.6).CanCollide = true
			end
		end

		-- ============================ CONCOURSE shops (z -6 .. 28) ============================
		building(-182, 11, 74, 34, 2, STONEM, { doorsS = { -182 }, doorsN = { -200, -164 }, spacing = 14 })
		building(182, 11, 74, 34, 2, BRICKM, { doorsS = { 182 }, doorsN = { 164, 200 }, spacing = 14 })
		signboard(f, Vector3.new(-182, 21, 28.6), Vector3.new(24, 3, 0.4), Enum.NormalId.Back, "STATION SHOPS", RGB(255, 226, 150))
		signboard(f, Vector3.new(182, 21, 28.6), Vector3.new(24, 3, 0.4), Enum.NormalId.Back, "BUFFET CAR CAFE", RGB(255, 150, 110))
		for _, p in ipairs({ { -106, 14 }, { -106, 4 }, { 106, 14 }, { 106, 4 } }) do kiosk(p[1], p[2], 90) end
		for _, x in ipairs({ -118, -62, 62, 118 }) do pole(f, x, 20, true) bench(x + 6, 6, 0) end
		for _, x in ipairs({ -138, 138 }) do B(x, SW, 22, 4, 3, 4, STONE2, M.Concrete, "Planter") tree(x, 22, 0.8) end

		-- ============================ PLATFORMS, TRACKS, TRAINS, CANOPIES ============================
		local PL = { { -14, 16 }, { -41, 18 }, { -79, 18 }, { -106, 16 } }       -- platform centre z, width
		local TR = { -27, -55, -65, -93, -119, -129 }                              -- track centre z
		local PH = 1.7
		for i, p in ipairs(PL) do
			local zc, w = p[1], p[2]
			B(0, SW, zc, 520, PH, w, PLATC, M.Concrete, "Platform")
			B(0, SW + PH, zc - w / 2 + 0.6, 520, 0.05, 0.9, YELC, M.SmoothPlastic, "Paint").CanCollide = false
			B(0, SW + PH, zc + w / 2 - 0.6, 520, 0.05, 0.9, YELC, M.SmoothPlastic, "Paint").CanCollide = false
			for _, s in ipairs({ -1, 1 }) do
				ramp(f, Vector3.new(s * 276, SW, zc), Vector3.new(s * 260, SW + PH, zc), math.min(w - 2, 8), PLATC, M.Concrete)
			end
			-- platform furniture
			for x = -230, 230, 46 do
				bench(x + 8, zc + (i % 2 == 0 and 3 or -3), 0)
				if (x / 46) % 2 == 0 then
					B(x - 10, SW + PH, zc + 5, 2.4, 3.6, 1.6, RGB(150, 40, 36), M.Metal, "Vending")
					B(x - 14, SW + PH, zc + 5, 1.4, 2.2, 1.4, RGB(60, 70, 66), M.Metal, "Bin")
				end
			end
			-- station name boards hanging on the platform edge
			for _, x in ipairs({ -150, 0, 150 }) do
				signboard(f, Vector3.new(x, SW + PH + 10, zc), Vector3.new(0.5, 3.2, 12), Enum.NormalId.Left, "PLATFORM " .. i, RGB(240, 240, 235))
				signboard(f, Vector3.new(x + 0.05, SW + PH + 10, zc), Vector3.new(0.5, 3.2, 12), Enum.NormalId.Right, "PLATFORM " .. i, RGB(240, 240, 235))
				B(x, SW + PH + 11.6, zc - 3, 0.3, 1.4, 0.3, STEELC, M.Metal, "Hanger").CanCollide = false
				B(x, SW + PH + 11.6, zc + 3, 0.3, 1.4, 0.3, STEELC, M.Metal, "Hanger").CanCollide = false
			end
		end
		local function track(zc)
			B(0, 0.1, zc, 560, 0.3, 9.8, GRAV, M.Pebble, "Ballast")
			for _, s in ipairs({ -2.35, 2.35 }) do
				local r = B(0, 0.4, zc + s, 560, 0.5, 0.35, RGB(120, 118, 114), M.Metal, "Rail")
				r.CanCollide, r.CastShadow = false, false
			end
			for x = -276, 276, 8 do
				local t = B(x, 0.4, zc, 1.3, 0.22, 7.4, RGB(72, 56, 42), M.Wood, "Sleeper")
				t.CanCollide, t.CanQuery, t.CastShadow = false, false, false
			end
		end
		for _, z in ipairs(TR) do track(z) end
		-- carriage: solid for cover, or hollow and enterable
		local function carriage(cx, zc, hollow, col, stripe)
			local y0 = SW + 1.2
			for _, bx in ipairs({ -8.5, 8.5 }) do
				B(cx + bx, SW + 0.4, zc, 5.4, 0.8, 5.6, DARK, M.Metal, "Bogie").CanCollide = false
			end
			if not hollow then
				B(cx, y0, zc, 24, 8.6, 8.2, col, M.Metal, "Train")
				B(cx, y0 + 8.6, zc, 23, 0.6, 7, STEELC, M.Metal, "TrainRoof")
				for _, s in ipairs({ -1, 1 }) do
					local g = B(cx, y0 + 4.4, zc + s * 4.12, 21, 2.6, 0.2, RGB(26, 36, 48), M.Glass, "TrainGlass")
					g.CanCollide, g.CanQuery = false, false
					local st = B(cx, y0 + 2.4, zc + s * 4.12, 24, 0.7, 0.2, stripe, M.SmoothPlastic, "TrainStripe")
					st.CanCollide, st.CanQuery = false, false
					for _, dx in ipairs({ -7, 7 }) do
						local d = B(cx + dx, y0, zc + s * 4.12, 3.4, 7, 0.25, col:Lerp(DARK, 0.35), M.Metal, "TrainDoor")
						d.CanCollide, d.CanQuery = false, false
					end
				end
				return
			end
			-- hollow
			local fl = y0 + 0.5
			B(cx, y0, zc, 24, 0.5, 8.2, RGB(60, 56, 52), M.Metal, "TrainFloor")
			B(cx, fl + 8, zc, 24, 0.5, 8.4, col, M.Metal, "TrainRoof")
			for _, s in ipairs({ -1, 1 }) do
				local z = zc + s * 4.0
				local function seg(x0, x1, y0_, y1_, name)
					B((x0 + x1) / 2 + cx, fl + y0_, z, x1 - x0, y1_ - y0_, 0.35, col, M.Metal, name or "TrainWall")
				end
				seg(-12, -9.5, 0, 8) seg(9.5, 12, 0, 8)
				seg(-4.5, 4.5, 0, 2.4) seg(-4.5, 4.5, 5.6, 8) seg(-0.6, 0.6, 2.4, 5.6)
				seg(-9.5, -4.5, 6.6, 8) seg(4.5, 9.5, 6.6, 8)
				local st = B(cx, fl + 1.3, z + s * 0.2, 24, 0.5, 0.1, stripe, M.SmoothPlastic, "TrainStripe")
				st.CanCollide, st.CanQuery = false, false
				-- seats along the wall
				B(cx, fl, zc + s * 2.9, 7, 1.1, 1.3, RGB(60, 60, 110), M.Fabric, "Seat")
			end
			for _, s in ipairs({ -1, 1 }) do
				local x = cx + s * 11.85
				B(x, fl, zc - 3.1, 0.3, 8, 2, col, M.Metal, "TrainEnd")
				B(x, fl, zc + 3.1, 0.3, 8, 2, col, M.Metal, "TrainEnd")
				B(x, fl + 6.2, zc, 0.3, 1.8, 4.2, col, M.Metal, "TrainEnd")
			end
			lamp(f, Vector3.new(cx, fl + 6.5, zc), RGB(255, 236, 190), 22, 0.8)
		end
		-- passenger train on track 1 (centre -27): gaps left for crossing
		local PASS_COLS = { RGB(150, 40, 40), RGB(44, 70, 120), RGB(60, 90, 70) }
		local t1 = { -226, -200.5, -175, -62, -36.5, -11, 14.5, 40, 150, 175.5, 201 }
		for i, x in ipairs(t1) do carriage(x, TR[1], i % 3 == 2, PASS_COLS[1], YELC) end
		-- long freight train on track 2 (centre -55): boxcars with several gaps
		local FRC = { RGB(110, 72, 52), RGB(70, 80, 64), RGB(100, 96, 84), RGB(86, 56, 48) }
		for i, x in ipairs({ -250, -224, -198, -172, -96, -70, -44, 30, 56, 82, 108, 134, 160, 186, 212 }) do
			carriage(x, TR[2], false, FRC[(i % 4) + 1], RGB(190, 190, 180))
		end
		-- track 3 (centre -65): kept open as a lane, with one stranded carriage and a few crates
		carriage(-150, TR[3], true, PASS_COLS[2], RGB(220, 220, 214))
		carriage(100, TR[3], false, RGB(70, 70, 76), RGB(210, 180, 40))
		-- passenger train on track 4 (centre -93)
		for i, x in ipairs({ -212, -186.5, -161, -135.5, -45, -19.5, 6, 31.5, 57, 82.5, 190, 215.5 }) do
			carriage(x, TR[4], i % 3 == 0, PASS_COLS[2], RGB(230, 230, 222))
		end
		-- locomotives at the heads of the passenger trains
		local function loco(x, zc, col, dir)
			B(x, SW + 0.9, zc, 24, 9.4, 8.4, col, M.Metal, "Loco")
			B(x - dir * 4, SW + 10.3, zc, 14, 3.8, 7.4, col:Lerp(DARK, 0.2), M.Metal, "LocoCab")
			local g = B(x - dir * 4, SW + 11.2, zc, 14.2, 1.8, 7.5, RGB(26, 36, 48), M.Glass, "TrainGlass")
			g.CanCollide, g.CanQuery = false, false
			B(x + dir * 11.6, SW + 4.2, zc, 0.6, 1.2, 5, YELC, M.Neon, "Headlight").CanCollide = false
			B(x, SW + 0.9, zc + 4.25, 24, 0.7, 0.2, YELC, M.SmoothPlastic, "Paint").CanCollide = false
			B(x, SW + 0.9, zc - 4.25, 24, 0.7, 0.2, YELC, M.SmoothPlastic, "Paint").CanCollide = false
			B(x + dir * 8, SW + 14.1, zc, 5, 1.4, 4, DARK, M.Metal, "LocoFan")
			for _, bx in ipairs({ -8, 8 }) do B(x + bx, SW + 0.2, zc, 5.4, 0.8, 5.6, DARK, M.Metal, "Bogie").CanCollide = false end
		end
		loco(-258, TR[1], RGB(150, 40, 40), -1)
		loco(-258, TR[4], RGB(44, 70, 120), -1)
		loco(238, TR[3], RGB(70, 70, 76), 1)
		-- long freight on track 5 (tanks) and an old wagon line on 6
		for i, x in ipairs({ -240, -210, -180, -150, -120, 70, 100, 130, 160, 190, 220 }) do
			if x % 2 == 0 then
				B(x, SW + 0.4, TR[5], 24, 1, 6, DARK, M.Metal, "TankChassis")
				local tk = B(x, SW + 1.4, TR[5], 22, 6.4, 6.4, (i % 2 == 0) and RGB(70, 76, 82) or RGB(104, 64, 46), M.Metal, "TankCar")
				tk.Shape = Enum.PartType.Cylinder
				tk.CFrame = CFrame.new(x, SW + 4.6, TR[5]) * CFrame.Angles(0, 0, 0)
				tk.Size = Vector3.new(22, 6.4, 6.4)
			end
		end
		for _, x in ipairs({ -200, -100, 150 }) do carriage(x, TR[6], false, FRC[(x % 3) + 1], RGB(190, 190, 180)) end
		-- buffer stops
		for _, z in ipairs(TR) do
			for _, s in ipairs({ -1, 1 }) do
				B(s * 281, SW, z, 2.4, 2.6, 7.4, RUSTC, M.Metal, "BufferStop")
			end
		end

		-- canopies: gaps left where the footbridge stairs come down
		local BX = { -130, 130 }
		local function canopy(z0, z1, segs)
			local zc, zl = (z0 + z1) / 2, z1 - z0
			for _, sg in ipairs(segs) do
				local cx, len = (sg[1] + sg[2]) / 2, sg[2] - sg[1]
				B(cx, SW + 14.3, zc, len, 0.9, zl, RGB(70, 84, 80), M.Metal, "CanopyRoof")
				B(cx, SW + 15.2, zc, len, 0.4, zl * 0.5, RGB(150, 190, 206), M.Glass, "CanopySky").CanCollide = false
				for z = z0 + 2, z1 - 1, 14 do B(cx, SW + 13.6, z, len, 0.8, 0.9, GREEN, M.Metal, "CanopyRib") end
				B(cx, SW + 13.2, z0 + 0.6, len, 1.4, 0.5, GREEN, M.Metal, "CanopyFascia")
				B(cx, SW + 13.2, z1 - 0.6, len, 1.4, 0.5, GREEN, M.Metal, "CanopyFascia")
			end
		end
		canopy(-50, -6, { { -212, -138 }, { -100, 100 }, { 138, 212 } })
		canopy(-114, -70, { { -212, -138 }, { -100, 100 }, { 138, 212 } })
		for _, pz in ipairs({ -14, -41, -79, -106 }) do
			for x = -204, 204, 34 do
				if not (x > -140 and x < -98) and not (x > 98 and x < 140) then
					column(x, pz, 13, 1.6, GREEN, M.Metal, SW + PH)
					if (x // 34) % 2 == 0 then hang(x, SW + 12.5, pz + 3, WARM, 42, 0.8) end
				end
			end
		end
		-- overhead signal gantries across all the tracks
		for _, gx in ipairs({ -60, 60, 190, -200 }) do
			B(gx, SW, -4, 1.6, 21, 1.6, STEELC, M.Metal, "GantryLeg")
			B(gx, SW, -136, 1.6, 21, 1.6, STEELC, M.Metal, "GantryLeg")
			local bm = B(gx, SW + 21, -70, 1.4, 1.6, 134, STEELC, M.Metal, "GantryBeam")
			bm.CanCollide = false
			for _, tz in ipairs(TR) do
				local sg = B(gx, SW + 18.4, tz, 1.2, 2.6, 1.2, DARK, M.Metal, "Signal")
				sg.CanCollide = false
				local lt = B(gx + 0.7, SW + 19.2, tz, 0.3, 0.7, 0.7, (tz % 20 == 0 or gx > 100) and RGB(255, 60, 40) or RGB(60, 230, 90), M.Neon, "SignalLight")
				lt.CanCollide = false
			end
		end

		-- ============================ FOOTBRIDGES (glazed, deck y 19) ============================
		for bi, X in ipairs(BX) do
			local dirx = (bi == 1) and 1 or -1     -- stairs lead towards the middle of the station
			local DY = SW + 19
			B(X, DY - 0.8, -60, 10, 0.9, 116, RGB(80, 84, 88), M.Concrete, "BridgeDeck")
			B(X, DY + 3.5, -60, 12, 0.7, 118, GREEN, M.Metal, "BridgeRoof")
			-- glazed walls (open doorways where the stairs meet the deck)
			local ops = {}
			for _, pz in ipairs({ -14, -41, -79, -106 }) do table.insert(ops, { c = pz, w = 6, y0 = 0, y1 = 3.4 }) end
			wallRun("z", X - dirx * 5.1, -118, -2, DY, 3.5, 0.3, RGB(120, 160, 180), M.Glass, {}, "BGlass")
			wallRun("z", X + dirx * 5.1, -118, -2, DY, 3.5, 0.3, RGB(120, 160, 180), M.Glass, ops, "BGlass")
			for z = -116, -4, 16 do
				B(X - 5.1, DY, z, 0.5, 3.5, 0.5, GREEN, M.Metal, "BridgeMullion")
				B(X + 5.1, DY, z, 0.5, 3.5, 0.5, GREEN, M.Metal, "BridgeMullion")
				if z % 32 == 0 then hang(X, DY + 3, z, WARM, 28, 0.8) end
			end
			-- stairs from each platform up to the deck + support piers on the platforms
			for _, pz in ipairs({ -14, -41, -79, -106 }) do
				stairs(Vector3.new(X + dirx * 35, SW + PH, pz), Vector3.new(X + dirx * 5, DY, pz), 5.4, RGB(110, 110, 112))
				B(X, SW + PH, pz + 4.5, 2.4, DY - SW - PH - 0.9, 2.4, STEELC, M.Concrete, "BridgePier")
				B(X, SW + PH, pz - 4.5, 2.4, DY - SW - PH - 0.9, 2.4, STEELC, M.Concrete, "BridgePier")
			end
			-- long stairs down to the concourse (south) and to the freight yard (north)
			stairs(Vector3.new(X, SW, 40), Vector3.new(X, DY, -2), 7, RGB(110, 110, 112))
			stairs(Vector3.new(X, SW, -160), Vector3.new(X, DY, -118), 7, RGB(110, 110, 112))
			signboard(f, Vector3.new(X, DY + 5.2, -1.6), Vector3.new(10, 1.8, 0.4), Enum.NormalId.Back, "PLATFORMS  >", RGB(255, 226, 150))
		end
		for _, p in ipairs(f:GetChildren()) do
			if p.Name == "BGlass" then p.Transparency, p.CanQuery, p.CastShadow = 0.5, false, false end
		end

		-- ============================ FREIGHT YARD (z -135 .. -215) ============================
		-- warehouse (west): roller doors facing the tracks and the yard, stair to the roof, racking inside
		do
			local x0, x1, z0, z1, h = -285, -140, -213, -148, 24
			local dS = opsRow(x0, x1, { -255, -200, -160 }, 16, 13, 17, 21, 20, 6)
			local dN = opsRow(x0, x1, { -250, -195 }, 16, 13, 17, 21, 20, 6)
			local dE = opsRow(z0, z1, { -180 }, 14, 12, 17, 21, 18, 6)
			local dW = opsRow(z0, z1, {}, 14, 12, 17, 21, 18, 6)
			shedWalls(x0, x1, z0, z1, h, RGB(132, 108, 92), M.Brick, dS, dN, dW, dE)
			B((x0 + x1) / 2, SW + h, (z0 + z1) / 2, x1 - x0, 1.2, z1 - z0, RGB(78, 82, 86), M.Metal, "WarehouseRoof")
			for x = x0 + 8, x1 - 6, 14 do B(x, SW + h - 2, (z0 + z1) / 2, 1.4, 1.6, z1 - z0, GREEN, M.Metal, "TrussBeam") end
			-- racking aisles
			for z = z0 + 10, z1 - 8, 15 do
				for x = x0 + 14, x1 - 26, 30 do
					if not (z > -185 and z < -165 and x > -215) then
						B(x, SW, z, 20, 9, 2.4, RGB(70, 84, 120), M.Metal, "Racking")
						B(x, SW, z + 5, 20, 5, 2.4, RGB(70, 84, 120), M.Metal, "Racking")
						crate(x - 6, SW + 9, z, 3, 2.6, 2)
					end
				end
			end
			-- first floor catwalk / office
			B(-160, SW + 11, -180, 24, 0.8, 62, RGB(90, 94, 98), M.DiamondPlate, "Catwalk")
			B(-171.5, SW + 11.8, -180, 0.6, 1.4, 62, STEELC, M.Metal, "CatwalkRail")
			stairs(Vector3.new(-202, SW, -175), Vector3.new(-172, SW + 11.8, -175), 5, RGB(90, 94, 98))
			stairs(Vector3.new(x1 + 40, SW, -170), Vector3.new(x1 - 2, SW + h + 1.2, -170), 6, RGB(100, 100, 102))
			B((x0 + x1) / 2, SW + h + 1.2, z0 + 0.7, x1 - x0, 1.4, 1.4, RGB(110, 100, 90), M.Brick, "RoofParapet")
			B((x0 + x1) / 2, SW + h + 1.2, z1 - 0.7, x1 - x0, 1.4, 1.4, RGB(110, 100, 90), M.Brick, "RoofParapet")
			B(x0 + 0.7, SW + h + 1.2, (z0 + z1) / 2, 1.4, 1.4, z1 - z0, RGB(110, 100, 90), M.Brick, "RoofParapet")
			B(x1 - 0.7, SW + h + 1.2, (z0 + z1) / 2, 1.4, 1.4, z1 - z0, RGB(110, 100, 90), M.Brick, "RoofParapet")
			for _, rx in ipairs({ -260, -230, -190 }) do B(rx, SW + h + 1.2, -185, 8, 3, 6, RGB(104, 108, 112), M.Metal, "ACUnit") end
			hang(-210, 20, -180, WARM, 70, 0.9) hang(-250, 20, -165, WARM, 60, 0.8) hang(-170, 20, -195, WARM, 60, 0.8)
			signboard(f, Vector3.new(-212, SW + 19.4, z1 + 0.15), Vector3.new(34, 3.4, 0.4), Enum.NormalId.Back, "FREIGHT WAREHOUSE 4", RGB(255, 190, 70))
		end
		-- locomotive depot (east): two inner tracks, a loco in the shed, high crane rail
		do
			local x0, x1, z0, z1, h = 140, 285, -213, -148, 26
			local dS = opsRow(x0, x1, { 190, 245 }, 22, 16, 19, 23, 22, 6)
			local dN = opsRow(x0, x1, { 190, 245 }, 22, 16, 19, 23, 22, 6)
			local dE = opsRow(z0, z1, {}, 14, 12, 19, 23, 18, 6)
			local dW = opsRow(z0, z1, { -180 }, 14, 12, 19, 23, 18, 6)
			shedWalls(x0, x1, z0, z1, h, RGB(96, 100, 104), M.Concrete, dS, dN, dW, dE)
			B((x0 + x1) / 2, SW + h, (z0 + z1) / 2, x1 - x0, 1.2, z1 - z0, RGB(70, 74, 80), M.Metal, "DepotRoof")
			for x = x0 + 8, x1 - 6, 16 do B(x, SW + h - 2.2, (z0 + z1) / 2, 1.6, 1.8, z1 - z0, DGREEN, M.Metal, "TrussBeam") end
			for _, tx in ipairs({ 190, 245 }) do
				B(tx, 0.1, (z0 + z1) / 2, 6, 0.3, z1 - z0 - 3, GRAV, M.Pebble, "Ballast")
				for _, s in ipairs({ -1.6, 1.6 }) do B(tx + s, 0.4, (z0 + z1) / 2, 0.35, 0.5, z1 - z0 - 3, RGB(120, 118, 114), M.Metal, "Rail").CanCollide = false end
				-- maintenance pit
				B(tx + 8, SW, (z0 + z1) / 2, 6, 0.4, 30, DARK, M.Metal, "PitCover")
			end
			-- locomotive standing in the depot (long axis along z)
			do
				local zc = -180
				B(190, SW + 0.9, zc, 8.4, 9.4, 24, RGB(210, 170, 40), M.Metal, "Loco")
				B(190, SW + 10.3, zc + 4, 7.4, 3.8, 14, RGB(180, 140, 30), M.Metal, "LocoCab")
				for _, dz in ipairs({ -8, 8 }) do B(190, SW + 0.2, zc + dz, 5.6, 0.8, 5.4, DARK, M.Metal, "Bogie").CanCollide = false end
			end
			-- overhead crane (runs on the long walls)
			B((x0 + x1) / 2, SW + 18, -180, x1 - x0 - 4, 1.4, 1.6, STEELC, M.Metal, "CraneRail").CanCollide = false
			B(215, SW + 17, -180, 2, 2, 62, DARK, M.Metal, "CraneBridge").CanCollide = false
			for _, z in ipairs({ -165, -195 }) do for _, x in ipairs({ 160, 215, 270 }) do crate(x, SW, z, 4, 4, 4) end end
			for _, p in ipairs({ { 160, -158 }, { 270, -205 } }) do B(p[1], SW, p[2], 6, 5, 6, RUSTC, M.CorrodedMetal, "FuelDrums") end
			stairs(Vector3.new(x0 - 40, SW, -170), Vector3.new(x0 + 2, SW + h + 1.2, -170), 6, RGB(100, 100, 102))
			B((x0 + x1) / 2, SW + h + 1.2, z0 + 0.7, x1 - x0, 1.4, 1.4, RGB(80, 84, 88), M.Concrete, "RoofParapet")
			B((x0 + x1) / 2, SW + h + 1.2, z1 - 0.7, x1 - x0, 1.4, 1.4, RGB(80, 84, 88), M.Concrete, "RoofParapet")
			B(x0 + 0.7, SW + h + 1.2, (z0 + z1) / 2, 1.4, 1.4, z1 - z0, RGB(80, 84, 88), M.Concrete, "RoofParapet")
			B(x1 - 0.7, SW + h + 1.2, (z0 + z1) / 2, 1.4, 1.4, z1 - z0, RGB(80, 84, 88), M.Concrete, "RoofParapet")
			hang(190, 22, -195, WARM, 70, 0.9) hang(245, 22, -165, WARM, 70, 0.9) hang(215, 22, -180, WARM, 60, 0.9)
			signboard(f, Vector3.new(212, SW + 23.4, z1 + 0.15), Vector3.new(34, 3.4, 0.4), Enum.NormalId.Back, "LOCOMOTIVE DEPOT", RGB(255, 190, 70))
		end
		-- signal box tower in the middle of the yard (enterable, four floors with a sniper roof)
		building(0, -150, 30, 24, 4, BRICKM, { doorsS = { -8 }, doorsN = { 8 }, pane = true, spacing = 10 })
		signboard(f, Vector3.new(0, 52, -137.4), Vector3.new(14, 2.4, 0.4), Enum.NormalId.Back, "SIGNAL BOX 7", RGB(255, 226, 150))
		-- container maze
		local CCOL = { RGB(150, 52, 42), RGB(46, 84, 128), RGB(52, 104, 70), RGB(190, 150, 40), RGB(100, 100, 106), RGB(120, 70, 40), RGB(150, 100, 40) }
		local stackN = 0
		for gz = -180, -210, -15 do
			for gx = -112, 112, 28 do
				if math.abs(gx) > 12 and not (math.abs(gx) > 100 and gz > -200) then
					local rot = (MapRand:NextNumber() < 0.5) and 0 or 90
					local lv = MapRand:NextInteger(0, 2)
					for l = 0, lv do
						container(f, gx + R(-1.5, 1.5), gz + R(-1.5, 1.5), rot, l, CCOL[MapRand:NextInteger(1, #CCOL)])
					end
					stackN += 1
					if lv >= 1 and stackN % 3 == 0 then
						ramp(f, Vector3.new(gx - 20, 0.1, gz), Vector3.new(gx - 6.5, 6.1, gz), 5, RGB(86, 86, 90), M.Metal)
					end
				end
			end
		end
		for _, p in ipairs({ { -60, -158 }, { 60, -158 }, { -90, -195 }, { 90, -195 }, { 0, -205 } }) do
			truck(f, p[1], p[2], 90 * (stackN % 2), RGB(70, 76, 62))
			stackN += 1
		end
		for _, p in ipairs({ { -30, -142 }, { 30, -142 }, { -70, -176 }, { 70, -176 } }) do
			jersey(f, p[1], p[2], 90) sandbags(f, p[1] + 6, p[2] + 4, 0)
		end
		-- water tower + fuel tanks + rail cranes
		do
			local wx, wz = 120, -143
			for _, o in ipairs({ { -3, -3 }, { 3, -3 }, { -3, 3 }, { 3, 3 } }) do B(wx + o[1], SW, wz + o[2], 0.9, 22, 0.9, STEELC, M.Metal, "TowerLeg") end
			B(wx, SW + 22, wz, 10, 1, 10, STEELC, M.Metal, "TowerDeck")
			cyl(wx, SW + 23, wz, 9, 8, RUSTC, M.CorrodedMetal, "WaterTank")
			B(wx, SW + 31, wz, 8.4, 1.4, 8.4, RUSTC, M.CorrodedMetal, "TankCap")
			for _, p in ipairs({ { -120, -140 }, { -90, -140 } }) do
				cyl(p[1], SW, p[2], 16, 12, RGB(110, 114, 116), M.Metal, "FuelTank")
				B(p[1], SW + 12, p[2], 6, 1, 6, STEELC, M.Metal, "TankLid")
			end
		end

		-- ============================ BRAVO LOGISTICS YARD (z -218 .. -295) ============================
		-- gantry crane
		do
			local cz = -250
			for _, x in ipairs({ -62, 62 }) do
				for _, dz in ipairs({ -9, 9 }) do B(x, SW, cz + dz, 4, 36, 4, RGB(214, 150, 36), M.Metal, "CraneLeg") end
				B(x, SW + 36, cz, 4, 3, 22, RGB(214, 150, 36), M.Metal, "CraneSill")
			end
			B(0, SW + 39, cz, 132, 3.4, 6, RGB(214, 150, 36), M.Metal, "CraneBeam")
			B(-20, SW + 30, cz, 6, 8, 6, DARK, M.Metal, "CraneCab")
			B(-20, SW + 18, cz, 0.5, 12, 0.5, DARK, M.Metal, "CraneCable").CanCollide = false
			stairs(Vector3.new(-120, SW, cz + 30), Vector3.new(-62, SW + 37.5, cz - 8), 4, RGB(214, 150, 36))
		end
		for _, gx in ipairs({ -225, -175, -125, -75, 75, 125, 175, 225 }) do
			for l = 0, MapRand:NextInteger(0, 2) do
				container(f, gx + R(-1, 1), -279 + R(-1, 1), 90, l, CCOL[MapRand:NextInteger(1, #CCOL)])
			end
		end
		for gx = -200, 200, 50 do
			if math.abs(gx) > 20 and math.abs(gx) ~= 150 then
				for l = 0, MapRand:NextInteger(0, 1) do
					container(f, gx + R(-2, 2), -234 + R(-2, 2), 0, l, CCOL[MapRand:NextInteger(1, #CCOL)])
				end
			end
		end
		for _, x in ipairs({ -240, -160, 160, 240 }) do truck(f, x, -262, 0, RGB(70, 76, 62)) end
		for _, x in ipairs({ -105, -55, 55, 105 }) do pole(f, x, -228, true) pole(f, x, -290, true) end
		for _, p in ipairs({ { -130, -225 }, { 130, -225 } }) do jersey(f, p[1], p[2], 0) jersey(f, p[1] + 7, p[2], 0) end
		-- site offices (enterable) at either end
		building(-255, -255, 44, 34, 2, BRICKM, { doorsS = { -255 }, doorsN = { -255 }, spacing = 12 })
		building(255, -255, 44, 34, 2, BRICKM, { doorsS = { 255 }, doorsN = { 255 }, spacing = 12 })
		signboard(f, Vector3.new(0, 14, -293.6), Vector3.new(60, 6, 0.4), Enum.NormalId.Back, "TERMINUS FREIGHT YARD", RGB(255, 190, 70))

		return { spawns = sp, half = 300 }
	end

	MapBuilders.GRID_PG = function(f)
		MapRand = Random.new(11)
		-- HAND-BUILT MAP: if ServerStorage has a Folder or Model called "CustomMap_GRID", it replaces the generated Proving Grounds.
		-- Inside it: your own parts, plus parts named AlphaSpawn / BravoSpawn / FFASpawn marking where players start.
		-- Optional attribute "Half" on the folder = half the map width in studs (default 150): invisible walls are placed there.
		do
			local custom = game:GetService("ServerStorage"):FindFirstChild("CustomMap_GRID")
			if custom then
				local sp = { Alpha = {}, Bravo = {}, FFA = {} }
				for _, ch in ipairs(custom:GetChildren()) do
					local c = ch:Clone()
					c.Parent = f
				end
				for _, d in ipairs(f:GetDescendants()) do
					if d:IsA("BasePart") then
						d.Anchored = true
						local key = (d.Name == "AlphaSpawn" and "Alpha") or (d.Name == "BravoSpawn" and "Bravo") or (d.Name == "FFASpawn" and "FFA") or nil
						if key then
							table.insert(sp[key], Vector3.new(d.Position.X, d.Position.Y + d.Size.Y / 2, d.Position.Z))
							d.Transparency = 1
							d.CanCollide = false
							d.CanQuery = false
						end
					end
				end
				local half = custom:GetAttribute("Half") or 150
				if #sp.Alpha == 0 then for i = -2, 2 do table.insert(sp.Alpha, Vector3.new(i * 12, 3, half - 20)) end end
				if #sp.Bravo == 0 then for i = -2, 2 do table.insert(sp.Bravo, Vector3.new(i * 12, 3, -half + 20)) end end
				if #sp.FFA == 0 then for i = -3, 3 do table.insert(sp.FFA, Vector3.new(i * 15, 3, 0)) end end
				setSpawns(sp)
				return { spawns = sp, half = half }
			end
		end
		local CONC, GREY, YEL, CY = RGB(112, 112, 114), RGB(72, 76, 82), RGB(225, 165, 30), RGB(90, 150, 190)
		local sp = { Alpha = {}, Bravo = {}, FFA = {} }
		for _, z in ipairs({ -30, -10, 10, 30 }) do
			table.insert(sp.Alpha, Vector3.new(-98, 0, z))
			table.insert(sp.Bravo, Vector3.new(98, 0, z))
		end
		for _, p in ipairs({ { -45, -90 }, { 45, 90 }, { -45, 90 }, { 45, -90 }, { 0, -100 }, { 0, 100 }, { -62, 0 }, { 62, 0 },
			{ 0, -40 }, { 0, 40 }, { -26, 62 }, { 26, -62 } }) do
			table.insert(sp.FFA, Vector3.new(p[1], 0, p[2]))
		end
		setSpawns(sp)
		blk(f, 0, -2, 0, 250, 2, 250, CONC, Enum.Material.Plastic, "Floor")
		for k = -90, 90, 30 do
			blk(f, k, 0, 0, 0.5, 0.1, 220, RGB(190, 180, 120), Enum.Material.SmoothPlastic, "Paint")
			blk(f, 0, 0, k, 220, 0.1, 0.5, RGB(190, 180, 120), Enum.Material.SmoothPlastic, "Paint")
		end
		-- perimeter walls
		for _, s in ipairs({ -1, 1 }) do
			blk(f, 0, 0, s * 109, 222, 12, 2, GREY, Enum.Material.Concrete, "Wall")
			blk(f, s * 109, 0, 0, 2, 12, 222, GREY, Enum.Material.Concrete, "Wall")
			blk(f, 0, 12, s * 109, 222, 0.8, 2.6, YEL, Enum.Material.SmoothPlastic, "WallCap")
			blk(f, s * 109, 12, 0, 2.6, 0.8, 222, YEL, Enum.Material.SmoothPlastic, "WallCap")
		end
		-- central tower
		blk(f, 0, 0, 0, 36, 7, 36, GREY, Enum.Material.Concrete, "Tower")
		blk(f, 0, 7, 0, 36.6, 0.5, 36.6, YEL, Enum.Material.SmoothPlastic, "TowerEdge")
		ramp(f, Vector3.new(50, 0, 0), Vector3.new(18, 7, 0), 12, GREY, Enum.Material.Concrete)
		ramp(f, Vector3.new(-50, 0, 0), Vector3.new(-18, 7, 0), 12, GREY, Enum.Material.Concrete)
		ramp(f, Vector3.new(0, 0, 50), Vector3.new(0, 7, 18), 12, GREY, Enum.Material.Concrete)
		ramp(f, Vector3.new(0, 0, -50), Vector3.new(0, 7, -18), 12, GREY, Enum.Material.Concrete)
		for _, sx in ipairs({ -1, 1 }) do
			for _, sz in ipairs({ -1, 1 }) do
				blk(f, sx * 12, 7.5, sz * 12, 5, 3, 1.5, GREY, Enum.Material.Concrete, "TowerCover")
			end
		end
		lamp(f, Vector3.new(0, 22, 0), RGB(255, 235, 200), 70, 1.2)
		-- shoot-houses
		local function house(cx, cz, rot)
			local base = CFrame.new(cx, 0, cz) * CFrame.Angles(0, math.rad(rot), 0)
			local function seg(ox, oz, sx, sz) place(f, base * CFrame.new(ox, 5, oz), sx, 10, sz, RGB(98, 94, 86), Enum.Material.Concrete, "HouseWall") end
			seg(-7.5, -13, 9, 1.4) seg(7.5, -13, 9, 1.4)
			seg(-7.5, 13, 9, 1.4) seg(7.5, 13, 9, 1.4)
			seg(-13, -6.5, 1.4, 12) seg(-13, 7.5, 1.4, 9) seg(13, -7.5, 1.4, 9) seg(13, 6.5, 1.4, 12)
			seg(0, 0, 14, 1.4) seg(-4, -7, 1.4, 8)
			place(f, base * CFrame.new(0, 10.3, 0), 27.5, 0.8, 27.5, RGB(60, 60, 64), Enum.Material.Concrete, "HouseRoof")
		end
		house(-58, -58, 0) house(58, -58, 90) house(-58, 58, 90) house(58, 58, 0)
		-- corner watchtowers
		for _, sx in ipairs({ -1, 1 }) do
			for _, sz in ipairs({ -1, 1 }) do
				local x, z = sx * 92, sz * 92
				blk(f, x, 0, z, 14, 18, 14, GREY, Enum.Material.Concrete, "Watchtower")
				blk(f, x, 18, z, 15, 0.6, 15, YEL, Enum.Material.SmoothPlastic, "WatchtowerEdge")
				blk(f, x - sx * 5, 18.6, z, 1.2, 2.4, 14, GREY, Enum.Material.Concrete, "WatchtowerCover")
				ramp(f, Vector3.new(x - sx * 40, 0, z), Vector3.new(x - sx * 7, 18, z), 8, GREY, Enum.Material.Concrete)
			end
		end
		-- vehicle yards (lots of wrecks and trucks) + cover
		for i = -3, 3 do
			car(f, -30 + i * 6, -82 + R(-2, 2), 90 + R(-8, 8), nil, false)
			car(f, 30 + i * 6, 82 + R(-2, 2), 90 + R(-8, 8), nil, false)
			car(f, 82 + R(-2, 2), -30 + i * 9, R(-8, 8), nil, false)
			car(f, -82 + R(-2, 2), 30 + i * 9, R(-8, 8), nil, i % 3 == 0)
		end
		for i, p in ipairs({ { 0, -66 }, { 0, 66 }, { -30, -30 }, { 30, 30 }, { -30, 30 }, { 30, -30 } }) do
			truck(f, p[1], p[2], 90 * (i % 2), RGB(66, 70, 54))
		end
		for _, p in ipairs({ { -22, -46 }, { 22, 46 }, { -46, 22 }, { 46, -22 }, { -80, -10 }, { 80, 10 }, { 12, -92 }, { -12, 92 } }) do
			jersey(f, p[1], p[2], R(0, 180))
			jersey(f, p[1] + 6, p[2] + 3, R(0, 180))
			sandbags(f, p[1] + 10, p[2] - 8, R(0, 180))
		end
		-- pop-up target silhouettes
		for k = -40, 40, 16 do
			if math.abs(k) > 12 then
				blk(f, k, 0, -76, 2.4, 5, 0.3, RGB(160, 150, 110), Enum.Material.Wood, "Target")
				blk(f, k, 0, 76, 2.4, 5, 0.3, RGB(160, 150, 110), Enum.Material.Wood, "Target")
			end
		end
		for _, p in ipairs({ { -40, -40 }, { 40, 40 }, { -40, 40 }, { 40, -40 } }) do pole(f, p[1], p[2], true) end
		signboard(f, Vector3.new(0, 14, -107.6), Vector3.new(36, 6, 0.4), Enum.NormalId.Back, "PROVING GROUNDS", CY)
		return { spawns = sp, half = 112 }
	end

	-- =================================================================
	-- THE OFFICE (id GRID): abandoned corporate tower, night shift. Dark, flickering, but readable.
	-- Ground floor 152 x 152: west lobby, east server wing, central cubicle farm, north meeting rooms,
	-- south private offices with an executive mezzanine above them (ramps on both sides).
	-- =================================================================
	MapBuilders.GRID_OFFICE = function(f)
		MapRand = Random.new(13)
		local M = Enum.Material
		local H, MZ, DH = 18, 9, 7.6
		local function rnd(a, b) return MapRand:NextNumber(a, b) end
		local function pick(t) return t[MapRand:NextInteger(1, #t)] end
		local WALL, WALL2 = RGB(128, 132, 126), RGB(92, 98, 96)
		local TRIM, BLACK, METAL = RGB(36, 38, 40), RGB(18, 20, 22), RGB(84, 88, 92)
		local CARP, MARB, TILE, SLAB = RGB(54, 58, 62), RGB(78, 80, 84), RGB(100, 104, 102), RGB(60, 64, 66)
		local WOOD, WOOD2 = RGB(92, 68, 48), RGB(58, 44, 34)
		local FAB = { RGB(64, 74, 88), RGB(76, 70, 64), RGB(58, 78, 74) }
		local CHAIRC = { RGB(30, 32, 36), RGB(42, 40, 38), RGB(28, 38, 46) }
		local SCR = { RGB(70, 130, 175), RGB(60, 150, 120), RGB(140, 150, 160), RGB(165, 55, 48), RGB(90, 100, 170) }

		local function B(cx, by, cz, sx, sy, sz, col, mat, name, noCol)
			local p = blk(f, cx, by, cz, sx, sy, sz, col, mat, name)
			if noCol then p.CanCollide = false end
			return p
		end
		local function Q(cf, sx, sy, sz, col, mat, name, noCol)
			local p = place(f, cf, sx, sy, sz, col, mat, name)
			if noCol then p.CanCollide = false end
			return p
		end
		local function L(x, y, z, col, range, br)
			local p = Instance.new("Part")
			p.Anchored = true
			p.Transparency = 1
			p.CanCollide = false
			p.CanQuery = false
			p.CanTouch = false
			p.Size = Vector3.new(0.5, 0.5, 0.5)
			p.Position = Vector3.new(x, y, z)
			p.Name = "Lamp"
			p.Parent = f
			local l = Instance.new("PointLight")
			l.Color = col
			l.Range = range
			l.Brightness = br
			l.Shadows = false
			l.Parent = p
			return l
		end
		local function text(parent, face, str, col, font, ppsd)
			local g = Instance.new("SurfaceGui")
			g.Face = face
			g.SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud
			g.PixelsPerStud = ppsd or 40
			g.LightInfluence = 0
			g.Parent = parent
			local t = Instance.new("TextLabel")
			t.Size = UDim2.new(1, 0, 1, 0)
			t.BackgroundTransparency = 1
			t.Text = str
			t.TextScaled = true
			t.Font = font or Enum.Font.GothamBlack
			t.TextColor3 = col
			t.Parent = g
		end
		local function scrawl(pos, size, face, str, col)
			local p = Instance.new("Part")
			p.Anchored = true
			p.CanCollide = false
			p.CanQuery = false
			p.Transparency = 1
			p.Size = size
			p.Position = pos
			p.Name = "Scrawl"
			p.Parent = f
			text(p, face, str, col or RGB(120, 22, 20), Enum.Font.PermanentMarker, 30)
		end

		-- ---------- walls ----------
		local function seg(ax, fix, s, e, y0, y1, th, col, mat, name)
			if e - s < 0.05 or y1 - y0 < 0.05 then return nil end
			local len, mid = e - s, (s + e) / 2
			if ax == "x" then return B(mid, y0, fix, len, y1 - y0, th, col, mat, name or "Wall") end
			return B(fix, y0, mid, th, y1 - y0, len, col, mat, name or "Wall")
		end
		local function wall(ax, fix, a0, a1, gaps, h, th, col, dh)
			gaps = gaps or {}
			dh = dh or DH
			col = col or WALL
			table.sort(gaps, function(p, q) return p[1] < q[1] end)
			local cur = a0
			for _, g in ipairs(gaps) do
				local gs, ge = g[1] - g[2] / 2, g[1] + g[2] / 2
				seg(ax, fix, cur, gs, 0, h, th, col, M.SmoothPlastic)
				seg(ax, fix, gs, ge, dh, h, th, col, M.SmoothPlastic, "Lintel")
				cur = ge
			end
			seg(ax, fix, cur, a1, 0, h, th, col, M.SmoothPlastic)
		end
		local function gwall(ax, fix, a0, a1, gaps, h)
			gaps = gaps or {}
			table.sort(gaps, function(p, q) return p[1] < q[1] end)
			local function piece(s, e)
				if e - s < 0.05 then return end
				seg(ax, fix, s, e, 0, 3, 0.6, WALL2, M.SmoothPlastic)
				local g = seg(ax, fix, s, e, 3, 9, 0.25, RGB(120, 150, 160), M.Glass, "Glass")
				if g then g.Transparency = 0.62 end
				seg(ax, fix, s, e, 9, h, 0.6, WALL, M.SmoothPlastic)
			end
			local cur = a0
			for _, g in ipairs(gaps) do
				local gs, ge = g[1] - g[2] / 2, g[1] + g[2] / 2
				piece(cur, gs)
				seg(ax, fix, gs, ge, DH, h, 0.6, WALL, M.SmoothPlastic, "Lintel")
				cur = ge
			end
			piece(cur, a1)
		end
		local function leaf(ax, fix, c, w, ang)
			local len, gs, a = w - 0.3, c - w / 2, math.rad(ang)
			if ax == "x" then
				Q(CFrame.new(gs + math.cos(a) * len / 2, 3.7, fix + math.sin(a) * len / 2) * CFrame.Angles(0, -a, 0), len, 7.4, 0.3, WOOD2, M.Wood, "Door")
			else
				Q(CFrame.new(fix + math.sin(a) * len / 2, 3.7, gs + math.cos(a) * len / 2) * CFrame.Angles(0, a, 0), 0.3, 7.4, len, WOOD2, M.Wood, "Door")
			end
		end

		-- ---------- lights ----------
		local flick = {}
		local ONC, OFFC = RGB(176, 198, 186), RGB(48, 50, 50)
		local function fixture(x, z, state, y)
			y = y or H
			local on = state ~= "dead"
			local pan = B(x, y - 0.6, z, 5, 0.3, 1.6, on and ONC or OFFC, on and M.Neon or M.SmoothPlastic, "Fixture", true)
			B(x, y - 0.4, z, 5.6, 0.2, 2.2, TRIM, M.Metal, "FixtureFrame", true)
			if on then
				local l = L(x, y - 3, z, RGB(205, 226, 214), 30, 0.95)
				if state == "flick" then table.insert(flick, { pan = pan, light = l }) end
			end
		end
		local function exitSign(x, y, z, sx, sz, lx, lz)
			B(x, y, z, sx, 0.9, sz, RGB(210, 36, 32), M.Neon, "Exit", true)
			L(x + lx, y - 0.5, z + lz, RGB(255, 50, 40), 15, 0.8)
		end

		-- ---------- furniture ----------
		local function laptop(cf, col)
			Q(cf * CFrame.new(0, 0.06, 0), 1.7, 0.12, 1.2, RGB(40, 42, 46), M.Metal, "Laptop", true)
			local hinge = cf * CFrame.new(0, 0.12, -0.58) * CFrame.Angles(math.rad(-14), 0, 0)
			Q(hinge * CFrame.new(0, 0.55, 0), 1.7, 1.1, 0.07, RGB(30, 32, 36), M.Metal, "LaptopLid", true)
			if col then
				Q(hinge * CFrame.new(0, 0.55, 0.045), 1.5, 0.9, 0.03, col:Lerp(RGB(0, 0, 0), 0.3), M.Neon, "Screen", true)
			end
		end
		local function scr()
			if MapRand:NextNumber() < 0.22 then return nil end
			return pick(SCR)
		end
		local function monitor(cf, col)
			Q(cf * CFrame.new(0, 0.05, 0), 1.4, 0.1, 1, BLACK, M.Metal, "MonBase", true)
			Q(cf * CFrame.new(0, 0.5, 0), 0.4, 0.9, 0.4, BLACK, M.Metal, "MonNeck", true)
			Q(cf * CFrame.new(0, 1.7, 0), 2.8, 1.7, 0.14, BLACK, M.SmoothPlastic, "Monitor", true)
			if col then Q(cf * CFrame.new(0, 1.7, 0.09), 2.6, 1.5, 0.03, col:Lerp(RGB(0, 0, 0), 0.35), M.Neon, "Screen", true) end
		end
		local function chair(cf, knocked)
			if knocked then cf = cf * CFrame.new(0, 1.1, 0) * CFrame.Angles(0, 0, math.rad(rnd(82, 98))) end
			local c = pick(CHAIRC)
			Q(cf * CFrame.new(0, 0.3, 0), 2.4, 0.2, 2.4, BLACK, M.Metal, "ChairBase", true)
			Q(cf * CFrame.new(0, 0.95, 0), 0.35, 1.2, 0.35, METAL, M.Metal, "ChairPost", true)
			Q(cf * CFrame.new(0, 1.65, 0), 2.1, 0.4, 2.1, c, M.Fabric, "ChairSeat", true)
			Q(cf * CFrame.new(0, 3.0, 0.95), 2.0, 2.3, 0.3, c, M.Fabric, "ChairBack", true)
			Q(cf * CFrame.new(-1.05, 2.2, 0), 0.2, 0.2, 1.5, BLACK, M.Metal, "ChairArm", true)
			Q(cf * CFrame.new(1.05, 2.2, 0), 0.2, 0.2, 1.5, BLACK, M.Metal, "ChairArm", true)
		end
		local function papers(x, y, z, n, spread)
			for _ = 1, n do
				Q(CFrame.new(x + rnd(-spread, spread), y, z + rnd(-spread, spread)) * CFrame.Angles(0, rnd(0, 6.2), 0), 1.1, 0.04, 1.5, RGB(176, 176, 166), M.SmoothPlastic, "Paper", true)
			end
		end
		local lampN = 0
		local function deskLamp(cf)
			Q(cf * CFrame.new(0, 0.08, 0), 0.9, 0.16, 0.9, BLACK, M.Metal, "LampBase", true)
			Q(cf * CFrame.new(0, 0.9, 0), 0.15, 1.6, 0.15, BLACK, M.Metal, "LampArm", true)
			Q(cf * CFrame.new(0, 1.75, 0.3), 0.9, 0.3, 0.9, RGB(255, 196, 120), M.Neon, "LampHead", true)
			if lampN < 14 then
				lampN += 1
				local p = cf.Position
				L(p.X, p.Y + 1.5, p.Z, RGB(255, 190, 120), 15, 0.8)
			end
		end
		local function desk(cf, w, withStuff)
			Q(cf * CFrame.new(0, 2.65, 1.65), w, 0.3, 2.9, WOOD, M.Wood, "Desk")
			Q(cf * CFrame.new(-w / 2 + 0.2, 1.25, 1.65), 0.3, 2.5, 2.7, WOOD2, M.Wood, "DeskSide")
			Q(cf * CFrame.new(w / 2 - 1.5, 1.25, 1.65), 2.6, 2.5, 2.7, METAL, M.Metal, "Drawers")
			if withStuff then
				laptop(cf * CFrame.new(rnd(-w / 4, 0), 2.8, 1.8), scr())
				if MapRand:NextNumber() < 0.5 then monitor(cf * CFrame.new(-w / 2 + 2.4, 2.8, 0.9), scr()) end
				if MapRand:NextNumber() < 0.5 then local pp = (cf * CFrame.new(0, 2.82, 1.8)).Position; papers(pp.X, pp.Y, pp.Z, 2, 1.2) end
				if MapRand:NextNumber() < 0.14 then deskLamp(cf * CFrame.new(w / 2 - 2.8, 2.8, 0.9)) end
			end
		end
		local function cabinet(cf)
			Q(cf, 2.2, 4.6, 2, RGB(92, 96, 100), M.Metal, "Cabinet")
			for i = 0, 2 do Q(cf * CFrame.new(0, -1.5 + i * 1.5, 1.02), 1.8, 0.08, 0.05, RGB(40, 42, 46), M.Metal, "Drawer", true) end
		end
		local function plant(cf)
			Q(cf * CFrame.new(0, 0.7, 0), 1.5, 1.4, 1.5, RGB(46, 44, 42), M.Concrete, "Pot")
			for i = 1, 3 do
				Q(cf * CFrame.new(rnd(-0.3, 0.3), 2.2 + i * 0.4, rnd(-0.3, 0.3)) * CFrame.Angles(rnd(-0.4, 0.4), rnd(0, 3), rnd(-0.4, 0.4)), 1.7 - i * 0.2, 1.3, 1.7 - i * 0.2, RGB(30, 58, 34), M.Grass, "Leaves", true)
			end
		end
		local function sofa(cf, col)
			col = col or RGB(54, 40, 38)
			Q(cf * CFrame.new(0, 0.9, 0), 7, 1.4, 3, col, M.Fabric, "Sofa")
			Q(cf * CFrame.new(0, 2.3, -1.2), 7, 2, 0.7, col, M.Fabric, "SofaBack")
			Q(cf * CFrame.new(-3.6, 1.6, 0), 0.8, 2.2, 3, col, M.Fabric, "SofaArm")
			Q(cf * CFrame.new(3.6, 1.6, 0), 0.8, 2.2, 3, col, M.Fabric, "SofaArm")
		end
		local function table4(cf, w, d, col)
			Q(cf * CFrame.new(0, 2.7, 0), w, 0.35, d, col or WOOD, M.Wood, "Table")
			for _, sx in ipairs({ -1, 1 }) do
				for _, sz in ipairs({ -1, 1 }) do
					Q(cf * CFrame.new(sx * (w / 2 - 0.4), 1.3, sz * (d / 2 - 0.4)), 0.4, 2.6, 0.4, WOOD2, M.Wood, "TableLeg")
				end
			end
		end
		local function vending(cf, col)
			Q(cf * CFrame.new(0, 3.2, 0), 3.2, 6.4, 2.6, RGB(26, 28, 32), M.Metal, "Vending")
			Q(cf * CFrame.new(-0.25, 3.5, 1.32), 2.2, 4.2, 0.08, col, M.Neon, "VendFace", true)
			Q(cf * CFrame.new(1.15, 3.2, 1.32), 0.5, 3, 0.08, RGB(14, 14, 16), M.Metal, "VendPanel", true)
			local p = (cf * CFrame.new(0, 3.5, 3.4)).Position
			L(p.X, p.Y, p.Z, col, 20, 0.9)
		end
		local function rack(cf)
			Q(cf * CFrame.new(0, 3.9, 0), 2.6, 7.8, 3, RGB(22, 24, 28), M.Metal, "Rack")
			for i = 0, 6 do
				local c = (MapRand:NextNumber() < 0.15) and RGB(255, 50, 40) or ((MapRand:NextNumber() < 0.5) and RGB(60, 255, 120) or RGB(60, 150, 255))
				Q(cf * CFrame.new(rnd(-0.9, 0.9), 1.2 + i * 1.0, 1.52), 0.22, 0.14, 0.05, c, M.Neon, "Led", true)
				Q(cf * CFrame.new(rnd(-0.9, 0.9), 1.2 + i * 1.0, 1.52), 0.22, 0.14, 0.05, c, M.Neon, "Led", true)
			end
		end
		local function shelf(cf, w)
			for _, sx in ipairs({ -1, 1 }) do Q(cf * CFrame.new(sx * w / 2, 3.6, 0), 0.25, 7.2, 2.2, METAL, M.Metal, "ShelfPost") end
			for i = 0, 4 do
				Q(cf * CFrame.new(0, 0.4 + i * 1.7, 0), w, 0.15, 2.2, RGB(70, 74, 78), M.Metal, "ShelfBoard")
				local x = -w / 2 + 0.8
				while x < w / 2 - 0.8 do
					if MapRand:NextNumber() < 0.7 then
						local bw = rnd(1.0, 1.9)
						Q(cf * CFrame.new(x, 0.5 + i * 1.7 + 0.55, 0), bw, 1.0, 1.7, RGB(rnd(120, 150), rnd(100, 120), rnd(70, 90)), M.Wood, "Box", true)
					end
					x += 2.1
				end
			end
		end
		local function boxes(x, z, n)
			for i = 1, n do
				Q(CFrame.new(x + rnd(-1.2, 1.2), 0.9 + (i > 2 and 1.8 or 0), z + rnd(-1, 1)) * CFrame.Angles(0, rnd(0, 3), 0), 2.2, 1.8, 2.2, RGB(rnd(120, 150), rnd(100, 120), rnd(70, 90)), M.Wood, "Box")
			end
		end
		local function whiteboard(cf, str, col)
			local p = Q(cf, 9, 4.6, 0.25, RGB(176, 180, 176), M.SmoothPlastic, "Whiteboard", true)
			Q(cf * CFrame.new(0, -2.5, 0.2), 9.2, 0.2, 0.4, METAL, M.Metal, "BoardTray", true)
			if str then text(p, Enum.NormalId.Front, str, col or RGB(30, 50, 120), Enum.Font.PermanentMarker, 24) end
		end
		local function bin(x, z)
			Q(CFrame.new(x, 1, z), 1.3, 2, 1.3, RGB(40, 42, 44), M.Metal, "Bin", true)
		end
		local function stain(x, z, sx, sz)
			Q(CFrame.new(x, 0.1, z) * CFrame.Angles(0, rnd(0, 3), 0), sx, 0.03, sz, RGB(52, 14, 12), M.SmoothPlastic, "Stain", true)
		end
		local function cooler(x, z)
			Q(CFrame.new(x, 1.5, z), 1.4, 3, 1.4, RGB(176, 180, 184), M.SmoothPlastic, "Cooler")
			Q(CFrame.new(x, 3.8, z), 1.1, 1.8, 1.1, RGB(110, 170, 200), M.Glass, "WaterJug", true)
		end

		-- ---------- one cubicle cell (local +Z = toward the seated person, back partition at z = 0) ----------
		local function cell(x, z, sgn)
			local base = CFrame.new(x, 0, z) * CFrame.Angles(0, sgn == 1 and 0 or math.pi, 0)
			desk(base, 9.2, true)
			chair(base * CFrame.new(rnd(-2.5, 2.5), 0, rnd(4.2, 5.8)) * CFrame.Angles(0, rnd(-0.7, 0.7), 0), MapRand:NextNumber() < 0.08)
			if MapRand:NextNumber() < 0.3 then bin(x + (sgn == 1 and 3.8 or -3.8), z + sgn * 5.5) end
			if MapRand:NextNumber() < 0.35 then papers(x, 0.09, z + sgn * rnd(4, 6.5), 2, 1.8) end
		end
		local function pod(cx, cz)
			local fab = pick(FAB)
			B(cx, 0, cz, 20.4, 4.6, 0.4, fab, M.Fabric, "Cubicle")
			for _, sgn in ipairs({ -1, 1 }) do
				for _, px in ipairs({ -10, 0, 10 }) do
					B(cx + px, 0, cz + sgn * 3.7, 0.4, 4.6, 7.4, fab, M.Fabric, "Cubicle")
				end
				cell(cx - 5, cz, sgn)
				cell(cx + 5, cz, sgn)
			end
		end

		-- =============== SHELL ===============
		B(0, -3, 0, 162, 3.05, 162, CARP, M.Fabric, "Carpet")
		B(0, H, 0, 158, 1.2, 158, RGB(70, 74, 74), M.Concrete, "Ceiling")
		local OUT = RGB(70, 74, 76)
		B(0, 0, -77, 158, H, 2, OUT, M.Concrete, "OuterWall")
		B(0, 0, 77, 158, H, 2, OUT, M.Concrete, "OuterWall")
		B(-77, 0, 0, 2, H, 158, OUT, M.Concrete, "OuterWall")
		B(77, 0, 0, 2, H, 158, OUT, M.Concrete, "OuterWall")
		-- zone floors
		local function patch(cx, cz, sx, sz, col, mat) B(cx, 0.05, cz, sx, 0.05, sz, col, mat, "FloorPatch", true) end
		patch(-62, 0, 28, 28, MARB, M.Marble)
		patch(62, 0, 28, 28, SLAB, M.Concrete)
		patch(62, 45, 28, 62, TILE, M.Slate)
		patch(62, -45, 28, 62, RGB(38, 42, 44), M.Metal)
		patch(-62, 45, 28, 62, RGB(66, 58, 52), M.Fabric)
		patch(0, -64, 96, 24, RGB(60, 50, 44), M.Fabric)
		patch(0, 0, 20, 58, RGB(46, 50, 56), M.Fabric)
		-- windows (faint blue night glow behind torn blinds)
		for _, s in ipairs({ -1, 1 }) do
			for k = -66, 66, 12 do
				local wp = B(k, 5, s * 75.9, 7, 8, 0.15, RGB(16, 28, 46), M.Neon, "Window", true)
				B(k, 9 + rnd(0, 1.5), s * 75.7, 7, 1.0, 0.1, RGB(30, 32, 34), M.SmoothPlastic, "Blind", true)
				if MapRand:NextNumber() < 0.5 then B(k + rnd(-1.5, 1.5), 6.2, s * 75.7, 5, 0.9, 0.1, RGB(30, 32, 34), M.SmoothPlastic, "Blind", true) end
				B(k, 4.5, s * 75.8, 7.4, 0.3, 0.3, TRIM, M.Metal, "Sill", true)
				wp.Name = "Window"
			end
			for k = -66, 66, 12 do
				if not (s == -1 and math.abs(k) < 14) then
					B(s * 75.9, 5, k, 0.15, 8, 7, RGB(16, 28, 46), M.Neon, "Window", true)
					B(s * 75.7, 9 + rnd(0, 1.5), k, 0.1, 1.0, 7, RGB(30, 32, 34), M.SmoothPlastic, "Blind", true)
					B(s * 75.8, 4.5, k, 0.3, 0.3, 7.4, TRIM, M.Metal, "Sill", true)
				end
			end
		end

		-- main partitions
		for _, s in ipairs({ -1, 1 }) do
			wall("z", s * 48, -76, 76, { { -34, 8 }, { 0, 14 }, { 34, 8 } }, H, 2, WALL)
			wall("x", -14, s < 0 and -76 or 49, s < 0 and -49 or 76, { { s * 62, 10 } }, H, 0.8, WALL)
			wall("x", 14, s < 0 and -76 or 49, s < 0 and -49 or 76, { { s * 62, 10 } }, H, 0.8, WALL)
		end
		-- north meeting rooms
		local ROOMX = { -40, -24, -8, 8, 24, 40 }
		local gaps = {}
		for _, cx in ipairs(ROOMX) do table.insert(gaps, { cx, 6 }) end
		gwall("x", -52, -47, 47, gaps, H)
		for _, px in ipairs({ -32, -16, 0, 16, 32 }) do wall("z", px, -76, -52, {}, H, 0.6, WALL) end
		-- south private offices under the mezzanine
		local gaps2 = {}
		for _, cx in ipairs(ROOMX) do table.insert(gaps2, { cx, 6 }) end
		wall("x", 52, -47, 47, gaps2, MZ - 1, 0.6, WALL, 6.6)
		for _, px in ipairs({ -32, -16, 0, 16, 32 }) do wall("z", px, 52, 76, {}, MZ - 1, 0.6, WALL) end
		-- mezzanine slab with a neon lip
		B(0, MZ - 1, 63, 96, 1, 26, RGB(58, 62, 66), M.Concrete, "MezzFloor")
		B(0, MZ - 1.4, 49.85, 96, 0.2, 0.15, RGB(40, 150, 170), M.Neon, "MezzLip", true)

		-- door leaves: some left ajar, some torn
		for i, cx in ipairs(ROOMX) do
			if i % 2 == 1 then leaf("x", -52, cx, 6, rnd(55, 85)) end
			if i % 3 ~= 0 then leaf("x", 52, cx, 6, rnd(-85, -50) * -1) end
		end

		-- =============== CEILING LIGHTS ===============
		local fx = { -36, -12, 12, 36 }
		local fz = { -40, -20, 0, 20, 40 }
		for _, x in ipairs(fx) do
			for _, z in ipairs(fz) do
				local r = MapRand:NextNumber()
				local st = (r < 0.5) and "on" or ((r < 0.68) and "flick" or "dead")
				if z == 0 and math.abs(x) == 12 then st = "on" end
				fixture(x, z, st)
			end
		end
		for _, cx in ipairs(ROOMX) do
			fixture(cx, -64, pick({ "on", "dead", "flick" }))
		end
		for i, cx in ipairs(ROOMX) do
			-- under the mezzanine: just a dim emergency tube in some rooms
			if i % 2 == 0 then fixture(cx, 64, "on", MZ - 1.0) end
		end
		for _, sx in ipairs({ -1, 1 }) do
			fixture(sx * 62, 0, "on")
			fixture(sx * 62, -44, sx < 0 and "on" or "flick")
			fixture(sx * 62, 44, sx < 0 and "flick" or "on")
			fixture(sx * 62, -64, "dead")
			fixture(sx * 62, 64, "on")
		end
		for _, x in ipairs({ -30, 0, 30 }) do fixture(x, 62, "on", H) end -- mezzanine
		-- ducts and pipes
		for _, x in ipairs({ -24, 24 }) do B(x, H - 3.6, 0, 3, 2.2, 100, RGB(70, 74, 78), M.Metal, "Duct", true) end
		B(0, H - 2.4, -26, 100, 0.6, 0.6, RGB(88, 60, 44), M.Metal, "Pipe", true)
		B(0, H - 2.4, 26, 100, 0.6, 0.6, RGB(88, 60, 44), M.Metal, "Pipe", true)
		-- exit signs over the hall gaps (both sides)
		for _, z in ipairs({ -34, 0, 34 }) do
			exitSign(-46.7, 9.2, z, 0.3, 2.4, 2, 0)
			exitSign(46.7, 9.2, z, 0.3, 2.4, -2, 0)
		end
		exitSign(-1.2, 7.4, -51.7, 2.4, 0.3, 0, 2)
		exitSign(1.2, 7.0, 52.3, 2.4, 0.3, 0, -2)

		-- =============== HALL ===============
		for _, z in ipairs({ -36, -12, 12, 36 }) do
			pod(-29, z)
			pod(29, z)
		end
		pod(0, -36)
		pod(0, 36)
		for _, x in ipairs({ -15, 15 }) do
			for _, z in ipairs({ -24, 0, 24 }) do
				B(x, 0, z, 2.4, H, 2.4, WALL2, M.Concrete, "Column")
				B(x, 0, z, 3, 0.8, 3, TRIM, M.Metal, "ColumnBase", true)
			end
		end
		-- central lounge
		sofa(CFrame.new(0, 0, -9), nil)
		sofa(CFrame.new(0, 0, 9) * CFrame.Angles(0, math.pi, 0), RGB(48, 52, 58))
		table4(CFrame.new(0, 0, 0), 5, 3, RGB(34, 30, 28))
		laptop(CFrame.new(0.5, 2.9, 0.2), SCR[1])
		plant(CFrame.new(-7, 0, -4))
		plant(CFrame.new(7, 0, 4))
		cooler(-6.5, 12)
		cooler(6.5, -12)
		-- barricade of desks and chairs at the west lobby doors (horror: someone tried to keep something out)
		for i = 1, 3 do
			Q(CFrame.new(-44, 1.6 + (i - 1) * 0.1, -9 + i * 1.7) * CFrame.Angles(0, rnd(-0.3, 0.3), math.rad(88)), 5.5, 0.4, 3, WOOD, M.Wood, "FlippedDesk")
		end
		Q(CFrame.new(-44, 4.2, -4) * CFrame.Angles(0, 0.3, math.rad(80)), 5.5, 0.4, 3, WOOD, M.Wood, "FlippedDesk")
		chair(CFrame.new(-42, 0, -2), true)
		chair(CFrame.new(-41, 0, -10), true)
		-- a second, smaller pile on the east side
		Q(CFrame.new(44, 1.6, 7) * CFrame.Angles(0, 0.2, math.rad(88)), 5.5, 0.4, 3, WOOD, M.Wood, "FlippedDesk")
		Q(CFrame.new(44, 1.6, 10) * CFrame.Angles(0, -0.2, math.rad(88)), 5.5, 0.4, 3, WOOD, M.Wood, "FlippedDesk")
		chair(CFrame.new(41.5, 0, 12), true)
		-- ramps up to the mezzanine
		for _, s in ipairs({ -1, 1 }) do
			local a, b = Vector3.new(s * 43.5, 0, 18), Vector3.new(s * 43.5, MZ, 50.3)
			ramp(f, a, b, 6, RGB(60, 62, 66), M.Metal)
			for _, off in ipairs({ -3.15, 3.15 }) do
				local aa, bb = a + Vector3.new(off, 0, 0), b + Vector3.new(off, 0, 0)
				local len = (bb - aa).Magnitude
				Q(CFrame.lookAt(aa, bb) * CFrame.new(0, 1.6, -len / 2), 0.3, 3.2, len, RGB(46, 48, 52), M.Metal, "RampRail")
			end
			B(s * 43.5, 0, 17.4, 6, 0.4, 1.5, RGB(220, 170, 30), M.SmoothPlastic, "RampMark", true)
		end
		-- mezzanine railing (half wall: shoot over it, duck behind it)
		B(0, MZ, 50.75, 80, 3.2, 0.5, RGB(48, 40, 34), M.Wood, "MezzRail")
		B(0, MZ + 3.2, 50.75, 80.4, 0.2, 0.8, TRIM, M.Metal, "MezzRailCap", true)
		for _, s in ipairs({ -1, 1 }) do
			B(s * 40.2, MZ, 53.5, 0.5, 3.2, 6, RGB(48, 40, 34), M.Wood, "MezzRail")
		end
		-- horror scrawls
		scrawl(Vector3.new(-47.2, 7, -20), Vector3.new(0.1, 3, 12), Enum.NormalId.Right, "THEY NEVER LEFT")
		scrawl(Vector3.new(47.2, 7, 20), Vector3.new(0.1, 3, 12), Enum.NormalId.Left, "DON'T TURN OFF THE LIGHTS")
		scrawl(Vector3.new(-40, 6, -51.6), Vector3.new(14, 3, 0.1), Enum.NormalId.Back, "NIGHT SHIFT")
		-- floor stains, papers, drag marks
		stain(-30, -24, 3.5, 2.5)
		stain(-22, -24, 1.8, 1.2)
		stain(5, 22, 4, 3)
		stain(24, 0, 2.5, 2)
		stain(-12, 26, 2, 5)
		stain(-12, 31, 1, 3)
		stain(40, -22, 3, 2)
		stain(-3, -33, 2, 6)
		for i = 0, 5 do stain(-44 + i * 1.8, -22 + i * 0.7, 0.8, 0.5) end
		for _ = 1, 26 do papers(rnd(-44, 44), 0.09, rnd(-48, 48), 1, 0) end

		-- =============== NORTH MEETING ROOMS ===============
		local MT = { "Q3 TARGETS", "ALL HANDS", "EXIT PLAN", "BUDGET", "SYNC 9AM", "DO NOT ENTER" }
		for i, cx in ipairs(ROOMX) do
			local cf = CFrame.new(cx, 0, -64)
			if i == 3 or i == 5 then
				-- trashed room: table shoved over
				Q(cf * CFrame.new(2, 2.5, 3) * CFrame.Angles(0, 0.4, math.rad(90)), 9, 0.4, 3.6, WOOD, M.Wood, "FlippedTable")
				chair(cf * CFrame.new(-4, 0, 4), true)
				chair(cf * CFrame.new(5, 0, -2), true)
				chair(cf * CFrame.new(-3, 0, -3) * CFrame.Angles(0, 2, 0), false)
				papers(cx, 0.09, -64, 8, 6)
				stain(cx - 2, -66, 3, 2)
			else
				table4(cf, 9.5, 3.6, WOOD)
				for k = -1, 1 do
					chair(cf * CFrame.new(k * 3, 0, 3.4), false)
					chair(cf * CFrame.new(k * 3, 0, -3.4) * CFrame.Angles(0, math.pi, 0), false)
				end
				laptop(cf * CFrame.new(-3, 2.9, -0.8) * CFrame.Angles(0, math.pi, 0), scr())
				laptop(cf * CFrame.new(2.4, 2.9, 0.9), scr())
				papers(cx, 2.9, -64, 3, 1.5)
			end
			whiteboard(CFrame.new(cx, 5.8, -75.6), MT[i], i == 6 and RGB(150, 24, 22) or nil)
			plant(CFrame.new(cx + 6, 0, -74))
			signboard(f, Vector3.new(cx, 9.8, -51.6), Vector3.new(6, 1.2, 0.2), Enum.NormalId.Back, "ROOM " .. (i + 10), RGB(160, 164, 150))
		end

		-- =============== SOUTH OFFICES (under the mezzanine) ===============
		for i, cx in ipairs(ROOMX) do
			local cf = CFrame.new(cx, 0, 74.4) * CFrame.Angles(0, math.pi, 0)
			desk(cf, 9.2, true)
			chair(cf * CFrame.new(rnd(-2, 2), 0, rnd(4.4, 5.6)) * CFrame.Angles(0, rnd(-0.8, 0.8), 0), i == 2)
			cabinet(CFrame.new(cx - 6.2, 0, 73.5) * CFrame.Angles(0, math.pi, 0))
			cabinet(CFrame.new(cx + 6.2, 0, 73.5) * CFrame.Angles(0, math.pi, 0))
			if i % 2 == 1 then plant(CFrame.new(cx + 6, 0, 56)) end
			signboard(f, Vector3.new(cx, 7.1, 51.7), Vector3.new(5, 0.8, 0.2), Enum.NormalId.Front, "OFFICE " .. (i + 20), RGB(160, 164, 150))
		end
		stain(8, 58, 3.5, 2.5)
		stain(-24, 66, 2, 3)

		-- =============== MEZZANINE (executive floor) ===============
		for k, x in ipairs({ -36, -20, -4, 12, 28 }) do
			local cf = CFrame.new(x, MZ, 74.3) * CFrame.Angles(0, math.pi, 0)
			desk(cf, 9.2, true)
			chair(cf * CFrame.new(rnd(-2, 2), 0, rnd(4.4, 5.8)) * CFrame.Angles(0, rnd(-0.8, 0.8), 0), k == 4)
		end
		table4(CFrame.new(0, MZ, 61), 12, 4, RGB(34, 30, 28))
		for k = -2, 2 do
			chair(CFrame.new(k * 2.6, MZ, 57.3) * CFrame.Angles(0, math.pi, 0), k == 2)
			chair(CFrame.new(k * 2.6, MZ, 64.7), false)
		end
		laptop(CFrame.new(-3, MZ + 2.9, 61), scr())
		laptop(CFrame.new(3, MZ + 2.9, 61) * CFrame.Angles(0, math.pi, 0), scr())
		sofa(CFrame.new(-34, MZ, 58) * CFrame.Angles(0, math.pi / 2, 0), RGB(40, 44, 52))
		sofa(CFrame.new(34, MZ, 58) * CFrame.Angles(0, -math.pi / 2, 0), RGB(40, 44, 52))
		for _, x in ipairs({ -28, -24, 24, 28 }) do cabinet(CFrame.new(x, MZ, 66)) end
		for _, x in ipairs({ -42, 42 }) do shelf(CFrame.new(x, MZ, 70), 6) end
		plant(CFrame.new(-44, MZ, 53))
		plant(CFrame.new(44, MZ, 53))
		whiteboard(CFrame.new(0, MZ + 5.8, 75.6) * CFrame.Angles(0, math.pi, 0), "WHO TURNED OFF THE CAMERAS?", RGB(150, 24, 22))
		signboard(f, Vector3.new(0, MZ + 8.4, 50.9), Vector3.new(14, 1.4, 0.2), Enum.NormalId.Front, "EXECUTIVE SUITE", RGB(170, 150, 100))

		-- =============== WEST WING ===============
		-- reception (Alpha spawn pocket)
		B(-56, 0, -6, 3, 3.4, 6, RGB(40, 34, 30), M.Wood, "Reception")
		B(-56, 3.4, -6, 3.4, 0.25, 6.4, RGB(100, 96, 90), M.Marble, "ReceptionTop", true)
		B(-56, 0, 6, 3, 3.4, 6, RGB(40, 34, 30), M.Wood, "Reception")
		B(-56, 3.4, 6, 3.4, 0.25, 6.4, RGB(100, 96, 90), M.Marble, "ReceptionTop", true)
		laptop(CFrame.new(-56.4, 3.65, -5) * CFrame.Angles(0, -math.pi / 2, 0), SCR[1])
		laptop(CFrame.new(-56.4, 3.65, 6) * CFrame.Angles(0, -math.pi / 2, 0), SCR[2])
		chair(CFrame.new(-60, 0, -5) * CFrame.Angles(0, -math.pi / 2, 0), false)
		chair(CFrame.new(-60, 0, 8) * CFrame.Angles(0, -math.pi / 2, 0), true)
		for i = 0, 2 do
			local z = -8 + i * 8
			B(-75.5, 0, z, 0.8, 8.6, 5.2, RGB(120, 124, 128), M.Metal, "ElevatorDoor")
			B(-75.0, 8.6, z, 0.3, 0.7, 2.4, RGB(255, 170, 40), M.Neon, "FloorIndicator", true)
			B(-75.0, 3.4, z + 3.4, 0.3, 0.7, 0.4, RGB(255, 170, 40), M.Neon, "CallButton", true)
		end
		signboard(f, Vector3.new(-74.6, 12.6, 0), Vector3.new(0.4, 3, 16), Enum.NormalId.Right, "MERIDIAN CORP", RGB(150, 142, 106))
		L(-72, 10, 0, RGB(255, 170, 70), 22, 0.7)
		plant(CFrame.new(-73.5, 0, -12))
		plant(CFrame.new(-73.5, 0, 12))
		sofa(CFrame.new(-52.5, 0, -10.5) * CFrame.Angles(0, -math.pi / 2, 0), nil)
		-- north-west: records / copy room
		for _, x in ipairs({ -72, -56 }) do
			for _, z in ipairs({ -72, -64, -56, -48, -20 }) do
				if not (x == -56 and z == -48) then shelf(CFrame.new(x, 0, z) * CFrame.Angles(0, math.pi / 2, 0), 7) end
			end
		end
		for _, z in ipairs({ -40, -28 }) do cabinet(CFrame.new(-74, 0, z) * CFrame.Angles(0, math.pi / 2, 0)) end
		Q(CFrame.new(-62, 1.3, -22), 3.4, 2.6, 2.6, RGB(150, 152, 150), M.SmoothPlastic, "Copier")
		Q(CFrame.new(-62, 2.75, -22), 3.2, 0.3, 2.4, RGB(30, 80, 100), M.Neon, "CopierLid", true)
		boxes(-66, -18, 4)
		boxes(-74, -17, 3)
		papers(-62, 0.09, -30, 14, 5)
		stain(-64, -44, 3, 2.5)
		-- south-west: waiting lounge
		sofa(CFrame.new(-71, 0, 24) * CFrame.Angles(0, math.pi / 2, 0), RGB(44, 50, 58))
		sofa(CFrame.new(-71, 0, 50) * CFrame.Angles(0, math.pi / 2, 0), RGB(54, 40, 38))
		table4(CFrame.new(-64, 0, 24), 3, 4.5, RGB(36, 32, 30))
		table4(CFrame.new(-64, 0, 50), 3, 4.5, RGB(36, 32, 30))
		vending(CFrame.new(-74.2, 0, 36) * CFrame.Angles(0, math.pi / 2, 0), RGB(255, 140, 40))
		vending(CFrame.new(-74.2, 0, 41) * CFrame.Angles(0, math.pi / 2, 0), RGB(70, 170, 255))
		plant(CFrame.new(-73, 0, 18))
		plant(CFrame.new(-73, 0, 72))
		plant(CFrame.new(-52, 0, 72))
		cooler(-52, 20)
		scrawl(Vector3.new(-49.3, 8, 60), Vector3.new(0.1, 3, 14), Enum.NormalId.Left, "HELP", RGB(120, 22, 20))
		papers(-62, 0.09, 45, 12, 8)
		stain(-60, 62, 3, 3)

		-- =============== EAST WING ===============
		-- mailroom (Bravo spawn pocket)
		B(56, 0, -6, 3, 3.4, 6, RGB(90, 94, 98), M.Metal, "SortingBench")
		B(56, 0, 6, 3, 3.4, 6, RGB(90, 94, 98), M.Metal, "SortingBench")
		laptop(CFrame.new(56.4, 3.5, -5) * CFrame.Angles(0, math.pi / 2, 0), SCR[3])
		laptop(CFrame.new(56.4, 3.5, 6) * CFrame.Angles(0, math.pi / 2, 0), SCR[1])
		chair(CFrame.new(60, 0, -5) * CFrame.Angles(0, math.pi / 2, 0), true)
		chair(CFrame.new(60, 0, 7) * CFrame.Angles(0, math.pi / 2, 0), false)
		for i = 0, 2 do
			boxes(75, -12 + i * 12, 3)
		end
		Q(CFrame.new(73.5, 0.6, 11.5), 3, 1.2, 4, RGB(120, 100, 70), M.Wood, "Pallet")
		Q(CFrame.new(73.5, 0.6, -11.5), 3, 1.2, 4, RGB(120, 100, 70), M.Wood, "Pallet")
		-- mail cart
		Q(CFrame.new(66, 1.4, 11), 3.4, 0.3, 2, METAL, M.Metal, "Cart")
		Q(CFrame.new(66, 2.4, 11), 3.2, 1.6, 1.8, RGB(150, 120, 90), M.Wood, "Box", true)
		L(70, 12, 0, RGB(210, 220, 255), 24, 0.6)
		-- north-east: server room
		for _, x in ipairs({ 72, 56 }) do
			for _, z in ipairs({ -72, -66, -60, -54, -48, -20, -26 }) do
				if not (x == 56 and (z == -48 or z == -54)) then rack(CFrame.new(x, 0, z) * CFrame.Angles(0, (x > 64) and -math.pi / 2 or math.pi / 2, 0)) end
			end
		end
		for _, z in ipairs({ -60, -40, -24 }) do L(64, 6, z, RGB(60, 130, 255), 20, 0.7) end
		L(64, 6, -70, RGB(255, 60, 40), 16, 0.5)
		cabinet(CFrame.new(75, 0, -40) * CFrame.Angles(0, -math.pi / 2, 0))
		cabinet(CFrame.new(75, 0, -33) * CFrame.Angles(0, -math.pi / 2, 0))
		Q(CFrame.new(64, 2.9, -40), 2.4, 0.3, 4, WOOD, M.Wood, "ServerDesk")
		laptop(CFrame.new(64, 3.05, -40) * CFrame.Angles(0, math.pi / 2, 0), SCR[4])
		scrawl(Vector3.new(49.3, 8, -22), Vector3.new(0.1, 3, 9), Enum.NormalId.Right, "IT WAS ALREADY INSIDE", RGB(120, 22, 20))
		stain(64, -30, 3, 3)
		-- south-east: cafeteria
		for _, x in ipairs({ 58, 70 }) do
			for _, z in ipairs({ 22, 50, 64 }) do
				table4(CFrame.new(x, 0, z), 5, 3.4, RGB(76, 70, 62))
				chair(CFrame.new(x, 0, z + 3), (x + z) % 7 == 0)
				chair(CFrame.new(x, 0, z - 3) * CFrame.Angles(0, math.pi, 0), (x * z) % 5 == 0)
			end
		end
		laptop(CFrame.new(58, 2.9, 50.5), SCR[2])
		laptop(CFrame.new(70, 2.9, 22.5) * CFrame.Angles(0, math.pi, 0), SCR[5])
		vending(CFrame.new(52, 0, 15.5) * CFrame.Angles(0, math.pi, 0), RGB(80, 200, 120))
		vending(CFrame.new(72.5, 0, 15.5) * CFrame.Angles(0, math.pi, 0), RGB(255, 100, 80))
		-- kitchen counter and fridge on the south wall
		B(62, 0, 74.4, 22, 3.2, 2.4, RGB(60, 56, 52), M.Wood, "KitchenCounter")
		B(62, 3.2, 74.4, 22.4, 0.25, 2.8, RGB(100, 100, 98), M.Marble, "CounterTop", true)
		Q(CFrame.new(72.5, 3.6, 74), 2.8, 7.2, 2.6, RGB(170, 172, 174), M.Metal, "Fridge")
		Q(CFrame.new(54, 3.6, 74), 2.6, 7.2, 2.4, RGB(30, 32, 34), M.Metal, "Oven")
		plant(CFrame.new(49.5, 0, 72))
		stain(66, 36, 2.5, 4)
		stain(58, 70, 2, 2)
		papers(62, 0.09, 40, 10, 8)

		-- =============== FLICKER ===============
		for _, fl in ipairs(flick) do
			task.spawn(function()
				local rng = Random.new(MapRand:NextInteger(1, 1000000))
				while fl.pan.Parent do
					task.wait(rng:NextNumber(1.5, 6))
					if not fl.pan.Parent then break end
					for i = 1, rng:NextInteger(3, 7) do
						local off = (i % 2 == 1)
						fl.light.Enabled = not off
						fl.pan.Material = off and M.SmoothPlastic or M.Neon
						fl.pan.Color = off and OFFC or ONC
						task.wait(rng:NextNumber(0.04, 0.15))
					end
					fl.light.Enabled = true
					fl.pan.Material = M.Neon
					fl.pan.Color = ONC
				end
			end)
		end

		-- =============== SPAWNS ===============
		local sp = { Alpha = {}, Bravo = {}, FFA = {} }
		for _, p in ipairs({ { -70, -9 }, { -70, -3 }, { -70, 3 }, { -70, 9 }, { -65, 0 } }) do
			table.insert(sp.Alpha, Vector3.new(p[1], 0, p[2]))
			table.insert(sp.Bravo, Vector3.new(-p[1], 0, p[2]))
		end
		for _, p in ipairs({ { 15, -12 }, { -15, -12 }, { 15, 12 }, { -15, 12 }, { 15, 40 }, { -15, 40 }, { 15, -40 }, { -15, -40 },
			{ 0, -22 }, { 0, 22 }, { -64, -38 }, { 64, 36 }, { -64, 36 }, { 64, -38 } }) do
			table.insert(sp.FFA, Vector3.new(p[1], 0, p[2]))
		end
		setSpawns(sp)
		return { spawns = sp, half = 80 }
	end

	-- If ServerStorage.CustomMap_GRID (your hand-edited copy of the office) has parts in it, that is used; otherwise the built-in office is generated.
	MapBuilders.GRID = function(f)
		local custom = game:GetService("ServerStorage"):FindFirstChild("CustomMap_GRID")
		if custom and #custom:GetChildren() > 0 then return MapBuilders.GRID_PG(f) end
		return MapBuilders.GRID_OFFICE(f)
	end

	local function buildMap(id)
		MapFolder:ClearAllChildren()
		-- every map is small: keep all of it (and the horizon backdrop) streamed in so nothing pops in or shows the baseplate
		pcall(function()
			Workspace.StreamingTargetRadius = 2048
			Workspace.StreamingMinRadius = 1024
		end)
		local info = MapBuilders[id](MapFolder)
		boundary(MapFolder, info.half, 90)
		return info
	end

	-- =================================================================
	-- MATCH STATE + RECORDS (players and bots share one record type)
	-- =================================================================
	local Match = {
		phase = "LOBBY", mode = nil, mapId = nil, mapInfo = nil, list = {}, records = {}, byModel = {}, bombers = {},
		score = { Alpha = 0, Bravo = 0 }, over = false, winner = nil, vote = nil, botCount = 0,
	}
	local BOT_NAMES = { "RAVEN", "KILO", "SPECTRE", "VIPER", "NOMAD", "HAVOC", "ECHO", "TITAN", "BISHOP", "ONYX", "COBRA", "GHOST" }
	local BOT_WEAPONS = { "VX44", "HEX12", "KR9", "THD" }

	local function newRecord(inst, name, team, isBot)
		local rec = {
			inst = inst, name = name, team = team, isBot = isBot, kills = 0, deaths = 0, assists = 0, headshots = 0,
			shots = 0, hits = 0, streak = 0, xpGain = 0, alive = false, spawnProtectUntil = 0, lastDamaged = 0,
			hitLog = {}, gg = 1, ids = { "VX44", "MK2" }, ammo = { { mag = 30, res = 0 }, { mag = 12, res = 0 } }, slot = 1,
			nextFire = 0, reloadUntil = 0, swapUntil = 0, burst = 0, reloadToken = 0, meleeAt = 0, left = false,
		}
		Match.records[inst] = rec
		table.insert(Match.list, rec)
		return rec
	end

	local function recRoot(rec)
		local m = rec.model
		return m and m:FindFirstChild("HumanoidRootPart")
	end
	local function isEnemy(a, b)
		if a == b then return false end
		if Match.mode and Match.mode.teams then return a.team ~= b.team end
		return true
	end
	-- killcam recorder: last ~4 seconds of every fighter's position and facing, sampled 10 times a second
	task.spawn(function()
		while true do
			task.wait(0.1)
			if Match.phase == "MATCH" then
				for _, r in ipairs(Match.list) do
					local rt = recRoot(r)
					if r.alive and rt then
						local tr = r.trail
						if not tr then tr = {} r.trail = tr end
						table.insert(tr, { rt.Position, rt.CFrame.LookVector })
						if #tr > 42 then table.remove(tr, 1) end
					end
				end
			end
		end
	end)
	local function packTrail(tr, n)
		local out = {}
		if not tr then return out end
		for i = math.max(1, #tr - n + 1), #tr do
			local e = tr[i]
			table.insert(out, { math.floor(e[1].X * 10) / 10, math.floor(e[1].Y * 10) / 10, math.floor(e[1].Z * 10) / 10, math.floor(e[2].X * 100) / 100, math.floor(e[2].Z * 100) / 100 })
		end
		return out
	end
	local function scoreOf(rec) return rec.kills * 100 + rec.assists * 50 + rec.headshots * 25 end

	local function syncStats(rec)
		if rec.isBot then return end
		local ls = rec.inst:FindFirstChild("leaderstats")
		if ls then
			if ls:FindFirstChild("Kills") then ls.Kills.Value = rec.kills end
			if ls:FindFirstChild("Deaths") then ls.Deaths.Value = rec.deaths end
		end
	end

	local function sendAmmo(rec, why)
		if rec.isBot or not rec.inst.Parent then return end
		Net.Ammo:FireClient(rec.inst, {
			slot = rec.slot, ids = rec.ids, why = why,
			att1 = rec.eff and rec.eff[1].att, att2 = rec.eff and rec.eff[2].att,
			a = { { rec.ammo[1].mag, rec.ammo[1].res }, { rec.ammo[2].mag, rec.ammo[2].res } },
		})
	end

	local function giveLoadout(rec)
		if Match.mode and Match.mode.id == "GUN" then
			rec.ids = { GUNGAME_LADDER[math.clamp(rec.gg, 1, #GUNGAME_LADDER)], "MK2" }
			if rec.ids[1] == "MK2" then rec.ids[2] = "VPR9" end
		elseif rec.isBot then
			rec.ids = { BOT_WEAPONS[math.random(1, #BOT_WEAPONS)], "MK2" }
		else
			local p = Profiles[rec.inst]
			local lo = p and p.loadout
			local lvl = levelOf(rec.inst)
			local pri = lo and lo.primary or "VX44"
			local sec = lo and lo.secondary or "MK2"
			local own = p and p.owned or {}
			local function ok(w) return w and (w.unlock <= lvl or (w.premium and own["W_" .. w.id])) end
			if not ok(WEAPONS[pri]) then pri = "VX44" end
			if not ok(WEAPONS[sec]) then sec = "MK2" end
			rec.ids = { pri, sec }
		end
		rec.eff = {}
		do
			local lo2 = (not rec.isBot) and Profiles[rec.inst] and Profiles[rec.inst].loadout
			local lvl2 = (not rec.isBot) and levelOf(rec.inst) or 1
			for i = 1, 2 do
				local base = WEAPONS[rec.ids[i]]
				local own2 = (not rec.isBot) and Profiles[rec.inst] and Profiles[rec.inst].owned or {}
				local list = (lo2 and lo2.att and not (Match.mode and Match.mode.id == "GUN")) and cleanAttach(lo2.att[rec.ids[i]], lvl2, own2, rec.ids[i]) or nil
				rec.eff[i] = effDef(base, list)
			end
		end
		for i = 1, 2 do
			local d = rec.eff[i]
			rec.ammo[i] = { mag = d.mag, res = d.reserve }
		end
		rec.frags, rec.stims = 2, 2
		if not rec.isBot and rec.inst then
			rec.inst:SetAttribute("Frags", 2)
			rec.inst:SetAttribute("Stims", 2)
			rec.inst:SetAttribute("StimUntil", 0)
			rec.inst:SetAttribute("Airstrikes", 0)
			rec.airstrikes = 0
			rec.inst:SetAttribute("Bombers", 0)
			rec.bombers = 0
			rec.inst:SetAttribute("UavUntil", 0)
		end
		rec.slot = 1
		rec.nextFire = 0
		rec.reloadUntil = 0
		rec.swapUntil = os.clock() + 0.4
		rec.burst = 0
		rec.reloadToken += 1
	end

	local function attachWorldGun(rec)
		local model = rec.model
		if not model then return end
		local old = model:FindFirstChild("BO_Gun")
		if old then old:Destroy() end
		if model:GetAttribute("IsBot") then
			local torso = model:FindFirstChild("Torso")
			local rootP = model.PrimaryPart
			if not torso or not rootP then return end
			local gdef = (rec.eff and rec.eff[rec.slot]) or WEAPONS[rec.ids[rec.slot]]
			if not rec.isBot and Profiles[rec.inst] and Profiles[rec.inst].equip then gdef = camoDef(gdef, Profiles[rec.inst].equip.camo) end
			local bg = buildGunModel(gdef, false)
			bg.Parent = model
			bg:PivotTo(rootP.CFrame * CFrame.new(0.25, 0.55, -2.1))
			local bw = Instance.new("WeldConstraint")
			bw.Part0 = torso
			bw.Part1 = bg.PrimaryPart
			bw.Parent = bg.PrimaryPart
			return
		end
		local hand = model:FindFirstChild("RightHand") or model:FindFirstChild("Right Arm") or model:FindFirstChild("RightArm")
		if not hand then return end
		local def = rec.eff[rec.slot]
		local g = buildGunModel(def, false)
		g.Parent = model
		g:PivotTo(hand.CFrame * CFrame.new(0, -0.4, -1.0))
		local w = Instance.new("WeldConstraint")
		w.Part0 = hand
		w.Part1 = g.PrimaryPart
		w.Parent = g.PrimaryPart
	end

	local function giveXp(rec, amount, reason)
		if rec.isBot or amount <= 0 then return end
		local p = Profiles[rec.inst]
		if not p then return end
		amount = math.floor(amount)
		local before = levelFromXp(p.xp)
		p.xp += amount
		rec.xpGain += amount
		local after = levelFromXp(p.xp)
		Net.Xp:FireClient(rec.inst, amount, reason)
		if after > before then
			Net.LevelUp:FireClient(rec.inst, after)
			rec.inst:SetAttribute("Level", after)
			local unlocked = {}
			for _, w in pairs(WEAPONS) do
				if w.unlock > before and w.unlock <= after then table.insert(unlocked, w.name) end
			end
			if #unlocked > 0 then Net.Banner:FireClient(rec.inst, "UNLOCKED: " .. table.concat(unlocked, ", "), "good") end
			sendProfile(rec.inst)
		end
	end

	-- spawn selection: far from enemies, with a little randomness
	local function pickSpawn(rec)
		local pts = (Match.mode.teams and Match.mapInfo.spawns[rec.team]) or Match.mapInfo.spawns.FFA
		local scored = {}
		for _, p in ipairs(pts) do
			local minD = 1e9
			for _, o in ipairs(Match.list) do
				if o.alive and isEnemy(rec, o) then
					local r = recRoot(o)
					if r then minD = math.min(minD, (r.Position - p).Magnitude) end
				end
			end
			table.insert(scored, { p = p, d = minD + math.random() * 15 })
		end
		table.sort(scored, function(a, b) return a.d > b.d end)
		return scored[math.random(1, math.min(rec.isBot and 5 or 2, #scored))].p
	end
	local function placeAt(model, pos)
		local y = pos.Y + 3.2
		model:PivotTo(CFrame.lookAt(Vector3.new(pos.X, y, pos.Z), Vector3.new(0, y, 0)))
		local root = model:FindFirstChild("HumanoidRootPart")
		if root then root.AssemblyLinearVelocity = Vector3.zero end
	end

	-- =================================================================
	-- DAMAGE, DEATH, SCORING
	-- =================================================================
	local function falloff(def, dist)
		if dist <= def.rNear then return def.dmg end
		if dist >= def.rFar then return def.dmgFar end
		return def.dmg + (def.dmgFar - def.dmg) * ((dist - def.rNear) / (def.rFar - def.rNear))
	end
	local function partMult(def, name)
		if name == "Head" or name:find("Helmet") or name == "Visor" then return def.headMult, true end
		if name:find("Arm") or name:find("Leg") or name:find("Hand") or name:find("Foot") then return 0.85, false end
		return 1, false
	end
	local function spreadDir(dir, deg)
		if deg <= 0.001 then return dir end
		local ang = math.rad(deg) * math.sqrt(math.random())
		local roll = math.random() * math.pi * 2
		local cf = CFrame.lookAt(Vector3.zero, dir) * CFrame.Angles(0, 0, roll) * CFrame.Angles(ang, 0, 0)
		return cf.LookVector
	end

	local function broadcastFeed(killer, victim, weaponId, headshot)
		local wname = ""
		if weaponId == "MELEE" then wname = "KNIFE" elseif weaponId == "FRAG" then wname = "FRAG" elseif weaponId == "AIRSTRIKE" then wname = "AIRSTRIKE" elseif WEAPONS[weaponId or ""] then wname = WEAPONS[weaponId].name end
		Net.Feed:FireAllClients({ k = killer and killer.name or "", kt = killer and killer.team or "", v = victim.name,
			vt = victim.team, w = wname, h = headshot == true })
	end

	local spawnBot -- forward
	local bomberExplode -- forward (suicide bomber scorestreak)
	local function onDied(rec)
		if rec.bomber then bomberExplode(rec) return end
		if not rec.alive then return end
		rec.alive = false
		rec.deaths += 1
		local now = os.clock()
		local killer = rec.lastHitBy
		if not killer or killer == rec or killer.left or now - (rec.lastHitAt or 0) > 8 then killer = nil end
		local assists = {}
		for r, t in pairs(rec.hitLog) do
			if r ~= killer and r ~= rec and not r.left and now - t < 8 then table.insert(assists, r) end
		end
		rec.hitLog = {}
		rec.streak = 0
		rec.airstrikes = 0
		rec.bombers = 0
		if not rec.isBot and rec.inst.Parent then rec.inst:SetAttribute("Airstrikes", 0) rec.inst:SetAttribute("Bombers", 0) end
		local head = rec.lastHead == true
		rec.diedAt = os.clock()
		-- tell the player straight away (before any scoring code that could error) so the respawn countdown always shows
		if not rec.isBot and rec.inst.Parent then
			pcall(function()
				local w0 = WEAPONS[rec.lastWeapon or ""]
				local wn = rec.lastWeapon == "MELEE" and "KNIFE" or (rec.lastWeapon == "FRAG" and "FRAG") or (rec.lastWeapon == "AIRSTRIKE" and "AIRSTRIKE") or (rec.lastWeapon == "BOMBER" and "BOMBER") or (w0 and w0.name or "")
				local kc
				if killer then
					local kpf = (not killer.isBot) and Profiles[killer.inst] or nil
					kc = { k = packTrail(killer.trail, 36), v = packTrail(rec.trail, 36), w = rec.lastWeapon, kn = killer.name, kt = killer.team, hs = head,
						ko = kpf and kpf.equip and kpf.equip.operator or nil }
				end
				Net.Died:FireClient(rec.inst, killer and killer.name or "", wn, head, CFG.RESPAWN_TIME, kc)
			end)
		end
		if killer then
			killer.kills += 1
			killer.streak += 1
			if head then killer.headshots += 1 end
			local mult = rec.isBot and CFG.BOT_XP_MULT or 1
			giveXp(killer, (CFG.XP_KILL + (head and CFG.XP_HEADSHOT or 0)) * mult, head and "HEADSHOT" or "ELIMINATION")
			for _, a in ipairs(assists) do
				a.assists += 1
				giveXp(a, CFG.XP_ASSIST * mult, "ASSIST")
				syncStats(a)
			end
			if not killer.isBot then
				Net.Hit:FireClient(killer.inst, "kill")
				if killer.streak == 2 then
					local pf = Profiles[killer.inst]
					if pf and pf.equip and pf.equip.streak == "BOMBER" then
						killer.bombers = (killer.bombers or 0) + 1
						killer.inst:SetAttribute("Bombers", killer.bombers)
						Net.Banner:FireClient(killer.inst, "SUICIDE BOMBER READY", "good")
					else
						killer.airstrikes = (killer.airstrikes or 0) + 1
						killer.inst:SetAttribute("Airstrikes", killer.airstrikes)
						Net.Banner:FireClient(killer.inst, "AIRSTRIKE READY", "good")
					end
				end
				if killer.streak == 3 or killer.streak == 5 or killer.streak == 7 or killer.streak == 10 then
					Net.Banner:FireClient(killer.inst, killer.streak .. " KILL STREAK", "streak")
				end
				-- killstreak rewards: 3 = UAV, 5 = SUPER UAV (radar + see enemies through walls), 7 = resupply, 10 = long SUPER UAV
				local k = killer.streak
				if k == 3 or k == 5 or k == 7 or k == 10 then
					local dur, tier, label
					if k == 3 then dur, tier, label = 20, 1, "UAV ONLINE"
					elseif k == 5 then dur, tier, label = 30, 2, "SUPER UAV ONLINE"
					elseif k == 10 then dur, tier, label = 45, 2, "SUPER UAV ONLINE"
					end
					if dur then
						for _, o in ipairs(Match.list) do
							if not o.isBot and not o.left and o.alive and (o == killer or (Match.mode and Match.mode.teams and o.team == killer.team)) then
								o.inst:SetAttribute("UavTier", tier)
								o.inst:SetAttribute("UavUntil", Workspace:GetServerTimeNow() + dur)
								Net.Banner:FireClient(o.inst, o == killer and label or ("TEAM " .. label), "good")
							end
						end
					else
						killer.frags, killer.stims = 2, 2
						killer.inst:SetAttribute("Frags", 2)
						killer.inst:SetAttribute("Stims", 2)
						Net.Banner:FireClient(killer.inst, "RESUPPLY: FRAG + STIM", "good")
					end
				end
			end
			syncStats(killer)
			if Match.phase == "MATCH" and not Match.over then
				local mode = Match.mode
				if mode.teams then
					Match.score[killer.team] += 1
					setState("ScoreAlpha", Match.score.Alpha)
					setState("ScoreBravo", Match.score.Bravo)
					if Match.score[killer.team] >= mode.limit then
						Match.winner = { team = killer.team }
						Match.over = true
					end
				elseif mode.id == "FFA" then
					if killer.kills >= mode.limit then
						Match.winner = { rec = killer }
						Match.over = true
					end
				else
					killer.gg += 1
					if killer.gg > #GUNGAME_LADDER then
						Match.winner = { rec = killer }
						Match.over = true
					elseif killer.alive then
						giveLoadout(killer)
						attachWorldGun(killer)
						sendAmmo(killer, "gg")
						if not killer.isBot then
							Net.Banner:FireClient(killer.inst, "WEAPON UP: " .. WEAPONS[killer.ids[1]].name, "good")
						end
					end
				end
			end
		end
		syncStats(rec)
		broadcastFeed(killer, rec, rec.lastWeapon, head)
		if rec.isBot then
			local m = rec.model
			if m then
				Match.byModel[m] = nil
				-- rigid topple: freeze the body and tip it over smoothly (no exploding limbs)
				for _, p in ipairs(m:GetDescendants()) do
					if p:IsA("BasePart") then p.Anchored = true; p.CanCollide = false end
				end
				local ai0 = rec.ai
				if ai0 and ai0.align then pcall(function() ai0.align:Destroy() end) end
				task.spawn(function()
					local ok0, start = pcall(function() return m:GetPivot() end)
					if not ok0 then return end
					local dir = (math.random() < 0.5) and 1 or -1
					local t0 = os.clock()
					while m.Parent and os.clock() - t0 < 0.55 do
						local a = math.min(1, (os.clock() - t0) / 0.55)
						a = a * a
						local rot = start * CFrame.new(0, -a * 1.6, 0) * CFrame.Angles(-a * math.rad(88), 0, dir * a * math.rad(10))
						pcall(function() m:PivotTo(rot) end)
						task.wait()
					end
				end)
				Debris:AddItem(m, 3)
			end
			task.delay(CFG.RESPAWN_TIME + 1, function()
				if not rec.alive and not rec.left and Match.phase == "MATCH" and Match.records[rec.inst] == rec then
					spawnBot(rec)
				end
			end)
		end
	end

	local function applyDamage(attacker, victim, amount, headshot, weaponId, fromPos)
		if Match.mode and Match.mode.id == "FREE" then return false end -- free roam: nobody can be hurt
		if not victim.alive or not victim.hum or victim.hum.Health <= 0 then return false end
		if os.clock() < victim.spawnProtectUntil then return false end
		local now = os.clock()
		victim.lastDamaged = now
		victim.lastHitBy = attacker
		victim.lastHitAt = now
		victim.hitLog[attacker] = now
		victim.lastWeapon = weaponId
		victim.lastHead = headshot
		victim.hum.Health = math.max(0, victim.hum.Health - math.max(1, amount))
		if not attacker.isBot then
			attacker.hits += 1
			Net.Hit:FireClient(attacker.inst, headshot and "head" or "body")
		end
		if not victim.isBot and fromPos and victim.inst.Parent then Net.Dmg:FireClient(victim.inst, fromPos) end
		if victim.hum.Health <= 0 then
			onDied(victim)
			return true
		end
		return false
	end

	-- =================================================================
	-- SHOOTING (server authoritative)
	-- =================================================================
	local function doFire(rec, def, origin, dir, ads, extraSpread)
		local root = recRoot(rec)
		local spread = (ads and def.ads or def.hip) + (extraSpread or 0)
		if root then
			local sp = root.AssemblyLinearVelocity.Magnitude
			if sp > 4 then spread += def.moveSpread * math.min(1, sp / 16) end
		end
		if rec.hum and rec.hum.FloorMaterial == Enum.Material.Air then spread += 2.5 end
		local params = RaycastParams.new()
		params.FilterType = Enum.RaycastFilterType.Exclude
		params.FilterDescendantsInstances = { rec.model }
		params.IgnoreWater = true
		local maxRange = def.rFar * 2
		local ends = {}
		local dmgMult = rec.isBot and CFG.BOT_DAMAGE or 1
		for _ = 1, (def.pellets or 1) do
			local d = spreadDir(dir, spread)
			local res = Workspace:Raycast(origin, d * maxRange, params)
			if res then
				table.insert(ends, res.Position)
				local model = res.Instance:FindFirstAncestorOfClass("Model")
				while model and not Match.byModel[model] do
					local up = model.Parent
					model = up and up:FindFirstAncestorOfClass("Model") or nil
				end
				local target = model and Match.byModel[model]
				if target and isEnemy(rec, target) then
					local dist = (res.Position - origin).Magnitude
					local mult, head = partMult(def, res.Instance.Name)
					applyDamage(rec, target, falloff(def, dist) * mult * dmgMult, head, def.id, origin)
				end
			else
				table.insert(ends, origin + d * maxRange)
			end
		end
		if rec.isBot and rec.model then rec.model:SetAttribute("Shot", ((rec.model:GetAttribute("Shot") or 0) + 1) % 1000) end
		for _, pl in ipairs(Players:GetPlayers()) do
			if pl ~= rec.inst then Net.ShotFx:FireClient(pl, def.id, origin, ends) end
		end
	end

	Net.Fire.OnServerEvent:Connect(function(player, origin, dir, ads)
		local rec = Match.records[player]
		if not rec or not rec.alive or Match.phase ~= "MATCH" or not rec.model then return end
		if typeof(origin) ~= "Vector3" or typeof(dir) ~= "Vector3" then return end
		if origin ~= origin or dir ~= dir or dir.Magnitude < 0.5 or dir.Magnitude > 1.5 then return end
		local now = os.clock()
		if now < rec.swapUntil - 0.15 or now < rec.reloadUntil - 0.15 then return end
		local def = rec.eff[rec.slot]
		local a = rec.ammo[rec.slot]
		local interval = 60 / def.rpm
		if now < rec.nextFire - 0.08 then return end
		if a.mag <= 0 then
			if rateOk(player, "deny", 0.3) then sendAmmo(rec, "empty") end
			return
		end
		a.mag -= 1
		rec.shots += 1
		if def.burst then
			rec.burst += 1
			if rec.burst >= def.burst then
				rec.burst = 0
				rec.nextFire = now + def.burstDelay
			else
				rec.nextFire = now + interval
			end
		else
			rec.nextFire = now + interval
		end
		local head = rec.model:FindFirstChild("Head")
		if head then
			local off = origin - head.Position
			if off.Magnitude > 10 then
				origin = head.Position
			elseif off.Magnitude > 0.2 then
				local p = RaycastParams.new()
				p.FilterType = Enum.RaycastFilterType.Exclude
				p.FilterDescendantsInstances = { rec.model }
				if Workspace:Raycast(head.Position, off, p) then origin = head.Position end
			end
		end
		doFire(rec, def, origin, dir.Unit, ads == true, 0)
	end)

	Net.Reload.OnServerEvent:Connect(function(player)
		local rec = Match.records[player]
		if not rec or not rec.alive or Match.phase ~= "MATCH" then return end
		local now = os.clock()
		if now < rec.reloadUntil or now < rec.swapUntil then return end
		local def = rec.eff[rec.slot]
		local a = rec.ammo[rec.slot]
		if a.mag >= def.mag or a.res <= 0 then return end
		local dur = (a.mag == 0) and def.reloadEmpty or def.reload
		rec.reloadUntil = now + dur
		rec.reloadToken += 1
		local tok, slot = rec.reloadToken, rec.slot
		task.delay(dur, function()
			if rec.reloadToken ~= tok or not rec.alive or rec.slot ~= slot then return end
			local take = math.min(def.mag - a.mag, a.res)
			a.mag += take
			a.res -= take
			sendAmmo(rec, "reloaded")
		end)
	end)

	-- =================================================================
	-- EQUIPMENT: frag grenades + stims (server validated)
	-- =================================================================
	local function fragBlast(rec, pos)
		local ex = Instance.new("Explosion")
		ex.BlastPressure = 0
		ex.BlastRadius = 0
		ex.DestroyJointRadiusPercent = 0
		ex.Position = pos
		ex.Parent = Workspace
		for _, pl in ipairs(Players:GetPlayers()) do Net.ShotFx:FireClient(pl, "FRAG", pos, { pos }) end
		for _, o in ipairs(Match.list) do
			if o.alive then
				local r2 = recRoot(o)
				if r2 and (o == rec or isEnemy(rec, o)) then
					local c = r2.Position + Vector3.new(0, 1, 0)
					local d = (c - pos).Magnitude
					if d < 17 then
						local rp = RaycastParams.new()
						rp.FilterType = Enum.RaycastFilterType.Exclude
						rp.FilterDescendantsInstances = { o.model }
						local hit = Workspace:Raycast(pos + Vector3.new(0, 0.5, 0), c - pos, rp)
						if not hit then
							local dmg = 140 * (1 - d / 17)
							if o == rec then dmg *= 0.4 end
							if dmg > 4 then applyDamage(rec, o, dmg, false, "FRAG", pos) end
						end
					end
				end
			end
		end
	end

	Net.Throw.OnServerEvent:Connect(function(player, origin, dir)
		local rec = Match.records[player]
		if not rec or not rec.alive or Match.phase ~= "MATCH" or (rec.frags or 0) <= 0 then return end
		if typeof(origin) ~= "Vector3" or typeof(dir) ~= "Vector3" or dir.Magnitude < 0.1 then return end
		if not rateOk(player, "throw", 0.9) then return end
		local root = recRoot(rec)
		if not root then return end
		if (origin - root.Position).Magnitude > 8 then origin = root.Position + Vector3.new(0, 1.5, 0) end
		rec.frags -= 1
		player:SetAttribute("Frags", rec.frags)
		local g = Instance.new("Part")
		g.Name = "BO_Frag"
		g.Shape = Enum.PartType.Ball
		g.Size = Vector3.new(0.9, 0.9, 0.9)
		g.Color = RGB(52, 60, 44)
		g.Material = Enum.Material.Metal
		g.CustomPhysicalProperties = PhysicalProperties.new(0.7, 0.6, 0.35, 1, 1)
		g.CanTouch = false
		g.CFrame = CFrame.new(origin + dir.Unit * 1.2)
		g.AssemblyLinearVelocity = dir.Unit * 62 + Vector3.new(0, 14, 0) + root.AssemblyLinearVelocity * 0.5
		g.Parent = Workspace
		pcall(function() g:SetNetworkOwner(nil) end)
		local light = Instance.new("PointLight")
		light.Color = RGB(255, 60, 40); light.Range = 6; light.Brightness = 1.5; light.Parent = g
		task.delay(2.3, function()
			if g.Parent then
				local pos = g.Position
				g:Destroy()
				if Match.phase == "MATCH" then fragBlast(rec, pos) end
			end
		end)
	end)


	-- ---------- scorestreak: AIRSTRIKE (2 kills): a missile falls from the sky, the player steers it with the mouse, it always detonates ----------
	local strikeBusy = {}
	local strikeSteer = {}
	local STRIKE_KILL_R = 44
	local function runAirstrike(rec, player, yaw0, pitch0)
		local models = {}
		for _, o in ipairs(Match.list) do if o.model then table.insert(models, o.model) end end
		-- the missile only ever collides with the BLACKOUT map itself: stray parts in the place (baseplate, terrain, sky props, invisible
		-- walls) can no longer stop it in mid-air, and characters are never "hit" so it keeps falling to the ground
		local rp = RaycastParams.new()
		rp.FilterType = Enum.RaycastFilterType.Include
		rp.FilterDescendantsInstances = { MapFolder }
		local folder = Instance.new("Folder")
		folder.Name = "BO_Strike"
		folder.Parent = Workspace
		Debris:AddItem(folder, 16)
		local function mk(size, col, tr, cf, shape, mat, parent)
			local q = Instance.new("Part")
			q.Anchored, q.CanCollide, q.CanQuery, q.CanTouch, q.CastShadow = true, false, false, false, false
			q.Material = mat or Enum.Material.Neon
			q.Color, q.Transparency, q.Size, q.CFrame = col, tr, size, cf
			if shape then q.Shape = shape end
			q.Parent = parent or folder
			return q
		end
		local function snd(pos, id, speed, vol)
			local h = mk(Vector3.new(1, 1, 1), Color3.new(), 1, CFrame.new(pos))
			local sd = Instance.new("Sound")
			sd.SoundId, sd.PlaybackSpeed, sd.Volume = id, speed, vol
			sd.RollOffMode = Enum.RollOffMode.Linear
			sd.RollOffMinDistance, sd.RollOffMaxDistance = 80, 700
			sd.Parent = h
			sd:Play()
		end
		local red, hot = Color3.fromRGB(255, 40, 30), Color3.fromRGB(255, 170, 60)
		local half = (Match.mapInfo and Match.mapInfo.half or 120) - 6
		local r0 = recRoot(rec)
		local base = r0 and r0.Position or Vector3.zero
		local halfX = ((Match.mapInfo and Match.mapInfo.halfX) or (half + 6)) - 6
		local pos = Vector3.new(math.clamp(base.X, -halfX, halfX), 650, math.clamp(base.Z, -half, half))
		local steer = { yaw = yaw0, pitch = pitch0 }
		strikeSteer[player] = steer
		local yaw, pitch = yaw0, pitch0

		-- the missile (the client camera rides on the part named BO_Missile_<userId>)
		local mis = mk(Vector3.new(3, 3, 14), hot, 0, CFrame.new(pos), nil, nil, Workspace)
		mis.Name = "BO_Missile_" .. player.UserId
		Debris:AddItem(mis, 16)
		local core = mk(Vector3.new(1.6, 1.6, 15), Color3.new(1, 1, 1), 0, CFrame.new(pos))
		local trail = mk(Vector3.new(5, 5, 90), red, 0.45, CFrame.new(pos))
		local lt = Instance.new("PointLight")
		lt.Color, lt.Range, lt.Brightness = hot, 60, 6
		lt.Parent = mis
		snd(pos, SND.lunge, 0.55, 5)
		-- with streaming on, the client only receives parts near its focus: follow the missile so it never vanishes mid-flight
		pcall(function() player.ReplicationFocus = mis end)
		player:SetAttribute("StrikeLive", true)

		local t0, last = os.clock(), os.clock()
		local impact
		local SPEED, TURN = 115, 1.9
		local okFly, errFly = pcall(function()
			while true do
				RunService.Heartbeat:Wait()
				local now = os.clock()
				local dt = math.min(now - last, 0.1)
				last = now
				if Match.phase ~= "MATCH" then return end
				local dyaw = ((steer.yaw - yaw + math.pi) % (2 * math.pi)) - math.pi
				yaw += math.clamp(dyaw, -TURN * dt, TURN * dt)
				local wantP = (now - t0 > 6) and math.rad(-85) or steer.pitch -- after 6 s the missile always dives, so it can never time out in the air
				pitch += math.clamp(wantP - pitch, -TURN * dt, TURN * dt)
				local dir = Vector3.new(-math.sin(yaw) * math.cos(pitch), math.sin(pitch), -math.cos(yaw) * math.cos(pitch))
				local step = dir * (SPEED * (1 + (now - t0) * 0.15) * dt)
				local np = pos + step
				np = Vector3.new(math.clamp(np.X, -halfX, halfX), np.Y, math.clamp(np.Z, -half, half))
				local seg = np - pos
				local hit = Workspace:Raycast(pos, (seg.Magnitude > 1e-4 and seg.Unit or dir) * (seg.Magnitude + 1.5), rp)
				if hit and hit.Instance.CanCollide and hit.Position.Y < 100 and hit.Instance.Name ~= "Ceiling" and hit.Instance.Name ~= "Bound" then impact = hit.Position return end
				if np.Y <= 0.5 then impact = Vector3.new(np.X, 0.5, np.Z) return end
				pos = np
				local cf = CFrame.lookAt(pos, pos + dir)
				mis.CFrame, core.CFrame = cf, cf
				trail.CFrame = cf * CFrame.new(0, 0, 52)
				if now - t0 > 12 then impact = Vector3.new(pos.X, 0.5, pos.Z) return end
			end
		end)
		if not okFly then warn("[STRIKE] flight error: " .. tostring(errFly)) end
		strikeSteer[player] = nil
		pcall(function() player.ReplicationFocus = nil end)
		player:SetAttribute("StrikeLive", false)
		impact = impact or Vector3.new(pos.X, 0.5, pos.Z)
		-- always land ON the ground: find the real surface under wherever the missile ended (never explode in mid-air)
		do
			local from = (impact.Y > 5) and impact or Vector3.new(impact.X, math.max(pos.Y, 8), impact.Z)
			local gr = Workspace:Raycast(from + Vector3.new(0, 3, 0), Vector3.new(0, -1500, 0), rp)
			if gr then impact = gr.Position end
		end
		mis:Destroy() core:Destroy() trail:Destroy()
		-- impact: loud, bright, obvious, with a ring that shows the kill radius
		local ex = Instance.new("Explosion")
		ex.BlastPressure, ex.BlastRadius, ex.DestroyJointRadiusPercent = 0, 24, 0
		ex.Position = impact
		ex.Parent = Workspace
		snd(impact, SND.bass, 0.45, 10)
		snd(impact, SND.collide, 0.4, 8)
		snd(impact, SND.flash, 0.5, 6)
		local ball = mk(Vector3.new(8, 8, 8), hot, 0.1, CFrame.new(impact + Vector3.new(0, 4, 0)), Enum.PartType.Ball)
		local wave = mk(Vector3.new(0.6, 8, 8), Color3.fromRGB(255, 230, 200), 0.3, CFrame.new(impact + Vector3.new(0, 0.5, 0)) * CFrame.Angles(0, 0, math.pi / 2), Enum.PartType.Cylinder)
		local pillar = mk(Vector3.new(10, 90, 10), red, 0.4, CFrame.new(impact + Vector3.new(0, 45, 0)))
		local zone = mk(Vector3.new(0.4, STRIKE_KILL_R * 2, STRIKE_KILL_R * 2), red, 0.55, CFrame.new(impact + Vector3.new(0, 0.4, 0)) * CFrame.Angles(0, 0, math.pi / 2), Enum.PartType.Cylinder)
		local fl = Instance.new("PointLight")
		fl.Color, fl.Range, fl.Brightness = Color3.fromRGB(255, 200, 120), 120, 10
		fl.Parent = ball
		task.spawn(function()
			for k = 1, 22 do
				ball.Size = Vector3.new(8, 8, 8) * (1 + k * 0.5)
				ball.Transparency = 0.1 + k * 0.04
				local w = 8 + k * 5
				wave.Size = Vector3.new(0.6, w, w)
				wave.Transparency = 0.3 + k * 0.03
				pillar.Transparency = 0.4 + k * 0.027
				zone.Transparency = 0.55 + k * 0.02
				fl.Brightness = 10 * (1 - k / 22)
				task.wait(0.03)
			end
			ball:Destroy() wave:Destroy() pillar:Destroy() zone:Destroy()
		end)
		-- aftermath: plain ground-level smoke for 10 seconds, then it stops and clears
		task.spawn(function()
			local gp = impact
			local rp = RaycastParams.new()
			rp.FilterType = Enum.RaycastFilterType.Exclude
			rp.FilterDescendantsInstances = { folder }
			local gr = Workspace:Raycast(impact + Vector3.new(0, 30, 0), Vector3.new(0, -200, 0), rp)
			if gr then gp = gr.Position end
			local src = mk(Vector3.new(30, 1, 30), Color3.new(), 1, CFrame.new(gp + Vector3.new(0, 1, 0)), nil, Enum.Material.Slate, Workspace)
			src.Name = "BO_StrikeSmoke"
			-- keep the impact area streamed in for this player so the smoke is actually there to see
			pcall(function() player.ReplicationFocus = src end)
			task.delay(4, function() pcall(function() if player.ReplicationFocus == src then player.ReplicationFocus = nil end end) end)
			local sm = Instance.new("ParticleEmitter")
			sm.Texture = "rbxasset://textures/particles/smoke_main.dds"
			sm.Color = ColorSequence.new(Color3.fromRGB(70, 66, 62), Color3.fromRGB(130, 126, 122))
			sm.Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.2), NumberSequenceKeypoint.new(0.7, 0.45), NumberSequenceKeypoint.new(1, 1) })
			sm.Size = NumberSequence.new({ NumberSequenceKeypoint.new(0, 16), NumberSequenceKeypoint.new(1, 40) })
			sm.Lifetime = NumberRange.new(4, 6)
			sm.Rate = 45
			sm.Speed = NumberRange.new(1, 4)
			sm.SpreadAngle = Vector2.new(60, 60)
			sm.Acceleration = Vector3.new(1.5, 1.2, 0)
			sm.Rotation = NumberRange.new(0, 360)
			sm.RotSpeed = NumberRange.new(-15, 15)
			sm.Parent = src
			task.wait(10)
			sm.Enabled = false
			task.wait(6.5)
			src:Destroy()
		end)
		if Match.phase ~= "MATCH" then return end
		-- everyone on the other team inside the radius is eliminated
		for _, o in ipairs(Match.list) do
			if o.alive and not o.left and o ~= rec and isEnemy(rec, o) then
				local r2 = recRoot(o)
				if r2 and Vector3.new(r2.Position.X - impact.X, 0, r2.Position.Z - impact.Z).Magnitude <= STRIKE_KILL_R and math.abs(r2.Position.Y - impact.Y) <= 90 then
					applyDamage(rec, o, 1000, false, "AIRSTRIKE", impact + Vector3.new(0, 60, 0))
				end
			end
		end
	end

	Net.Airstrike.OnServerEvent:Connect(function(player, cmd, a, b)
		if cmd == "steer" then
			local st = strikeSteer[player]
			if st and type(a) == "number" and type(b) == "number" and a == a and b == b then
				st.yaw = a
				st.pitch = math.clamp(b, math.rad(-85), math.rad(-45))
			end
			return
		end
		if cmd ~= "launch" then return end
		local rec = Match.records[player]
		if not rec or not rec.alive or Match.phase ~= "MATCH" or (rec.airstrikes or 0) <= 0 then return end
		if type(a) ~= "number" or type(b) ~= "number" or a ~= a or b ~= b then return end
		if strikeBusy[player] or not rateOk(player, "airstrike", 3) then return end
		strikeBusy[player] = true
		rec.airstrikes -= 1
		player:SetAttribute("Airstrikes", rec.airstrikes)
		Net.Banner:FireClient(player, "FRIENDLY AIRSTRIKE INBOUND", "good")
		task.delay(18, function() strikeBusy[player] = nil end)
		task.spawn(function()
			pcall(runAirstrike, rec, player, a, math.clamp(b, math.rad(-85), math.rad(-45)))
			strikeBusy[player] = nil
			strikeSteer[player] = nil
		end)
	end)

	Net.Stim.OnServerEvent:Connect(function(player)
		local rec = Match.records[player]
		if not rec or not rec.alive or Match.phase ~= "MATCH" or (rec.stims or 0) <= 0 then return end
		if not rateOk(player, "stim", 2.5) or not rec.hum then return end
		if rec.hum.Health >= rec.hum.MaxHealth and os.clock() < (rec.stimUntil or 0) then return end
		rec.stims -= 1
		player:SetAttribute("Stims", rec.stims)
		rec.hum.Health = math.min(rec.hum.MaxHealth, rec.hum.Health + 55)
		rec.lastDamaged = os.clock() - 3
		rec.stimUntil = os.clock() + 5
		player:SetAttribute("StimUntil", Workspace:GetServerTimeNow() + 5)
	end)

	Net.Swap.OnServerEvent:Connect(function(player, slot)
		local rec = Match.records[player]
		if not rec or not rec.alive or Match.phase ~= "MATCH" then return end
		if slot ~= 1 and slot ~= 2 then return end
		if slot == rec.slot or not rateOk(player, "swap", 0.15) then return end
		rec.slot = slot
		rec.swapUntil = os.clock() + WEAPONS[rec.ids[slot]].switch
		rec.reloadToken += 1
		rec.reloadUntil = 0
		rec.burst = 0
		attachWorldGun(rec)
		sendAmmo(rec, "swap")
	end)

	Net.Melee.OnServerEvent:Connect(function(player)
		local rec = Match.records[player]
		if not rec or not rec.alive or Match.phase ~= "MATCH" then return end
		local now = os.clock()
		if now - rec.meleeAt < CFG.MELEE_COOLDOWN then return end
		rec.meleeAt = now
		local root = recRoot(rec)
		if not root then return end
		local look = root.CFrame.LookVector
		local best, bestD
		for _, o in ipairs(Match.list) do
			if o.alive and isEnemy(rec, o) then
				local r2 = recRoot(o)
				if r2 then
					local off = r2.Position - root.Position
					local d = off.Magnitude
					if d <= CFG.MELEE_RANGE and d > 0.1 and off.Unit:Dot(look) > 0.35 and (not bestD or d < bestD) then
						best, bestD = o, d
					end
				end
			end
		end
		if best then applyDamage(rec, best, CFG.MELEE_DAMAGE, false, "MELEE", root.Position) end
	end)

	-- health regeneration + out-of-bounds
	task.spawn(function()
		while true do
			task.wait(0.2)
			local now = os.clock()
			for _, rec in ipairs(Match.list) do
				local hum = rec.hum
				if rec.alive and hum and hum.Health > 0 and hum.Health < hum.MaxHealth and now - rec.lastDamaged > CFG.REGEN_DELAY then
					hum.Health = math.min(hum.MaxHealth, hum.Health + CFG.REGEN_PER_SEC * 0.2)
				end
			end
		end
	end)
	task.spawn(function()
		while true do
			task.wait(0.5)
			for _, rec in ipairs(Match.list) do
				local root = rec.alive and recRoot(rec)
				if root and root.Position.Y < -30 and rec.hum then rec.hum.Health = 0 end
			end
			for _, pl in ipairs(Players:GetPlayers()) do
				local char = pl.Character
				local root = char and char:FindFirstChild("HumanoidRootPart")
				if root and not Match.records[pl] and root.Position.Y < CFG.HUB_POS.Y - 20 and hubSpawnCF then
					char:PivotTo(hubSpawnCF)
				end
			end
		end
	end)

	-- =================================================================
	-- SCORESTREAK: SUICIDE BOMBER (2 kills, chosen in the loadout instead of the airstrike). A heavily armoured bot spawns by the player,
	-- runs at the nearest enemy and detonates next to them (everything in range dies). If it is shot down, it explodes where it fell.
	-- =================================================================
	local BOMBER_HP, BOMBER_R, BOMBER_LIFE, BOMBER_TRIGGER = 1800, 28, 50, 9
	bomberExplode = function(b)
		if b.exploded then return end
		b.exploded, b.alive = true, false
		local root = recRoot(b)
		local pos = (root and root.Position) or b.lastPos or Vector3.zero
		Match.byModel[b.model] = nil
		for i, x in ipairs(Match.bombers) do
			if x == b then table.remove(Match.bombers, i) break end
		end
		pcall(function() b.model:Destroy() end)
		local ex = Instance.new("Explosion")
		ex.BlastPressure, ex.BlastRadius, ex.DestroyJointRadiusPercent = 0, 14, 0
		ex.Position = pos
		ex.Parent = Workspace
		local holder = Instance.new("Part")
		holder.Anchored, holder.CanCollide, holder.CanQuery, holder.CanTouch, holder.Transparency, holder.Size = true, false, false, false, 1, Vector3.new(1, 1, 1)
		holder.Position = pos
		holder.Parent = Workspace
		Debris:AddItem(holder, 4)
		for _, p in ipairs({ { SND.bass, 0.5, 10 }, { SND.collide, 0.45, 8 }, { SND.flash, 0.55, 6 } }) do
			local sd = Instance.new("Sound")
			sd.SoundId, sd.PlaybackSpeed, sd.Volume = p[1], p[2], p[3]
			sd.RollOffMode = Enum.RollOffMode.Linear
			sd.RollOffMinDistance, sd.RollOffMaxDistance = 60, 500
			sd.Parent = holder
			sd:Play()
		end
		local function glow(size, col, tr, mat, shape)
			local q = Instance.new("Part")
			q.Anchored, q.CanCollide, q.CanQuery, q.CanTouch, q.CastShadow = true, false, false, false, false
			q.Material, q.Color, q.Transparency, q.Size, q.Position = mat or Enum.Material.Neon, col, tr, size, pos
			if shape then q.Shape = shape end
			q.Parent = Workspace
			return q
		end
		local ball = glow(Vector3.new(6, 6, 6), Color3.fromRGB(255, 170, 70), 0.1, nil, Enum.PartType.Ball)
		local ring = glow(Vector3.new(0.6, 6, 6), Color3.fromRGB(255, 230, 200), 0.3, nil, Enum.PartType.Cylinder)
		ring.CFrame = CFrame.new(pos) * CFrame.Angles(0, 0, math.pi / 2)
		local lt = Instance.new("PointLight")
		lt.Color, lt.Range, lt.Brightness = Color3.fromRGB(255, 190, 110), 90, 8
		lt.Parent = ball
		task.spawn(function()
			for k = 1, 16 do
				ball.Size = Vector3.new(6, 6, 6) * (1 + k * 0.55)
				ball.Transparency = 0.1 + k * 0.055
				local w = 6 + k * (BOMBER_R * 2 / 14)
				ring.Size = Vector3.new(0.6, w, w)
				ring.Transparency = 0.3 + k * 0.042
				lt.Brightness = 8 * (1 - k / 16)
				task.wait(0.03)
			end
			ball:Destroy() ring:Destroy()
		end)
		-- everyone on the other side inside the radius dies immediately (credited to the owner)
		local owner = b.owner
		if owner and not owner.left and Match.phase == "MATCH" then
			for _, o in ipairs(Match.list) do
				if o.alive and not o.left and isEnemy(owner, o) then
					local r2 = recRoot(o)
					if r2 and (r2.Position - pos).Magnitude <= BOMBER_R then
						o.spawnProtectUntil = 0
						applyDamage(owner, o, 5000, false, "BOMBER", pos + Vector3.new(0, 5, 0))
					end
				end
			end
		end
	end
	local function bomberBrain(b)
		local hum = b.hum
		local born, lastCheck, blinkAt = os.clock(), os.clock(), 0
		local root = recRoot(b)
		local lastPos = root and root.Position or Vector3.zero
		local path, pathIdx, pathAt = nil, 2, 0
		local sound = root and root:FindFirstChild("BeepSnd")
		while b.alive and b.model.Parent and Match.phase == "MATCH" do
			task.wait(0.12)
			root = recRoot(b)
			if not root or hum.Health <= 0 then break end
			local now = os.clock()
			b.lastPos = root.Position
			if now - born > BOMBER_LIFE or root.Position.Y < -30 then bomberExplode(b) return end
			local tgt, td
			for _, o in ipairs(Match.list) do
				if o.alive and not o.left and isEnemy(b.owner, o) then
					local r2 = recRoot(o)
					if r2 then
						local d = (r2.Position - root.Position).Magnitude
						if not td or d < td then tgt, td = r2, d end
					end
				end
			end
			if tgt then
				if td <= BOMBER_TRIGGER then bomberExplode(b) return end
				if not path or now > pathAt then
					pathAt = now + 1.1
					local pth = PathfindingService:CreatePath({ AgentRadius = 2, AgentHeight = 5, AgentCanJump = true, WaypointSpacing = 6 })
					local ok = pcall(function() pth:ComputeAsync(root.Position, tgt.Position) end)
					if ok and pth.Status == Enum.PathStatus.Success then path, pathIdx = pth:GetWaypoints(), 2 else path = nil end
				end
				local wp = path and path[pathIdx]
				if wp then
					hum:MoveTo(wp.Position)
					if wp.Action == Enum.PathWaypointAction.Jump then hum.Jump = true end
					if Vector3.new(wp.Position.X - root.Position.X, 0, wp.Position.Z - root.Position.Z).Magnitude < 4 then pathIdx += 1 end
				else
					hum:MoveTo(tgt.Position)
				end
				-- beep faster as it closes in
				local gap = math.clamp(td / 90, 0.12, 0.7)
				if now >= blinkAt then
					blinkAt = now + gap
					if sound then sound.PlaybackSpeed = 1.4 + (1 - gap) * 0.8 sound:Play() end
					for _, d in ipairs(b.model:GetChildren()) do
						if d.Name == "BomberLight" then d.Transparency = (d.Transparency > 0.5) and 0 or 0.85 end
					end
				end
			end
			if now - lastCheck > 1.2 then
				if (root.Position - lastPos).Magnitude < 1.5 then hum.Jump = true path = nil end
				lastPos, lastCheck = root.Position, now
			end
		end
		if b.alive then bomberExplode(b) end
	end
	local function spawnBomber(owner)
		local oroot = recRoot(owner)
		if not oroot then return false end
		local color = RGB(255, 50, 40)
		local m = buildBotBody("BOMBER", color, false, RGB(34, 36, 42))
		m:SetAttribute("Team", owner.team)
		m:SetAttribute("IsBot", true)
		m:SetAttribute("Bomber", true)
		m.Parent = BotFolder
		weldBot(m)
		pcall(dressCharacter, m, OPERATOR_ORDER[math.random(1, #OPERATOR_ORDER)], false, color)
		local torso, head = m:FindFirstChild("Torso"), m:FindFirstChild("Head")
		local function plate(n, size, off, col, mat, glowing)
			local p = Instance.new("Part")
			p.Name = n
			p.Size, p.Color, p.Material = size, col, mat or Enum.Material.Metal
			p.CanCollide, p.CanTouch, p.Massless, p.CastShadow = false, false, true, false
			p.CFrame = torso.CFrame * CFrame.new(off)
			p.Parent = m
			local w = Instance.new("WeldConstraint")
			w.Part0, w.Part1 = torso, p
			w.Parent = p
			return p
		end
		local steel, dark = RGB(96, 102, 110), RGB(30, 32, 36)
		plate("ArmorChest", Vector3.new(2.5, 2.1, 0.5), Vector3.new(0, 0.1, -0.7), steel)
		plate("ArmorBack", Vector3.new(2.5, 2.1, 0.5), Vector3.new(0, 0.1, 0.7), steel)
		plate("ArmorSideL", Vector3.new(0.5, 1.8, 1.4), Vector3.new(-1.3, 0, 0), steel)
		plate("ArmorSideR", Vector3.new(0.5, 1.8, 1.4), Vector3.new(1.3, 0, 0), steel)
		plate("ArmorShoulderL", Vector3.new(1.3, 0.5, 1.5), Vector3.new(-1.5, 1.2, 0), dark)
		plate("ArmorShoulderR", Vector3.new(1.3, 0.5, 1.5), Vector3.new(1.5, 1.2, 0), dark)
		plate("ArmorHelmet", Vector3.new(1.5, 0.9, 1.5), Vector3.new(0, 2.15, 0), dark)
		plate("ArmorVisor", Vector3.new(1.3, 0.35, 0.2), Vector3.new(0, 1.75, -0.8), RGB(255, 40, 30), Enum.Material.Neon)
		for k = -1, 1 do
			plate("BombPack", Vector3.new(0.5, 1.3, 0.5), Vector3.new(k * 0.8, -0.2, -1.05), RGB(150, 40, 30))
			local l = plate("BomberLight", Vector3.new(0.35, 0.35, 0.2), Vector3.new(k * 0.8, 0.6, -1.35), RGB(255, 40, 30), Enum.Material.Neon)
			l.Transparency = 0.85
		end
		local hl = Instance.new("Highlight")
		hl.Name = "BO_BotHL"
		hl.FillColor, hl.FillTransparency = Color3.fromRGB(255, 40, 30), 0.82
		hl.OutlineColor, hl.OutlineTransparency = Color3.fromRGB(255, 70, 50), 0
		hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
		hl.Parent = m
		local hum = m:FindFirstChildOfClass("Humanoid")
		hum.MaxHealth, hum.Health, hum.WalkSpeed = BOMBER_HP, BOMBER_HP, 21
		hum.BreakJointsOnDeath = false
		for _, st in ipairs({ Enum.HumanoidStateType.FallingDown, Enum.HumanoidStateType.Ragdoll, Enum.HumanoidStateType.PlatformStanding }) do
			pcall(function() hum:SetStateEnabled(st, false) end)
		end
		local snd = Instance.new("Sound")
		snd.Name = "BeepSnd"
		snd.SoundId = "rbxasset://sounds/electronicpingshort.wav"
		snd.Volume = 1.6
		snd.RollOffMode = Enum.RollOffMode.Linear
		snd.RollOffMinDistance, snd.RollOffMaxDistance = 20, 160
		snd.Parent = m.PrimaryPart
		local fwd = oroot.CFrame.LookVector * Vector3.new(1, 0, 1)
		fwd = (fwd.Magnitude > 0.1) and fwd.Unit or Vector3.new(0, 0, -1)
		placeAt(m, oroot.Position - Vector3.new(0, 3.2, 0) + fwd * 5)
		pcall(function() m.PrimaryPart:SetNetworkOwner(nil) end)
		local b = { bomber = true, isBot = true, name = "BOMBER", team = owner.team, owner = owner, model = m, hum = hum, alive = true, spawnProtectUntil = 0,
			lastDamaged = 0, hitLog = {}, kills = 0, deaths = 0, assists = 0, headshots = 0, streak = 0, hits = 0, shots = 0, left = false }
		Match.byModel[m] = b
		table.insert(Match.bombers, b)
		task.spawn(bomberBrain, b)
		return true
	end
	local function clearBombers()
		for _, b in ipairs(Match.bombers) do
			b.alive, b.exploded = false, true
			pcall(function() b.model:Destroy() end)
		end
		Match.bombers = {}
	end
	Net.Streak.OnServerEvent:Connect(function(pl, cmd)
		if cmd ~= "bomber" then return end
		local rec = Match.records[pl]
		if not rec or not rec.alive or Match.phase ~= "MATCH" or (rec.bombers or 0) <= 0 then return end
		if not rateOk(pl, "bomber", 1.5) then return end
		if not spawnBomber(rec) then return end
		rec.bombers -= 1
		pl:SetAttribute("Bombers", rec.bombers)
		Net.Banner:FireClient(pl, "SUICIDE BOMBER DEPLOYED", "good")
		for _, o in ipairs(Match.list) do
			if not o.isBot and not o.left and isEnemy(rec, o) and o.inst and o.inst.Parent then Net.Banner:FireClient(o.inst, "SUICIDE BOMBER INBOUND", "warn") end
		end
	end)

	-- =================================================================
	-- BOTS
	-- =================================================================
	spawnBot = function(rec)
		if rec.model then
			Match.byModel[rec.model] = nil
			rec.model:Destroy()
		end
		local color = (rec.team == "Alpha" and RGB(255, 140, 30)) or (rec.team == "Bravo" and RGB(0, 200, 255)) or RGB(255, 70, 90)
		local uni = (rec.team == "Alpha" and RGB(104, 94, 64)) or (rec.team == "Bravo" and RGB(64, 82, 112)) or RGB(78, 80, 90)
		local m = buildBotBody(rec.name, color, false, uni)
		m:SetAttribute("Team", rec.team)
		m:SetAttribute("IsBot", true)
		m.Parent = BotFolder
		weldBot(m)
		rec.opId = rec.opId or OPERATOR_ORDER[math.random(1, #OPERATOR_ORDER)]
		do
			local okd, err = pcall(dressCharacter, m, rec.opId, false, color)
			if not okd then warn("BLACKOUT bot dress failed: " .. tostring(err)) end
		end
		do -- thin team-coloured outline so bots are readable even in dim spots
			local hl = Instance.new("Highlight")
			hl.Name = "BO_BotHL"
			hl.FillTransparency = 1
			hl.OutlineColor = color
			hl.OutlineTransparency = 0.45
			hl.DepthMode = Enum.HighlightDepthMode.Occluded
			hl.Parent = m
		end
		pcall(function() m.PrimaryPart:SetNetworkOwner(nil) end) -- server simulates bots: no client-ownership rubber-banding
		rec.model = m
		rec.hum = m:FindFirstChildOfClass("Humanoid")
		Match.byModel[m] = rec
		do -- stable physics: no tripping/ragdoll states, no auto-rotate fighting our facing controller
			local h = rec.hum
			h.AutoRotate = false
			h.BreakJointsOnDeath = false
			h.RequiresNeck = false
			for _, st in ipairs({ Enum.HumanoidStateType.FallingDown, Enum.HumanoidStateType.Ragdoll, Enum.HumanoidStateType.PlatformStanding, Enum.HumanoidStateType.Flying, Enum.HumanoidStateType.Swimming, Enum.HumanoidStateType.Climbing }) do
				pcall(function() h:SetStateEnabled(st, false) end)
			end
		end
		placeAt(m, pickSpawn(rec))
		rec.alive = true
		rec.spawnProtectUntil = os.clock() + CFG.SPAWN_PROTECT
		rec.lastDamaged = 0
		rec.hitLog = {}
		rec.lastHitBy = nil
		giveLoadout(rec)
		attachWorldGun(rec)
		rec.hum.Died:Connect(function() onDied(rec) end)
		local rootP = m.PrimaryPart
		local att = Instance.new("Attachment"); att.Name = "FaceAtt"; att.Parent = rootP
		local align = Instance.new("AlignOrientation")
		align.Mode = Enum.OrientationAlignmentMode.OneAttachment
		align.Attachment0 = att
		align.RigidityEnabled = false
		align.MaxTorque = 1e6
		align.Responsiveness = 18
		align.CFrame = rootP.CFrame.Rotation
		align.Parent = rootP
		rec.ai = { align = align, nextShot = os.clock() + 1, path = nil, pathIdx = 1, pathAt = 0, strafe = (math.random() < 0.5) and 1 or -1,
			strafeAt = 0, lastPos = m.PrimaryPart.Position, lastMoveCheck = os.clock(), seenAt = 0, burstLeft = 0 }
	end

	local function botThink(rec, now)
		local ai, hum, root = rec.ai, rec.hum, recRoot(rec)
		if not ai or not hum or not root or hum.Health <= 0 then return end
		local best, bestD
		for _, o in ipairs(Match.list) do
			if o.alive and isEnemy(rec, o) then
				local r2 = recRoot(o)
				if r2 then
					local d = (r2.Position - root.Position).Magnitude
					if not bestD or d < bestD then best, bestD = o, d end
				end
			end
		end
		for _, o in ipairs(Match.bombers) do
			if o.alive and isEnemy(rec, o) then
				local r2 = recRoot(o)
				if r2 then
					local d = (r2.Position - root.Position).Magnitude
					if not bestD or d < bestD then best, bestD = o, d end
				end
			end
		end
		if not best then return end
		local tr = recRoot(best)
		local tp = tr.Position
		local def = WEAPONS[rec.ids[1]]
		-- each bot picks its own flank point and keeps away from teammates, so they never clump up
		if not ai.flank or now > (ai.flankAt or 0) then
			local ang = math.random() * math.pi * 2
			ai.flank = Vector3.new(math.cos(ang), 0, math.sin(ang)) * (18 + math.random() * 34)
			ai.flankAt = now + 7 + math.random() * 8
		end
		local sep = Vector3.zero
		for _, o in ipairs(Match.list) do
			if o ~= rec and o.alive and not isEnemy(rec, o) then
				local r3 = recRoot(o)
				if r3 then
					local dv = root.Position - r3.Position
					dv = Vector3.new(dv.X, 0, dv.Z)
					local dm = dv.Magnitude
					if dm < 16 and dm > 0.1 then sep += dv.Unit * ((16 - dm) / 16) end
				end
			end
		end
		local lim = (Match.mapInfo and Match.mapInfo.half or 100) - 8
		local limX = ((Match.mapInfo and Match.mapInfo.halfX) or (lim + 8)) - 4
		local goal = tp
		if bestD > 38 then
			local g = tp + ai.flank
			goal = Vector3.new(math.clamp(g.X, -limX, limX), tp.Y, math.clamp(g.Z, -lim, lim))
		end
		goal += sep * 10
		local params = RaycastParams.new()
		params.FilterType = Enum.RaycastFilterType.Exclude
		params.FilterDescendantsInstances = { rec.model, best.model }
		local eye = root.Position + Vector3.new(0, 1.7, 0)
		local aim = tp + Vector3.new(0, 1.2, 0)
		local los = bestD < 140 and Workspace:Raycast(eye, aim - eye, params) == nil
		if los then
			if ai.seenAt == 0 then ai.seenAt = now end
		else
			ai.seenAt = 0
		end
		if los then
			ai.align.CFrame = CFrame.lookAt(root.Position, Vector3.new(tp.X, root.Position.Y, tp.Z)).Rotation
			if now > ai.strafeAt then
				ai.strafe = -ai.strafe
				ai.strafeAt = now + 0.8 + math.random() * 1.2
			end
			local flatV = Vector3.new(tp.X - root.Position.X, 0, tp.Z - root.Position.Z)
			local flat = flatV.Magnitude > 0.1 and flatV.Unit or Vector3.new(0, 0, -1)
			local side = Vector3.new(-flat.Z, 0, flat.X) * ai.strafe
			local dest = root.Position + side * 6
			if bestD > def.rFar * 0.55 then
				dest = root.Position + flat * 10 + side * 3
			elseif bestD < 12 then
				dest = root.Position - flat * 8 + side * 4
			end
			hum:MoveTo(dest + sep * 7)
			if now - ai.seenAt > 0.38 and now >= ai.nextShot and bestD <= def.rFar * 0.9 then
				if ai.burstLeft <= 0 then ai.burstLeft = math.random(3, 7) end
				ai.burstLeft -= 1
				rec.shots += 1
				doFire(rec, def, eye, (aim - eye).Unit, false, 1.15)
				if ai.burstLeft <= 0 then
					ai.nextShot = now + 0.6 + math.random() * 0.6
				else
					ai.nextShot = now + math.max(0.1, 60 / def.rpm)
				end
			end
		else
			if not ai.path or now > ai.pathAt then
				ai.pathAt = now + 1.5 + math.random() * 0.5
				local path = PathfindingService:CreatePath({ AgentRadius = 2, AgentHeight = 5, AgentCanJump = true, WaypointSpacing = 8 })
				local ok = pcall(function() path:ComputeAsync(root.Position, goal) end)
				if ok and path.Status == Enum.PathStatus.Success then
					ai.path = path:GetWaypoints()
					ai.pathIdx = 2
				else
					ai.path = nil
				end
			end
			local wp = ai.path and ai.path[ai.pathIdx]
			local mv = hum.MoveDirection
			if mv.Magnitude > 0.1 then ai.align.CFrame = CFrame.lookAt(root.Position, root.Position + Vector3.new(mv.X, 0, mv.Z)).Rotation end
			if wp then
				hum:MoveTo(wp.Position)
				if wp.Action == Enum.PathWaypointAction.Jump then hum.Jump = true end
				local flatD = Vector3.new(wp.Position.X - root.Position.X, 0, wp.Position.Z - root.Position.Z).Magnitude
				if flatD < 4 then ai.pathIdx += 1 end
			else
				hum:MoveTo(goal)
			end
		end
		if now - ai.lastMoveCheck > 1.5 then
			if (root.Position - ai.lastPos).Magnitude < 1.5 and not los then
				hum.Jump = true
				ai.path = nil
			end
			ai.lastPos = root.Position
			ai.lastMoveCheck = now
		end
	end

	task.spawn(function()
		while true do
			task.wait(0.12)
			if Match.phase == "MATCH" and not Match.over then
				local now = os.clock()
				for _, rec in ipairs(Match.list) do
					if rec.isBot and rec.alive and rec.model and rec.model.Parent then
						local ai = rec.ai
						if ai and not ai.busy then
							ai.busy = true
							task.spawn(function()
								local ok, err = pcall(botThink, rec, now)
								if not ok then warn("BLACKOUT bot error: " .. tostring(err)) end
								ai.busy = false
							end)
						end
					end
				end
			end
		end
	end)


	-- =================================================================
	-- MATCH FLOW: LOBBY -> VOTE -> COUNTDOWN -> MATCH -> ENDING
	-- =================================================================
	local clientReady = {}

	local function setPhase(phase, status, duration)
		Match.phase = phase
		setState("Phase", phase)
		setState("Status", status)
		setState("EndsAt", duration and (Workspace:GetServerTimeNow() + duration) or 0)
	end
	local function queuedPlayers()
		local t = {}
		for _, pl in ipairs(Players:GetPlayers()) do
			if pl:GetAttribute("Queued") then table.insert(t, pl) end
		end
		return t
	end
	local function humanCount()
		local n = 0
		for _, r in ipairs(Match.list) do
			if not r.isBot and not r.left then n += 1 end
		end
		return n
	end
	local function removeRec(rec)
		rec.left = true
		rec.alive = false
		if rec.model then Match.byModel[rec.model] = nil end
		Match.records[rec.inst] = nil
		for i, r in ipairs(Match.list) do
			if r == rec then
				table.remove(Match.list, i)
				break
			end
		end
	end
	local function commitStats(rec, won)
		if rec.committed or rec.isBot then return end
		rec.committed = true
		local p = Profiles[rec.inst]
		if not p then return end
		p.kills += rec.kills
		p.deaths += rec.deaths
		p.assists += rec.assists
		p.headshots += rec.headshots
		p.shots += rec.shots
		p.hits += rec.hits
		if won ~= nil then
			p.matches += 1
			if won then p.wins += 1 else p.losses += 1 end
		end
	end
	local function leaveMatch(pl)
		local rec = Match.records[pl]
		if not rec then return end
		commitStats(rec, nil)
		removeRec(rec)
		pl:SetAttribute("InMatch", false)
		pl:SetAttribute("Team", "")
		local char = pl.Character
		if char then
			local g = char:FindFirstChild("BO_Gun")
			if g then g:Destroy() end
			if hubSpawnCF and char:FindFirstChild("HumanoidRootPart") then char:PivotTo(hubSpawnCF) end
		end
	end

	local function hookPlayerChar(rec, char)
		local hum = char:WaitForChild("Humanoid", 10)
		local root = char:WaitForChild("HumanoidRootPart", 10)
		if not hum or not root or rec.left then return end
		if rec.model then Match.byModel[rec.model] = nil end
		rec.model = char
		rec.hum = hum
		Match.byModel[char] = rec
		hum.MaxHealth = 100
		hum.Health = 100
		hum.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None
		hum.HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff
		hum.UseJumpPower = true
		hum.JumpPower = 50
		rec.alive = true
		rec.hitLog = {}
		rec.lastDamaged = 0
		rec.lastHitBy = nil
		rec.spawnProtectUntil = os.clock() + CFG.SPAWN_PROTECT
		hum.Died:Connect(function()
			if rec.hum == hum then onDied(rec) end -- ignore a stale body dying after a newer one was hooked
		end)
		local okSp, errSp = pcall(function()
			placeAt(char, pickSpawn(rec))
			giveLoadout(rec)
			attachWorldGun(rec)
		end)
		if not okSp then warn("BLACKOUT spawn setup error for " .. rec.name .. ": " .. tostring(errSp)) end
		rec.inst:SetAttribute("Team", rec.team)
		rec.inst:SetAttribute("InMatch", true)
		Net.Spawned:FireClient(rec.inst, rec.team)
		sendAmmo(rec, "spawn")
	end

	local function spawnAll()
		for _, rec in ipairs(Match.list) do
			if rec.isBot then
				spawnBot(rec)
			else
				local pl = rec.inst
				local char = pl.Character
				local hum = char and char:FindFirstChildOfClass("Humanoid")
				if hum and hum.Health > 0 then
					task.spawn(hookPlayerChar, rec, char)
				else
					pl:LoadCharacter()
				end
			end
		end
	end

	-- respawn safety net: if a player is still dead well after the respawn timer (the automatic respawn failed, or the new
	-- body was never set up), hook their live body or force a fresh character so they are never stuck
	task.spawn(function()
		while true do
			task.wait(1)
			if Match.phase == "MATCH" or Match.phase == "COUNTDOWN" then
				for _, rec in ipairs(Match.list) do
					if not rec.isBot and not rec.left and not rec.alive and rec.inst.Parent and rec.diedAt
						and os.clock() - rec.diedAt > CFG.RESPAWN_TIME + 1.5 then
						rec.diedAt = os.clock()
						local pl = rec.inst
						local char = pl.Character
						local hum = char and char:FindFirstChildOfClass("Humanoid")
						if hum and hum.Health > 0 and rec.model ~= char then
							task.spawn(hookPlayerChar, rec, char)
						else
							pcall(function() pl:LoadCharacter() end)
						end
					end
				end
			end
		end
	end)

	local function buildRows()
		local rows = {}
		for _, r in ipairs(Match.list) do
			local pf = (not r.isBot) and Profiles[r.inst] or nil
			table.insert(rows, { n = r.name, t = r.team, k = r.kills, d = r.deaths, a = r.assists, s = scoreOf(r), g = r.gg, b = r.isBot,
				o = r.opId or (pf and pf.equip and pf.equip.operator) or "OP_STANDARD", c = (r.shots or 0) > 0 and math.floor((r.hits or 0) / r.shots * 100 + 0.5) or 0,
				x = r.alive and true or false, u = (not r.isBot and r.inst) and r.inst.UserId or nil })
		end
		table.sort(rows, function(a, b)
			if Match.mode and Match.mode.id == "GUN" and a.g ~= b.g then return a.g > b.g end
			return a.s > b.s
		end)
		return rows
	end
	local function broadcastScores()
		local rows = buildRows()
		for _, rec in ipairs(Match.list) do
			if not rec.isBot and rec.inst.Parent then
				Net.Scores:FireClient(rec.inst, rows, Match.score.Alpha, Match.score.Bravo)
			end
		end
	end

	-- ---------- voting (one vote per queued player, ties broken randomly) ----------
	local function voteTally(v)
		local t = { map = {}, mode = {}, size = {} }
		for _, id in ipairs(v.maps) do t.map[id] = 0 end
		for _, id in ipairs(v.modes) do t.mode[id] = 0 end
		for _, id in ipairs(v.sizes or {}) do t.size[id] = 0 end
		for pl, pick in pairs(v.picks) do
			if pl.Parent then
				if pick.map and t.map[pick.map] then t.map[pick.map] += 1 end
				if pick.mode and t.mode[pick.mode] then t.mode[pick.mode] += 1 end
				if pick.size and t.size[pick.size] then t.size[pick.size] += 1 end
			end
		end
		return t
	end
	local function pushVote()
		local v = Match.vote
		if not v then return end
		local tally = voteTally(v)
		for _, pl in ipairs(queuedPlayers()) do
			local mine = v.picks[pl] or {}
			Net.VoteState:FireClient(pl, { maps = v.maps, modes = v.modes, tally = tally, mine = { map = mine.map, mode = mine.mode, size = mine.size, start = mine.start }, sizes = v.sizes,
				open = v.open, endsAt = v.endsAt, result = v.result })
		end
	end
	local function pickWinner(v, kind, options)
		local counts = {}
		for _, o in ipairs(options) do counts[o] = 0 end
		for pl, pick in pairs(v.picks) do
			if pl.Parent and pick[kind] and counts[pick[kind]] ~= nil then counts[pick[kind]] += 1 end
		end
		local best, tied = -1, {}
		for _, o in ipairs(options) do
			if counts[o] > best then
				best = counts[o]
				tied = { o }
			elseif counts[o] == best then
				table.insert(tied, o)
			end
		end
		return tied[math.random(1, #tied)]
	end
	local function runVote()
		local v = { open = true, maps = shuffle({ table.unpack(MAP_ORDER) }), modes = MODE_ORDER, sizes = { "3", "4", "5" }, picks = {},
			endsAt = Workspace:GetServerTimeNow() + CFG.VOTE_TIME }
		Match.vote = v
		setPhase("VOTE", "VOTE FOR MAP AND MODE", CFG.VOTE_TIME)
		pushVote()
		local endClock = os.clock() + CFG.VOTE_TIME
		local lastPush = 0
		while os.clock() < endClock do
			task.wait(0.25)
			local q = queuedPlayers()
			if #q < CFG.MIN_PLAYERS then
				v.open = false
				Match.vote = nil
				return nil
			end
			local all = #q > 0
			for _, pl in ipairs(q) do
				local p = v.picks[pl]
				if not (p and p.start) then
					all = false
					break
				end
			end
			if #q >= 10 then
				all = true
				for _, pl in ipairs(q) do
					local p = v.picks[pl]
					if not (p and p.map and p.mode) then all = false break end
				end
			end
			if all and endClock - os.clock() > 1.2 then
				endClock = os.clock() + 1
				v.endsAt = Workspace:GetServerTimeNow() + 1
			end
			if os.clock() - lastPush > 1 then
				lastPush = os.clock()
				pushVote()
			end
		end
		v.open = false
		local mapId = pickWinner(v, "map", v.maps)
		local modeId = pickWinner(v, "mode", v.modes)
		Match.size = tonumber(pickWinner(v, "size", v.sizes)) or 4
		v.result = { map = mapId, mode = modeId, size = Match.size }
		pushVote()
		task.wait(1.2)
		Match.vote = nil
		return mapId, modeId
	end

	Net.Vote.OnServerEvent:Connect(function(player, kind, id)
		local v = Match.vote
		if not v or not v.open or not player:GetAttribute("Queued") then return end
		if type(kind) ~= "string" or type(id) ~= "string" then return end
		if not rateOk(player, "vote", 0.2) then return end
		if kind == "start" then
			local pk = v.picks[player] or {}
			pk.start = true
			v.picks[player] = pk
			pushVote()
			return
		end
		if kind == "map" then
			if not table.find(v.maps, id) then return end
		elseif kind == "mode" then
			if not table.find(v.modes, id) then return end
		elseif kind == "size" then
			if not table.find(v.sizes, id) then return end
		else
			return
		end
		local pick = v.picks[player] or {}
		pick[kind] = id
		v.picks[player] = pick
		pushVote()
	end)

	-- ---------- match setup ----------
	local function addBot(team, names)
		Match.botCount += 1
		local nm = names[((Match.botCount - 1) % #names) + 1]
		return newRecord("BOT" .. Match.botCount, nm, team, true)
	end
	local function setupMatch(mapId, modeId)
		Match.mode = MODES[modeId]
		Match.mapId = mapId
		Match.list = {}
		Match.records = {}
		clearBombers()
		Match.byModel = {}
		Match.score = { Alpha = 0, Bravo = 0 }
		Match.over = false
		Match.winner = nil
		Match.botCount = 0
		Match.mapInfo = buildMap(mapId)
		setState("ModeId", modeId)
		setState("MapId", mapId)
		setState("ScoreAlpha", 0)
		setState("ScoreBravo", 0)
		setState("Limit", Match.mode.limit)
		local plist = shuffle(queuedPlayers())
		local nA, nB = 0, 0
		for _, pl in ipairs(plist) do
			local team = "FFA"
			if Match.mode.teams then
				if nA <= nB then
					team = "Alpha"
					nA += 1
				else
					team = "Bravo"
					nB += 1
				end
			end
			local rec = newRecord(pl, pl.Name, team, false)
			rec.startXp = Profiles[pl] and Profiles[pl].xp or 0
			pl:SetAttribute("InMatch", true)
			pl:SetAttribute("Team", team)
		end
		local names = shuffle({ table.unpack(BOT_NAMES) })
		if modeId == "FREE" then
			-- free roam: no AI at all
		elseif #plist >= (Match.size or 4) * 2 then
			-- enough real players: no AI at all
		elseif Match.mode.teams then
			local fill = (Match.size or 4) * 2
			local per = math.max(math.ceil(fill / 2), math.max(nA, nB))
			for _ = nA + 1, per do addBot("Alpha", names) end
			for _ = nB + 1, per do addBot("Bravo", names) end
		else
			for _ = #plist + 1, (Match.size or 4) * 2 do addBot("FFA", names) end
		end
	end

	-- AI is only filler: if real players drop below the threshold mid-match, top the lobby back up with bots
	task.spawn(function()
		while true do
			task.wait(6)
			if Match.phase == "MATCH" and not Match.over and Match.mode and Match.mode.id ~= "FREE" and humanCount() < (Match.size or 4) * 2 then
				local nA, nB, total, bots = 0, 0, 0, 0
				for _, r in ipairs(Match.list) do
					if not r.left then
						total += 1
						if r.isBot then bots += 1 end
						if r.team == "Alpha" then nA += 1 elseif r.team == "Bravo" then nB += 1 end
					end
				end
				if total < (Match.size or 4) * 2 then
					local team = "FFA"
					if Match.mode.teams then team = (nA <= nB) and "Alpha" or "Bravo" end
					local rec = addBot(team, shuffle({ table.unpack(BOT_NAMES) }))
					spawnBot(rec)
				end
			end
		end
	end)

	local function joinMatch(pl)
		if Match.records[pl] or Match.phase ~= "MATCH" or not Match.mode then return end
		local team = "FFA"
		if Match.mode.teams then
			local nA, nB = 0, 0
			for _, r in ipairs(Match.list) do
				if r.team == "Alpha" then nA += 1 elseif r.team == "Bravo" then nB += 1 end
			end
			team = (nA <= nB) and "Alpha" or "Bravo"
		end
		for _, r in ipairs(Match.list) do
			if r.isBot and r.team == team then
				if r.model then r.model:Destroy() end
				removeRec(r)
				break
			end
		end
		local rec = newRecord(pl, pl.Name, team, false)
		rec.startXp = Profiles[pl] and Profiles[pl].xp or 0
		pl:SetAttribute("InMatch", true)
		pl:SetAttribute("Team", team)
		local char = pl.Character
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if hum and hum.Health > 0 then
			task.spawn(hookPlayerChar, rec, char)
		else
			pl:LoadCharacter()
		end
	end

	local function decideByTime()
		local mode = Match.mode
		if mode.teams then
			if Match.score.Alpha > Match.score.Bravo then
				Match.winner = { team = "Alpha" }
			elseif Match.score.Bravo > Match.score.Alpha then
				Match.winner = { team = "Bravo" }
			else
				Match.winner = { draw = true }
			end
			return
		end
		local top, topKey, tied
		for _, r in ipairs(Match.list) do
			local key = (mode.id == "GUN") and (r.gg * 1000 + r.kills) or r.kills
			if not top or key > topKey then
				top, topKey, tied = r, key, false
			elseif key == topKey then
				tied = true
			end
		end
		if top and not tied and top.kills > 0 then Match.winner = { rec = top } else Match.winner = { draw = true } end
	end

	local function finishMatch()
		setPhase("ENDING", "MATCH COMPLETE", nil)
		local mode = Match.mode
		local w = Match.winner or { draw = true }
		local winnerLabel = "DRAW"
		if w.team then winnerLabel = string.upper(w.team) .. " WINS" elseif w.rec then winnerLabel = w.rec.name .. " WINS" end
		local rows = buildRows()
		for _, rec in ipairs(Match.list) do
			if not rec.isBot and rec.inst.Parent then
				local won = ((w.team ~= nil and w.team == rec.team) or (w.rec == rec)) and true or false
				if rec.shots > 0 or rec.kills > 0 then giveXp(rec, CFG.XP_PARTICIPATE, "MATCH COMPLETE") end
				if won then giveXp(rec, CFG.XP_WIN, "VICTORY") end
				commitStats(rec, won)
				local p = Profiles[rec.inst]
				local lvlBefore = levelFromXp(rec.startXp or 0)
				local lvl, into, need = levelFromXp(p and p.xp or 0)
				Net.Results:FireClient(rec.inst, {
					mode = mode.name, map = MAPS[Match.mapId].name, winner = winnerLabel, won = won, draw = w.draw == true,
					rows = rows, you = rec.name, team = rec.team, teams = mode.teams, kills = rec.kills, deaths = rec.deaths,
					assists = rec.assists, headshots = rec.headshots, xp = rec.xpGain, lvlBefore = lvlBefore, lvl = lvl,
					into = into, need = need, scoreA = Match.score.Alpha, scoreB = Match.score.Bravo,
				})
				sendProfile(rec.inst)
				task.spawn(saveProfile, rec.inst, false)
			end
		end
		task.wait(CFG.END_SCREEN_TIME)
	end

	local function cleanupMatch()
		for _, rec in ipairs(Match.list) do
			if rec.isBot then
				if rec.model then rec.model:Destroy() end
			else
				local pl = rec.inst
				if pl.Parent then
					pl:SetAttribute("InMatch", false)
					pl:SetAttribute("Team", "")
					local char = pl.Character
					if char then
						local g = char:FindFirstChild("BO_Gun")
						if g then g:Destroy() end
						if hubSpawnCF and char:FindFirstChild("HumanoidRootPart") then char:PivotTo(hubSpawnCF) end
					end
				end
			end
		end
		BotFolder:ClearAllChildren()
		Match.list = {}
		Match.records = {}
		clearBombers()
		Match.byModel = {}
		Match.over = false
		Match.winner = nil
		MapFolder:ClearAllChildren()
		setState("ModeId", "")
		setState("MapId", "")
		setState("ScoreAlpha", 0)
		setState("ScoreBravo", 0)
	end

	task.spawn(function()
		while true do
			setPhase("LOBBY", "WAITING FOR PLAYERS", nil)
			repeat task.wait(0.4) until #queuedPlayers() >= CFG.MIN_PLAYERS
			local mapId, modeId = runVote()
			if mapId then
				setPhase("COUNTDOWN", "GET READY", nil)
				setupMatch(mapId, modeId)
				task.wait(3) -- let the finished map replicate to every client before anyone drops in (prevents falling/pop-in)
				spawnAll()
				setPhase("COUNTDOWN", "MATCH STARTS SOON", CFG.COUNTDOWN)
				local t0 = os.clock() + CFG.COUNTDOWN
				while os.clock() < t0 and humanCount() > 0 do task.wait(0.2) end
				if humanCount() > 0 then
					local mode = Match.mode
					setPhase("MATCH", "FIGHT", mode.time)
					local endClock = os.clock() + mode.time
					local lastScores = 0
					while os.clock() < endClock and not Match.over and humanCount() > 0 do
						task.wait(0.25)
						if os.clock() - lastScores > 1 then
							lastScores = os.clock()
							broadcastScores()
						end
					end
					if humanCount() > 0 then
						if not Match.over then decideByTime() end
						Match.over = true
						finishMatch()
					end
				end
				cleanupMatch()
			end
		end
	end)

	-- =================================================================
	-- PLAYER CONNECTIONS + CLIENT REQUESTS
	-- =================================================================
	local function onCharAdded(pl, char)
		local rec = Match.records[pl]
		if rec and (Match.phase == "COUNTDOWN" or Match.phase == "MATCH") and not rec.left then
			hookPlayerChar(rec, char)
		else
			local root = char:WaitForChild("HumanoidRootPart", 10)
			if root and hubSpawnCF then char:PivotTo(hubSpawnCF) end
			pl:SetAttribute("InMatch", false)
		end
	end

	local function onPlayerAdded(pl)
		local ls = Instance.new("Folder")
		ls.Name = "leaderstats"
		for _, n in ipairs({ "Kills", "Deaths" }) do
			local v = Instance.new("IntValue")
			v.Name = n
			v.Parent = ls
		end
		ls.Parent = pl
		pl:SetAttribute("Queued", false)
		pl:SetAttribute("InMatch", false)
		pl:SetAttribute("Team", "")
		pl:SetAttribute("Level", 1)
		pl.CharacterAdded:Connect(function(c) onCharAdded(pl, c) end)
		if pl.Character then task.spawn(onCharAdded, pl, pl.Character) end
		task.spawn(loadProfile, pl)
		task.delay(12, function()
			if pl.Parent and not clientReady[pl] then
				warn("BLACKOUT: no CLIENT script detected for " .. pl.Name .. ". Paste this same code into a LocalScript "
					.. "(not a Script) inside StarterPlayer > StarterPlayerScripts.")
			end
		end)
	end
	Players.PlayerAdded:Connect(onPlayerAdded)
	for _, pl in ipairs(Players:GetPlayers()) do task.spawn(onPlayerAdded, pl) end

	Players.PlayerRemoving:Connect(function(pl)
		leaveMatch(pl)
		saveProfile(pl, true)
		Profiles[pl] = nil
		rlTimes[pl] = nil
		clientReady[pl] = nil
	end)

	Net.ClientReady.OnServerEvent:Connect(function(pl)
		clientReady[pl] = true
		print("BLACKOUT: client script confirmed for " .. pl.Name)
		sendProfile(pl)
	end)

	Net.Queue.OnServerEvent:Connect(function(pl, want)
		if type(want) ~= "boolean" or not rateOk(pl, "queue", 0.5) then return end
		pl:SetAttribute("Queued", want)
		if want then
			joinMatch(pl)
			Net.Banner:FireClient(pl, "QUEUED - match starts when voting begins", "info")
		else
			leaveMatch(pl)
		end
	end)

	Net.Loadout.OnServerEvent:Connect(function(pl, primary, secondary)
		if not rateOk(pl, "loadout", 0.5) then return end
		local p = Profiles[pl]
		if not p or type(primary) ~= "string" or type(secondary) ~= "string" then return end
		local a, b = WEAPONS[primary], WEAPONS[secondary]
		local lvl = levelOf(pl)
		local own = p.owned or {}
		local function locked(w) return w.unlock > lvl and not (w.premium and own["W_" .. w.id]) end
		if not a or not b or a.slot ~= "primary" or b.slot ~= "secondary" or locked(a) or locked(b) then
			sendProfile(pl)
			return
		end
		p.loadout = { primary = primary, secondary = secondary, att = p.loadout and p.loadout.att or {} }
		sendProfile(pl)
	end)

	Net.Attach.OnServerEvent:Connect(function(pl, wid, list)
		if not rateOk(pl, "attach", 0.3) then return end
		local p = Profiles[pl]
		if not p or type(wid) ~= "string" or not WEAPONS[wid] then return end
		p.loadout = p.loadout or { primary = "VX44", secondary = "MK2" }
		p.loadout.att = p.loadout.att or {}
		p.loadout.att[wid] = cleanAttach(list, levelOf(pl), p.owned or {}, wid)
		sendProfile(pl)
		-- apply immediately if the player is alive in a match (not Gun Game)
		local rec = Match.records[pl]
		if rec and rec.alive and rec.eff and Match.mode and Match.mode.id ~= "GUN" then
			for i = 1, 2 do
				if rec.ids[i] == wid then
					rec.eff[i] = effDef(WEAPONS[wid], p.loadout.att[wid])
					if rec.ammo[i].mag > rec.eff[i].mag then rec.ammo[i].mag = rec.eff[i].mag end
				end
			end
			attachWorldGun(rec)
			sendAmmo(rec, "attach")
		end
	end)


	-- =================================================================
	-- STORE: purchases (Developer Products), ownership, operators
	-- =================================================================
	local MarketplaceService = game:GetService("MarketplaceService")
	local RunServiceS = game:GetService("RunService")
	local function owns(p, id)
		if not p then return false end
		if OPERATORS[id] then
			local lv = levelFromXp(p.xp)
			return operatorOwned(id, p.owned, { level = lv, kills = p.kills, headshots = p.headshots, wins = p.wins })
		end
		local o = CAMOS[id]
		if o and o.free then return true end
		return p.owned[id] == true
	end
	local function applyOperator(pl)
		local p = Profiles[pl]
		local char = pl.Character
		if not p or not char then return end
		if char:GetAttribute("OpStyled") == p.equip.operator then return end
		for _, x in ipairs(char:GetChildren()) do
			if x:IsA("BasePart") and string.sub(x.Name, 1, 2) == "Op" then x:Destroy() end
		end
		char:SetAttribute("OpStyled", p.equip.operator)
		styleOperator(char, p.equip.operator, false)
	end
	local function grantItem(pl, id)
		local p = Profiles[pl]
		local it = STORE_BY_ID[id]
		if not p or not it then return false end
		if it.grants then
			for _, g in ipairs(it.grants) do p.owned[g] = true end
		end
		p.owned[id] = true
		sendProfile(pl)
		task.spawn(saveProfile, pl, false)
		Net.StoreMsg:FireClient(pl, "UNLOCKED: " .. it.name, true)
		return true
	end
	Net.Buy.OnServerEvent:Connect(function(pl, id)
		if not rateOk(pl, "buy", 1) or type(id) ~= "string" then return end
		local it = STORE_BY_ID[id]
		local p = Profiles[pl]
		if not it or not p then return end
		if p.owned[id] then
			Net.StoreMsg:FireClient(pl, "ALREADY OWNED", false)
			return
		end
		local pid = STORE_PRODUCTS[id] or 0
		if pid > 0 then
			pcall(function() MarketplaceService:PromptProductPurchase(pl, pid) end)
		elseif RunServiceS:IsStudio() then
			grantItem(pl, id) -- free test grant while no product ID is configured (Studio only)
		else
			Net.StoreMsg:FireClient(pl, "NOT AVAILABLE YET", false)
		end
	end)
	-- battle pass claiming (free track for everyone, premium track needs the pass)
	Net.Pass.OnServerEvent:Connect(function(pl, action, track, lvl)
		if not rateOk(pl, "pass", 0.25) then return end
		local p = Profiles[pl]
		if not p then return end
		p.passClaimed = type(p.passClaimed) == "table" and p.passClaimed or {}
		if action == "buy" then
			if p.owned.PASS_PREMIUM then
				Net.StoreMsg:FireClient(pl, "ALREADY OWNED", false)
				return
			end
			local pid = STORE_PRODUCTS.PASS_PREMIUM or 0
			if pid > 0 then
				pcall(function() MarketplaceService:PromptProductPurchase(pl, pid) end)
			elseif RunServiceS:IsStudio() then
				grantItem(pl, "PASS_PREMIUM")
			else
				Net.StoreMsg:FireClient(pl, "NOT AVAILABLE YET", false)
			end
			return
		end
		if action ~= "claim" or (track ~= "free" and track ~= "prem") or type(lvl) ~= "number" then return end
		lvl = math.floor(lvl)
		if lvl < 1 or lvl > PASS_MAX or lvl > levelFromXp(p.xp) then return end
		if track == "prem" and not p.owned.PASS_PREMIUM then
			Net.StoreMsg:FireClient(pl, "PREMIUM PASS REQUIRED", false)
			return
		end
		local key = track .. lvl
		if p.passClaimed[key] then return end
		local r = PASS[track][lvl]
		if not r then return end
		p.passClaimed[key] = true
		if r.kind == "item" then
			for _, id in ipairs(r.ids) do p.owned[id] = true end
		elseif r.kind == "xp" then
			p.xp += r.n
		end
		sendProfile(pl)
		task.spawn(saveProfile, pl, false)
		Net.StoreMsg:FireClient(pl, "CLAIMED: " .. r.label, true)
	end)
	-- daily login reward: once per UTC day, 7-day streak, bonus XP
	local DAILY_XP_S = { 100, 150, 200, 250, 300, 400, 750 }
	Net.Daily.OnServerEvent:Connect(function(pl)
		if not rateOk(pl, "daily", 1) then return end
		local p = Profiles[pl]
		if not p then return end
		p.daily = type(p.daily) == "table" and p.daily or { last = 0, streak = 0 }
		local today = os.time() // 86400
		if p.daily.last == today then return end
		local day = (today - p.daily.last == 1) and (p.daily.streak % 7 + 1) or 1
		p.daily.last, p.daily.streak = today, day
		p.xp += DAILY_XP_S[day]
		sendProfile(pl)
		task.spawn(saveProfile, pl, false)
	end)
	Net.EquipItem.OnServerEvent:Connect(function(pl, kind, id)
		if not rateOk(pl, "equipitem", 0.4) or type(kind) ~= "string" or type(id) ~= "string" then return end
		local p = Profiles[pl]
		if not p then return end
		if kind == "operator" and OPERATORS[id] and owns(p, id) then
			p.equip.operator = id
			sendProfile(pl)
			applyOperator(pl)
		elseif kind == "camo" and CAMOS[id] and owns(p, id) then
			p.equip.camo = id
			sendProfile(pl)
		elseif kind == "streak" and (id == "AIRSTRIKE" or id == "BOMBER") then
			p.equip.streak = id
			sendProfile(pl)
		end
	end)
	do
		local reverse = {}
		for id, pid in pairs(STORE_PRODUCTS) do if pid > 0 then reverse[pid] = id end end
		MarketplaceService.ProcessReceipt = function(info)
			local pl = Players:GetPlayerByUserId(info.PlayerId)
			if not pl then return Enum.ProductPurchaseDecision.NotProcessedYet end
			local id = reverse[info.ProductId]
			if not id then return Enum.ProductPurchaseDecision.NotProcessedYet end
			if not Profiles[pl] then return Enum.ProductPurchaseDecision.NotProcessedYet end
			grantItem(pl, id)
			return Enum.ProductPurchaseDecision.PurchaseGranted
		end
	end
	task.spawn(function()
		while true do
			task.wait(0.4)
			for _, pl in ipairs(Players:GetPlayers()) do pcall(applyOperator, pl) end
		end
	end)

	Net.Settings.OnServerEvent:Connect(function(pl, t)
		if not rateOk(pl, "settings", 1) then return end
		local clean = sanitizeSettings(t)
		local p = Profiles[pl]
		if clean and p then p.settings = clean end
	end)

	task.spawn(function()
		while true do
			task.wait(CFG.AUTOSAVE)
			for _, pl in ipairs(Players:GetPlayers()) do task.spawn(saveProfile, pl, false) end
		end
	end)
	game:BindToClose(function()
		local pending = 0
		for _, pl in ipairs(Players:GetPlayers()) do
			pending += 1
			task.spawn(function()
				saveProfile(pl, true)
				pending -= 1
			end)
		end
		local t0 = os.clock()
		while pending > 0 and os.clock() - t0 < 20 do task.wait(0.2) end
	end)

	do
		local okT, tech = pcall(function() return game:GetService("Lighting").Technology end)
		if okT and tech == Enum.Technology.Voxel then
			warn("BLACKOUT: Lighting.Technology is Voxel. That mode makes surfaces flicker as you move. In Studio, select Lighting in the Explorer and set Technology to ShadowMap (or Future).")
		end
		local missing = 0
		for _, it in ipairs(STORE_ITEMS) do if (STORE_PRODUCTS[it.id] or 0) == 0 then missing += 1 end end
		if missing > 0 then
			warn("BLACKOUT store: " .. missing .. " items have no Developer Product ID yet (STORE_PRODUCTS). In Studio they are free test grants; in a live game they show NOT AVAILABLE YET.")
		end
	end
	print("BLACKOUT: server online. Hub + 3 maps + 4 modes ready. Waiting for the client script.")
end

-- =====================================================================
-- CLIENT
-- =====================================================================
local function runClient()
	local DL, RT, PC, MS, TS, XT = {}, {}, {}, {}, {}, {}
	-- OPTIONAL ARTWORK: upload your logo / cover art in Creator Hub (Assets > Decals/Images), then paste "rbxassetid://YOURID" here.
	local MENU_LOGO_ID = ""
	local MENU_COVER_ID = ""
	local Players = game:GetService("Players")
	local Workspace = game:GetService("Workspace")
	local RS = game:GetService("ReplicatedStorage")
	local TweenService = game:GetService("TweenService")
	local UIS = game:GetService("UserInputService")
	local CAS = game:GetService("ContextActionService")
	local StarterGui = game:GetService("StarterGui")
	local Lighting = game:GetService("Lighting")
	local SoundService = game:GetService("SoundService")
	local Debris = game:GetService("Debris")

	local player = Players.LocalPlayer
	local playerGui = player:WaitForChild("PlayerGui")
	if player:GetAttribute("BOClientLoaded") then return end
	player:SetAttribute("BOClientLoaded", true)

	local camera = Workspace.CurrentCamera
	local NetFolder = RS:WaitForChild("BO_Net")
	local Net = {}
	for _, n in ipairs({ "Queue", "Daily", "Fire", "Reload", "Swap", "Melee", "Vote", "VoteState", "Loadout", "Settings", "Profile",
		"Feed", "ShotFx", "Hit", "Ammo", "Dmg", "Died", "Spawned", "Xp", "LevelUp", "Results", "Scores", "Banner", "ClientReady", "Throw", "Stim", "Attach", "Buy", "EquipItem", "StoreMsg", "Airstrike", "Streak", "Pass" }) do
		Net[n] = NetFolder:WaitForChild(n)
	end
	local State = RS:WaitForChild("BO_State")

	task.spawn(function()
		for _ = 1, 10 do
			local ok = pcall(function()
				StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Backpack, false)
				StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.PlayerList, false)
				StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Health, false)
			end)
			if ok then break end
			task.wait(0.5)
		end
	end)

	-- ---------- theme + UI helpers ----------
	local COL = {
		bg = RGB(10, 12, 13), panel = RGB(24, 29, 32), panel2 = RGB(38, 45, 50), line = RGB(86, 96, 102), accent = RGB(255, 108, 10), accent2 = RGB(255, 176, 40),
		orange = RGB(255, 140, 30), cyan = RGB(40, 200, 230), red = RGB(235, 60, 50), green = RGB(110, 220, 120),
		text = RGB(238, 240, 236), dim = RGB(150, 160, 162), gold = RGB(255, 176, 40),
	}
	local function purp(c)
		local r, g, b = c.R * 255, c.G * 255, c.B * 255
		local mx, mn = math.max(r, g, b), math.min(r, g, b)
		if mx < 125 and mx - mn <= 24 then
			local v = (r + g + b) / 3
			return Color3.fromRGB(math.min(255, v * 1.1 + 6), v * 0.72, math.min(255, v * 1.55 + 14))
		end
		return c
	end
	local function new(class, props, parent)
		local i = Instance.new(class)
		for k, v in pairs(props) do
			if XT.purple then
				if typeof(v) == "Color3" and (k == "BackgroundColor3" or k == "Color") then v = purp(v)
				elseif typeof(v) == "ColorSequence" and k == "Color" then
					local kp = {}
					for _, q in ipairs(v.Keypoints) do table.insert(kp, ColorSequenceKeypoint.new(q.Time, purp(q.Value))) end
					v = ColorSequence.new(kp)
				end
			end
			i[k] = v
		end
		i.Parent = parent
		return i
	end
	local function corner(inst, r) return new("UICorner", { CornerRadius = UDim.new(0, (r and r >= 30) and r or math.min(r or 6, 6)) }, inst) end
	-- BLACKOUT type: chunky Bangers for headings, friendly Nunito for everything else, applied to every text object
	do
		local NU = "rbxasset://fonts/families/Nunito.json"
		local function skin(t)
			if not (t:IsA("TextLabel") or t:IsA("TextButton") or t:IsA("TextBox")) then return end
			local f = t.Font
			if f == Enum.Font.GothamBlack then t.Font = Enum.Font.Oswald end
		end
		XT.skinFonts = function(root)
			for _, d in ipairs(root:GetDescendants()) do pcall(skin, d) end
			if not XT.skinConn then
				XT.skinConn = root.DescendantAdded:Connect(function(d) task.defer(function() pcall(skin, d) end) end)
			end
		end
	end
	local function stroke(inst, color, thick, trans)
		return new("UIStroke", { Color = color, Thickness = thick or 1, Transparency = trans or 0, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, inst)
	end
	local function label(parent, props)
		local p = { BackgroundTransparency = 1, Font = Enum.Font.GothamBold, TextColor3 = COL.text, TextSize = 16, Text = "" }
		for k, v in pairs(props) do p[k] = v end
		return new("TextLabel", p, parent)
	end
	local function tween(inst, t, goal, style, dir)
		local tw = TweenService:Create(inst, TweenInfo.new(t, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), goal)
		tw:Play()
		return tw
	end
	local function copy(t)
		local o = {}
		for k, v in pairs(t) do o[k] = (type(v) == "table") and copy(v) or v end
		return o
	end
	local function fmtTime(sec)
		sec = math.max(0, math.floor(sec + 0.5))
		return string.format("%d:%02d", sec // 60, sec % 60)
	end
	local function serverNow() return Workspace:GetServerTimeNow() end

	-- ---------- settings state (saved on the server, validated there) ----------
	local S = { keys = copy(DEFAULT_KEYS), pad = copy(DEFAULT_PAD), mobile = copy(DEFAULT_MOBILE), g = copy(DEFAULT_G) }
	local settingsLoaded = false
	local settingsHooks = {}
	local function mergeSettings(t)
		if type(t) ~= "table" then return end
		for _, sec in ipairs({ "keys", "pad", "g" }) do
			if type(t[sec]) == "table" then
				for k, v in pairs(t[sec]) do
					if S[sec][k] ~= nil and type(v) == type(S[sec][k]) then S[sec][k] = v end
				end
			end
		end
		if type(t.mobile) == "table" then
			for k, v in pairs(t.mobile) do
				if S.mobile[k] ~= nil and type(v) == "table" and type(v[1]) == "number" then S.mobile[k] = { v[1], v[2] } end
			end
		end
	end
	local saveThread
	local function saveSettings()
		if saveThread then task.cancel(saveThread) end
		saveThread = task.delay(1, function()
			saveThread = nil
			Net.Settings:FireServer({ keys = S.keys, pad = S.pad, mobile = S.mobile, g = S.g })
		end)
	end
	local function settingsChanged()
		for _, f in ipairs(settingsHooks) do pcall(f) end
	end

	local P = { level = 1, xp = 0, into = 0, need = xpForLevel(1), kills = 0, deaths = 0, assists = 0, wins = 0, losses = 0,
		matches = 0, headshots = 0, shots = 0, hits = 0, loadout = { primary = "VX44", secondary = "MK2" }, saved = false,
		owned = {}, equip = { operator = "OP_STANDARD", camo = "C_NONE" } }
	local profileHooks = {}
	-- adaptive quality: 3 = full, 2 = no bloom/lighter, 1 = minimum (keeps the framerate smooth on weak devices)
	local Q = { level = (UIS.TouchEnabled and not UIS.KeyboardEnabled) and 2 or 3, max = 3, frames = 0, acc = 0, low = 0, high = 0, changedAt = 0 }

	-- ---------- lighting (per player: hub vs map) ----------
	local lightObjs = {}
	local function ensureFx()
		lightObjs.bloom = lightObjs.bloom or Lighting:FindFirstChild("BO_Bloom") or new("BloomEffect", { Name = "BO_Bloom", Size = 24, Threshold = 2.2 }, Lighting)
		lightObjs.cc = lightObjs.cc or Lighting:FindFirstChild("BO_CC") or new("ColorCorrectionEffect", { Name = "BO_CC" }, Lighting)
	end
	local currentLight
	local function applyLighting(name, instant)
		if currentLight == name then return end
		local L = LIGHTING[name]
		if not L then return end
		currentLight = name
		ensureFx()
		for _, ch in ipairs(Lighting:GetChildren()) do
			if ch:IsA("Atmosphere") or ch:IsA("Sky") and false then ch:Destroy() end
			if (ch:IsA("BloomEffect") and ch ~= lightObjs.bloom) or (ch:IsA("ColorCorrectionEffect") and ch ~= lightObjs.cc) or ch:IsA("SunRaysEffect") or ch:IsA("DepthOfFieldEffect") then ch:Destroy() end
		end
		-- view-angle dependent sky reflections made floors flash bright/dark when looking down: switch them off
		pcall(function()
			Lighting.EnvironmentSpecularScale = L.spec or 0
			Lighting.EnvironmentDiffuseScale = L.diff or 0.35
			Lighting.ShadowSoftness = 0.3
			Lighting.GlobalShadows = L.shadows == true -- only maps without self-lit floors use real shadows
			Lighting.FogStart = 100000
			Lighting.FogEnd = 100001
		end)
		local info = TweenInfo.new(instant and 0.01 or 1.5)
		TweenService:Create(Lighting, info, { ClockTime = L.clock, Brightness = L.bright * 0.9, Ambient = L.ambient:Lerp(RGB(255, 255, 255), 0.22),
			OutdoorAmbient = L.outdoor:Lerp(RGB(255, 255, 255), 0.22), ExposureCompensation = L.exposure, ColorShift_Top = L.shift or RGB(0, 0, 0) }):Play()
		TweenService:Create(lightObjs.bloom, info, { Intensity = L.bloom * 0.5 }):Play()
		TweenService:Create(lightObjs.cc, info, { TintColor = L.tint, Saturation = L.sat }):Play()
	end
	local function refreshLighting(instant)
		local name = "HUB"
		if player:GetAttribute("InMatch") then
			local id = State:GetAttribute("MapId")
			if LIGHTING[id] then name = id end
		end
		pcall(function()
			local Lx = LIGHTING[name] or LIGHTING.HUB
			Lighting.EnvironmentSpecularScale = Lx.spec or 0
			Lighting.EnvironmentDiffuseScale = Lx.diff or 0.35
			Lighting.GlobalShadows = Lx.shadows == true
		end)
		applyLighting(name, instant)
	end

	-- ---------- sound ----------
	XT.sfxGroup = new("SoundGroup", { Name = "BO_SFX", Volume = 1 }, SoundService)
	local function playSound(path, speed, vol, parent, len, fxList)
		local s = Instance.new("Sound")
		s.SoundId = path
		s.PlaybackSpeed = speed or 1
		s.Volume = vol or 1
		s.SoundGroup = XT.sfxGroup
		s.RollOffMode = Enum.RollOffMode.InverseTapered
		s.RollOffMinDistance = 12
		s.RollOffMaxDistance = 220
		if fxList then
			for _, fx in ipairs(fxList) do fx.Parent = s end
		end
		s.Parent = parent or SoundService
		s:Play()
		if len then
			task.delay(len, function()
				if s.Parent then
					s:Stop()
					s:Destroy()
				end
			end)
		else
			Debris:AddItem(s, 4)
		end
		return s
	end
	local function uiClick() playSound(SND.click, 1.15, 0.35) end
	local function uiHover() playSound(SND.click2, 1.6, 0.12) end

	local function playGunShot(id, pos, vol)
		local parent
		if pos then
			parent = new("Part", { Anchored = true, CanCollide = false, CanQuery = false, Transparency = 1,
				Size = Vector3.new(0.2, 0.2, 0.2), Position = pos, Name = "SfxEmitter" }, Workspace)
			Debris:AddItem(parent, 3)
		end
		local custom = CUSTOM_SOUND_IDS[id]
		if custom and custom ~= "" then
			playSound(custom, 1, vol or 1, parent)
			return
		end
		local prof = SFX_PROFILE[id]
		if not prof then return end
		for i, layer in ipairs(prof) do
			local fx
			if i == 1 then
				fx = {}
				if prof.dist then table.insert(fx, new("DistortionSoundEffect", { Level = prof.dist })) end
				if prof.reverb then
					table.insert(fx, new("ReverbSoundEffect", { DecayTime = 2.2, Density = 0.8, Diffusion = 0.8, DryLevel = 0, WetLevel = -8 }))
				end
				if prof.eq then table.insert(fx, new("EqualizerSoundEffect", { HighGain = 6, MidGain = -4, LowGain = -8 })) end
			end
			playSound(SND[layer[1]], layer[2], layer[3] * (vol or 1), parent, layer[4], fx)
		end
	end

	-- ---------- world effects ----------
	local fxFolder = new("Folder", { Name = "BO_ClientFx" }, Workspace)
	local function tracer(from, to, color)
		local d = (to - from).Magnitude
		if d < 1 then return end
		local p = new("Part", { Anchored = true, CanCollide = false, CanQuery = false, CanTouch = false, Material = Enum.Material.Neon,
			Color = color, Size = Vector3.new(0.12, 0.12, d), CFrame = CFrame.lookAt(from, to) * CFrame.new(0, 0, -d / 2),
			Transparency = 0.15, Name = "Tracer" }, fxFolder)
		tween(p, 0.14, { Transparency = 1, Size = Vector3.new(0.02, 0.02, d) })
		Debris:AddItem(p, 0.25)
	end
	local function impact(pos, color)
		if Q.level <= 1 then return end
		local p = new("Part", { Anchored = true, CanCollide = false, CanQuery = false, CanTouch = false, Material = Enum.Material.Neon,
			Shape = Enum.PartType.Ball, Color = color, Size = Vector3.new(0.5, 0.5, 0.5), Position = pos, Name = "Impact" }, fxFolder)
		tween(p, 0.18, { Transparency = 1, Size = Vector3.new(1.4, 1.4, 1.4) })
		Debris:AddItem(p, 0.25)
	end

	Net.ShotFx.OnClientEvent:Connect(function(id, origin, ends)
		if id == "FRAG" and typeof(origin) == "Vector3" then
			local ball = new("Part", { Anchored = true, CanCollide = false, CanQuery = false, CanTouch = false, Material = Enum.Material.Neon,
				Shape = Enum.PartType.Ball, Color = RGB(255, 150, 60), Size = Vector3.new(2, 2, 2), Position = origin, Name = "Blast" }, fxFolder)
			tween(ball, 0.45, { Transparency = 1, Size = Vector3.new(22, 22, 22) })
			Debris:AddItem(ball, 0.6)
			playSound(SND.bass, 0.55, 1.0 * S.g.sfx)
			playSound(SND.collide, 0.6, 0.8 * S.g.sfx)
			return
		end
		local def = WEAPONS[id]
		if not def or typeof(origin) ~= "Vector3" or type(ends) ~= "table" then return end
		playGunShot(id, origin, 0.8 * S.g.sfx)
		for i, e in ipairs(ends) do
			if i > 5 then break end
			if typeof(e) == "Vector3" then
				local dir = e - origin
				if dir.Magnitude > 2 then
					tracer(origin + dir.Unit * 1.5, e, def.color)
					if dir.Magnitude < 400 then impact(e, def.color) end
				end
			end
		end
	end)

	-- ---------- game state helpers ----------
	local function inMatch() return player:GetAttribute("InMatch") == true end
	local function phase() return State:GetAttribute("Phase") or "LOBBY" end
	local function myHum()
		local c = player.Character
		return c and c:FindFirstChildOfClass("Humanoid")
	end
	local function myRoot()
		local c = player.Character
		return c and c:FindFirstChild("HumanoidRootPart")
	end
	local function isAlive()
		local h = myHum()
		return h ~= nil and h.Health > 0
	end
	local function teamColor(team)
		if S.g.colorblind then
			if team == "Alpha" then return RGB(255, 210, 60) elseif team == "Bravo" then return RGB(90, 130, 255) end
			return RGB(255, 255, 255)
		end
		if team == "Alpha" then return COL.orange elseif team == "Bravo" then return COL.cyan end
		return RGB(255, 90, 100)
	end
	local function prettyKey(name)
		local map = { MouseButton1 = "LMB", MouseButton2 = "RMB", MouseButton3 = "MMB", LeftShift = "L-SHIFT", RightShift = "R-SHIFT",
			LeftControl = "L-CTRL", RightControl = "R-CTRL", Space = "SPACE", One = "1", Two = "2", Three = "3", Four = "4", Five = "5",
			ButtonA = "A", ButtonB = "B", ButtonX = "X", ButtonY = "Y", ButtonL1 = "LB", ButtonR1 = "RB", ButtonL2 = "LT",
			ButtonR2 = "RT", ButtonL3 = "L3", ButtonR3 = "R3", ButtonSelect = "VIEW", ButtonStart = "MENU", Return = "ENTER",
			LeftAlt = "L-ALT", Tab = "TAB", Backspace = "BKSP" }
		return map[name] or string.upper(name)
	end

	-- weapon state mirrored from the server (server is the authority)
	local W = { ids = { "VX44", "MK2" }, slot = 1, ammo = { { 30, 0 }, { 12, 0 } }, reloadUntil = 0, swapUntil = 0, nextFire = 0,
		burstQ = 0, ads = false, adsT = 0, sprinting = false, sliding = false, slideUntil = 0, slideCd = 0, crouch = false }
	local function curDef()
		local base = WEAPONS[W.ids[W.slot]]
		if not base then return nil end
		return effDef(base, W.att and W.att[W.slot])
	end

	-- =================================================================
	-- (client continues: menu, HUD, vote, results, controls)
	-- =================================================================
	-- =================================================================
	-- GUI ROOT + MENU
	-- =================================================================
	local gui = new("ScreenGui", { Name = "BO_Gui", ResetOnSpawn = false, IgnoreGuiInset = true, DisplayOrder = 10,
		ZIndexBehavior = Enum.ZIndexBehavior.Sibling }, playerGui)
	local UI = { page = "DEPLOY", menuOpen = true, explore = false, pages = {}, navBtns = {} }

	local function button(parent, text, size, pos, color, cb, textSize)
		local b = new("TextButton", { Size = size, Position = pos, BackgroundColor3 = color or COL.panel2, AutoButtonColor = false,
			Text = text, Font = Enum.Font.GothamBold, TextColor3 = COL.text, TextSize = textSize or 16, BorderSizePixel = 0 }, parent)
		corner(b, 8)
		local base = b.BackgroundColor3
		b.MouseEnter:Connect(function()
			uiHover()
			tween(b, 0.1, { BackgroundColor3 = base:Lerp(Color3.new(1, 1, 1), 0.15) })
		end)
		b.MouseLeave:Connect(function() tween(b, 0.1, { BackgroundColor3 = base }) end)
		b.Activated:Connect(function()
			uiClick()
			if cb then cb(b) end
		end)
		return b
	end

	local menuHolder = new("Frame", { Name = "Menu", Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(4, 6, 10), BackgroundTransparency = 0,
		BorderSizePixel = 0, ZIndex = 2 }, gui)
	menuHolder.BackgroundColor3 = RGB(255, 255, 255)
	new("UIGradient", { Rotation = 62, Color = ColorSequence.new({ ColorSequenceKeypoint.new(0, RGB(67, 74, 74)), ColorSequenceKeypoint.new(0.4, RGB(34, 38, 38)),
		ColorSequenceKeypoint.new(1, RGB(13, 14, 14)) }) }, menuHolder)
	local menuRoot = new("Frame", { Size = UDim2.fromOffset(1250, 650), Position = UDim2.new(0.5, 0, 0.5, 34), AnchorPoint = Vector2.new(0.5, 0.5),
		BackgroundTransparency = 1, ZIndex = 6 }, menuHolder)
	MS.menuScale = new("UIScale", {}, menuRoot)
	local function rescale()
		local vp = camera.ViewportSize
		MS.menuScale.Scale = math.clamp(math.min(vp.X / 1290, (vp.Y - 76) / 650), 0.4, 2.6)
	end
	rescale()
	camera:GetPropertyChangedSignal("ViewportSize"):Connect(rescale)

	-- title-screen layout (inspired by the cover art): art backdrop, big title, bottom nav bar
	local side = new("Frame", { Size = UDim2.fromOffset(1, 1), BackgroundTransparency = 1, Visible = false }, menuRoot)
	local homeUi = new("Frame", { Name = "Home", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ZIndex = 3 }, menuHolder)
	local art = new("Frame", { Name = "Art", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ClipsDescendants = true, ZIndex = 1 }, menuHolder)
	do
		-- loud, friendly backdrop: big diagonal colour slabs behind the soldiers, a sunburst of stripes, a purple city at the bottom
		for k = 0, 11 do
			new("Frame", { Size = UDim2.new(0, 70, 1.8, 0), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.55), Rotation = -90 + k * 15,
				BackgroundColor3 = RGB(255, 255, 255), BackgroundTransparency = 0.955, BorderSizePixel = 0, ZIndex = 1 }, art)
		end
		local slabP = new("Frame", { Size = UDim2.fromScale(0.3, 1.7), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.52, 0.5), Rotation = 16,
			BackgroundColor3 = RGB(255, 106, 10), BackgroundTransparency = 0.32, BorderSizePixel = 0, ZIndex = 1 }, art)
		new("UIGradient", { Rotation = 90, Color = ColorSequence.new(RGB(255, 106, 10), RGB(190, 78, 7)) }, slabP)
		new("Frame", { Size = UDim2.fromScale(0.035, 1.7), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.69, 0.5), Rotation = 16,
			BackgroundColor3 = RGB(255, 176, 40), BackgroundTransparency = 0.05, BorderSizePixel = 0, ZIndex = 1 }, art)
		new("Frame", { Size = UDim2.fromScale(0.012, 1.7), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.725, 0.5), Rotation = 16,
			BackgroundColor3 = RGB(40, 225, 230), BackgroundTransparency = 0.1, BorderSizePixel = 0, ZIndex = 1 }, art)
		TweenService:Create(slabP, TweenInfo.new(3.2, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), { BackgroundTransparency = 0.5 }):Play()
		local rng = Random.new(11)
		-- drifting embers
		for i = 1, 22 do
			task.spawn(function()
				local ember = new("Frame", { Size = UDim2.fromOffset(2 + (i % 3) * 2, 2 + (i % 3) * 2), BackgroundColor3 = (i % 4 == 0) and RGB(255, 176, 40) or RGB(255, 106, 10),
					BorderSizePixel = 0, ZIndex = 2, Visible = false }, art)
				task.wait(math.random() * 6)
				while ember.Parent do
					if menuHolder.Visible then
						local x = math.random()
						ember.Position = UDim2.fromScale(x, 1.02)
						ember.BackgroundTransparency = 0.1
						ember.Visible = true
						local dur = 5 + math.random() * 7
						TweenService:Create(ember, TweenInfo.new(dur, Enum.EasingStyle.Sine), { Position = UDim2.fromScale(x + (math.random() - 0.5) * 0.18, 0.1 + math.random() * 0.4),
							BackgroundTransparency = 1 }):Play()
						task.wait(dur)
					else
						ember.Visible = false
						task.wait(1)
					end
				end
			end)
		end
	end
	-- content area
	XT.purple = false
	local pageBack = new("Frame", { Name = "PageBack", Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, Visible = false, ZIndex = 4 }, menuHolder)
	new("UIGradient", { Rotation = 90, Color = ColorSequence.new(RGB(30, 36, 39), RGB(10, 12, 13)) }, pageBack)
	for k = 0, 9 do
		new("Frame", { Size = UDim2.new(0, 46, 2, 0), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.1 + k * 0.1, 0.5), Rotation = 20, BackgroundColor3 = RGB(255, 255, 255),
			BackgroundTransparency = 0.975, BorderSizePixel = 0, ZIndex = 4 }, pageBack)
	end
	new("Frame", { Size = UDim2.new(1, 0, 0, 5), Position = UDim2.new(0, 0, 1, -5), BackgroundColor3 = COL.accent, BorderSizePixel = 0, ZIndex = 5 }, pageBack)
	local content = new("Frame", { Size = UDim2.fromOffset(830, 650), Position = UDim2.fromOffset(0, 0), BackgroundTransparency = 1, Visible = false,
		BorderSizePixel = 0, ClipsDescendants = false }, menuRoot)

	local function makePage(id, title, sub)
		local pg = new("Frame", { Name = id, Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false }, content)
		label(pg, { Size = UDim2.fromOffset(780, 46), Position = UDim2.fromOffset(28, 19), Text = title, TextSize = 46, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent,
			TextXAlignment = Enum.TextXAlignment.Left })
		label(pg, { Size = UDim2.fromOffset(780, 46), Position = UDim2.fromOffset(25, 16), Text = title, TextSize = 46, Font = Enum.Font.GothamBlack, ZIndex = 2,
			TextXAlignment = Enum.TextXAlignment.Left })
		label(pg, { Size = UDim2.fromOffset(780, 20), Position = UDim2.fromOffset(28, 60), Text = sub, TextSize = 13, TextColor3 = COL.dim,
			Font = Enum.Font.Gotham, TextXAlignment = Enum.TextXAlignment.Left })
		new("Frame", { Size = UDim2.fromOffset(780, 2), Position = UDim2.fromOffset(25, 79), BackgroundColor3 = COL.line, BorderSizePixel = 0 }, pg)
		new("Frame", { Size = UDim2.fromOffset(90, 5), Position = UDim2.fromOffset(25, 77), BackgroundColor3 = COL.accent2, BorderSizePixel = 0 }, pg)
		UI.pages[id] = pg
		return pg
	end
	local function showPage(id)
		UI.page = id
		for pid, pg in pairs(UI.pages) do pg.Visible = (pid == id) end
		content.Visible = id ~= nil
		pageBack.Visible = id ~= nil
		if XT.sidePanel then XT.sidePanel.Visible = id ~= nil end
		homeUi.Visible = id == nil
		if id == "PASS" and XT.buildPass then XT.buildPass() end
		if MS.nav then MS.nav.Visible = id ~= nil end
		art.Visible = true
		for nid, nb in pairs(UI.navBtns) do
			local u = nb:FindFirstChild("Under")
			local on = (nid == (id or "HOME"))
			nb.TextColor3 = on and COL.accent2 or COL.text
			nb.BackgroundTransparency = on and 0.7 or 1
			nb.BackgroundColor3 = COL.accent
			if u then u.Size = UDim2.new(on and 0.6 or 0, 0, 0, 4) end
		end
	end
	local closeBtn = button(content, "BACK HOME", UDim2.fromOffset(130, 36), UDim2.new(1, -150, 0, 16), COL.accent, function()
		if inMatch() then
			UI.menuOpen = false
			UI.inGameMenu = false
			UI.refresh()
		else
			showPage(nil)
		end
	end, 13)
	closeBtn.ZIndex = 20
	closeBtn.TextColor3 = RGB(255, 255, 255)
	closeBtn.Font = Enum.Font.GothamBlack
	closeBtn.TextSize = 18
	local exploreBtn
	-- ===================================================================
	-- BLACKOUT MAIN MENU: cover-art backdrop, 3D operator showcase, top nav, big PLAY, live featured store card
	-- ===================================================================
	local pcBar
	do
		-- optional real artwork: upload your cover art / logo to Roblox (Creator Hub > Assets) and paste the IDs at the top of the script
		if MENU_COVER_ID ~= "" then
			new("ImageLabel", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Image = MENU_COVER_ID, ScaleType = Enum.ScaleType.Crop, ZIndex = 1 }, art)
		end
		-- left-to-right readability shade
		local shade = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(8, 8, 8), BorderSizePixel = 0, ZIndex = 2 }, menuHolder)
		new("UIGradient", { Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.12), NumberSequenceKeypoint.new(0.5, 0.55), NumberSequenceKeypoint.new(1, 0.95) }) }, shade)

		-- ---------- shared 3D builders (menu showcase + store preview) ----------
		-- a clean built-in soldier (no avatar accessories) holding a rifle pose; torso is the pivot
		function XT.soldier(opId, animated)
			-- invisible skeleton (same proportions as the bot rig) dressed by the shared operator builder
			local m = Instance.new("Model")
			m.Name = "Operator"
			local function bone(n, size, cf)
				local p = Instance.new("Part")
				p.Name, p.Size, p.CFrame = n, size, cf
				p.Anchored, p.CanCollide, p.CanQuery, p.CanTouch, p.CastShadow = true, false, false, false, false
				p.Transparency = 1
				p.Parent = m
				return p
			end
			m.PrimaryPart = bone("Torso", Vector3.new(2, 2, 1), CFrame.new(0, 3, 0))
			bone("Head", Vector3.new(1.1, 1.1, 1.1), CFrame.new(0, 4.65, 0))
			local rotR = CFrame.Angles(0, math.rad(14), 0) * CFrame.Angles(math.rad(78), 0, 0)
			local rotL = CFrame.Angles(0, math.rad(-22), 0) * CFrame.Angles(math.rad(86), 0, 0)
			bone("RightArm", Vector3.new(0.8, 2, 0.8), CFrame.new(1.4, 3.85, 0) * rotR * CFrame.new(0, -1, 0))
			bone("LeftArm", Vector3.new(0.8, 2, 0.8), CFrame.new(-1.4, 3.85, 0) * rotL * CFrame.new(0, -1, 0))
			bone("RightLeg", Vector3.new(0.9, 2, 0.9), CFrame.new(0.5, 1, 0))
			bone("LeftLeg", Vector3.new(0.9, 2, 0.9), CFrame.new(-0.5, 1, 0))
			local ok, err = xpcall(dressCharacter, debug.traceback, m, opId, not animated, nil)
			if not ok then warn("BLACKOUT operator preview failed (" .. tostring(opId) .. "): " .. tostring(err)) end
			if not animated then return m end
			-- animated: remember which bone each clothing piece hangs off, drop the welds and drive everything by CFrame
			local deps = {}
			for _, d in ipairs(m:GetDescendants()) do
				if d:IsA("WeldConstraint") and d.Part0 and d.Part1 then
					local bn, q = d.Part0, d.Part1
					deps[bn] = deps[bn] or {}
					table.insert(deps[bn], { q, bn.CFrame:ToObjectSpace(q.CFrame) })
				end
			end
			for _, d in ipairs(m:GetDescendants()) do
				if d:IsA("WeldConstraint") then d:Destroy() elseif d:IsA("BasePart") then d.Anchored = true end
			end
			return m, deps
		end
		XT.gunHold = CFrame.new(0.1, 0.45, -1.9)
		function XT.gun(def)
			local g = buildGunModel(def, true)
			for _, d in ipairs(g:GetDescendants()) do
				if d:IsA("BasePart") then d.CastShadow = false end
			end
			return g
		end
		function XT.ring(accent)
			local a = new("Part", { Shape = Enum.PartType.Cylinder, Size = Vector3.new(0.25, 8.4, 8.4), Material = Enum.Material.Neon, Color = accent, Anchored = true, CanCollide = false,
				CFrame = CFrame.new(0, 0.1, 0) * CFrame.Angles(0, 0, math.pi / 2), CastShadow = false })
			local b = new("Part", { Shape = Enum.PartType.Cylinder, Size = Vector3.new(0.3, 7.2, 7.2), Material = Enum.Material.SmoothPlastic, Color = RGB(14, 16, 24), Anchored = true,
				CanCollide = false, CFrame = CFrame.new(0, 0.17, 0) * CFrame.Angles(0, 0, math.pi / 2), CastShadow = false })
			return a, b
		end

		-- ---------- operator showcase (3D): two animated soldiers, lobby-style ----------
		local SC = { mx = 0, spin = 0, drag = false, last = 0, lastDrag = 0 }
		local vp = new("ViewportFrame", { Name = "Showcase", Size = UDim2.fromOffset(600, 620), Position = UDim2.fromOffset(340, 50), BackgroundTransparency = 1, ZIndex = 5,
			Ambient = RGB(161, 177, 177), LightColor = RGB(255, 236, 214), LightDirection = Vector3.new(-0.6, -0.5, -0.9) }, menuHolder)
		local cam = new("Camera", { FieldOfView = 34 }, vp)
		vp.CurrentCamera = cam
		local actors = {}
		local nameplate, subplate
		local function weaponDef()
			local id = P.loadout and P.loadout.primary or "VX44"
			local def = WEAPONS[id] or WEAPONS.VX44
			local att = P.loadout and P.loadout.att and P.loadout.att[id]
			def = effDef(def, att and cleanAttach(att, P.level, P.owned or {}) or nil)
			return camoDef(def, P.equip and P.equip.camo)
		end
		local function clearActors()
			for _, a in ipairs(actors) do
				a.m:Destroy()
				a.g:Destroy()
				a.ringA:Destroy()
				a.ringB:Destroy()
			end
			actors = {}
		end
		local function newActor(opId, def, x, z, yaw, ph)
			local m, deps = XT.soldier(opId, true)
			local g = XT.gun(def)
			m.Parent = vp
			g.Parent = vp
			local bones = {}
			for _, n in ipairs({ "Torso", "Head", "RightArm", "LeftArm", "RightLeg", "LeftLeg" }) do bones[n] = m:FindFirstChild(n) end
			local t0 = bones.Torso.CFrame
			local loc = {}
			for n, b in pairs(bones) do loc[n] = t0:ToObjectSpace(b.CFrame) end
			local op = OPERATORS[opId] or OPERATORS.OP_STANDARD
			local ringA, ringB = Instance.new("Folder"), Instance.new("Folder")
			return { m = m, g = g, deps = deps or {}, bones = bones, loc = loc, x = x, z = z, yaw = yaw, ph = ph, aim = 0, aimUntil = 0, reloadUntil = 0,
				nextEvent = os.clock() + 1 + ph, ringA = ringA, ringB = ringB, look = 0 }
		end
		local function build()
			clearActors()
			local opId = P.equip and P.equip.operator or "OP_STANDARD"
			local op = OPERATORS[opId] or OPERATORS.OP_STANDARD
			table.insert(actors, newActor(opId, weaponDef(), -0.02, 0, -0.35, 0))
			if nameplate then
				nameplate.Text = op.name
				nameplate.TextColor3 = op.accent
				subplate.Text = op.desc
			end
		end
		table.insert(profileHooks, function() if menuHolder.Visible then build() end end)
		vp.InputBegan:Connect(function(i)
			if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
				SC.drag, SC.last = true, i.Position.X
			end
		end)
		UIS.InputChanged:Connect(function(i)
			if SC.drag and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
				SC.spin = math.clamp(SC.spin + (i.Position.X - SC.last) * 0.008, -1.2, 1.2)
				SC.last = i.Position.X
				SC.lastDrag = os.clock()
			end
		end)
		UIS.InputEnded:Connect(function(i)
			if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then SC.drag = false end
		end)
		local function setBone(a, name, cf)
			local b = a.bones[name]
			if not b then return end
			b.CFrame = cf
			local list = a.deps[b]
			if list then
				for _, d in ipairs(list) do d[1].CFrame = cf * d[2] end
			end
		end
		local function poseActor(a, now, dt, queued)
			-- schedule gestures: raise the rifle and aim, or check / reload it
			if now > a.nextEvent then
				local r = math.random()
				if a.aimUntil < now and r < (queued and 0.75 or 0.45) then
					a.aimUntil = now + 1.8 + math.random() * 1.6
				elseif r < 0.85 then
					a.reloadUntil = now + 1.5
				end
				a.nextEvent = now + (queued and 2.3 or 4.8) + math.random() * 2
			end
			a.aim += ((now < a.aimUntil and 1 or 0) - a.aim) * math.min(1, dt * 5)
			local rl = math.clamp((a.reloadUntil - now) / 1.5, 0, 1)
			local rk = rl > 0 and math.sin((1 - rl) * math.pi) or 0
			local breathe = math.sin(now * 1.7 + a.ph)
			local sway = math.sin(now * 0.9 + a.ph * 2)
			local spin = (a.x < 0 and SC.spin or SC.spin * 0.6)
			local yaw = a.yaw + sway * 0.05 + spin + (queued and math.sin(now * 0.55 + a.ph) * 0.14 or 0)
			local root = CFrame.new(a.x, 3, a.z) * CFrame.Angles(0, math.pi + yaw, 0)
			local T = root * CFrame.new(0, breathe * 0.035, 0)
			-- rifle rig: arms and gun rotate together about the shoulders so the hands stay on the gun
			local low = math.rad(-12)
			local pitch = low + (math.rad(2) - low) * a.aim - rk * math.rad(26) + breathe * math.rad(1.1)
			local ayaw = math.sin(now * 0.5 + a.ph) * math.rad(10) * (1 - a.aim) + math.sin(now * 1.3 + a.ph) * math.rad(0.8) * a.aim
			local roll = rk * math.rad(22) * (a.x < 0 and 1 or -1)
			local A = CFrame.new(0, 0.85, 0) * CFrame.Angles(pitch, ayaw, roll) * CFrame.new(0, -0.85, 0)
			local TA = T * A
			setBone(a, "Torso", T)
			setBone(a, "RightArm", TA * a.loc.RightArm)
			setBone(a, "LeftArm", TA * a.loc.LeftArm)
			a.g:PivotTo(TA * XT.gunHold)
			-- head scans around, then snaps onto the aim line while the rifle is up
			local hy = math.sin(now * 0.55 + a.ph) * 0.55 * (1 - a.aim) + ayaw * 1.2 * a.aim
			local hp = math.sin(now * 0.8 + a.ph * 3) * 0.06 + (rk > 0 and 0.22 * rk or 0)
			local neck = CFrame.new(0, 1.2, 0)
			setBone(a, "Head", T * neck * CFrame.Angles(hp, hy, 0) * neck:Inverse() * a.loc.Head)
			-- weight shift in the legs
			setBone(a, "RightLeg", root * CFrame.Angles(0, 0, sway * 0.025) * a.loc.RightLeg)
			setBone(a, "LeftLeg", root * CFrame.Angles(0, 0, sway * 0.025) * a.loc.LeftLeg)
		end
		local lastT = os.clock()
		RunService.RenderStepped:Connect(function()
			if not menuHolder.Visible or not vp.Visible then return end
			local now = os.clock()
			local dt = math.min(0.1, now - lastT)
			lastT = now
			if not SC.drag and now - SC.lastDrag > 1.5 then SC.spin += (0 - SC.spin) * math.min(1, dt * 1.2) end
			local queued = player:GetAttribute("Queued") == true
			local mx = (UIS:GetMouseLocation().X / math.max(1, camera.ViewportSize.X) - 0.5)
			SC.mx += (mx - SC.mx) * 0.07
			for _, a in ipairs(actors) do pcall(poseActor, a, now, dt, queued) end
			cam.CFrame = CFrame.lookAt(Vector3.new(SC.mx * 1.2, 4.0, 13.4), Vector3.new(0, 3.1, -1))
		end)
		homeUi:GetPropertyChangedSignal("Visible"):Connect(function() vp.Visible = homeUi.Visible end)

		-- ---------- NEW HOME: big playful layout on a 1280 x 720 canvas, scaled to fit ----------
		local PINK, YEL, INK = COL.accent, COL.accent2, RGB(18, 20, 20)
		local FUN = Enum.Font.GothamBlack -- re-skinned to the fun display font by the font pass
		local stage = new("Frame", { Name = "Stage", Size = UDim2.fromOffset(1280, 720), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
			BackgroundTransparency = 1, ZIndex = 4 }, homeUi)
		local stageScale = new("UIScale", {}, stage)
		function XT.deckFit()
			local v = camera.ViewportSize
			stageScale.Scale = math.clamp(math.min(v.X / 1280, v.Y / 720), 0.35, 1.7)
		end
		XT.deckFit()
		camera:GetPropertyChangedSignal("ViewportSize"):Connect(XT.deckFit)
		vp.Parent = stage
		vp.ZIndex = 5
		local colL = new("Frame", { Size = UDim2.fromOffset(460, 720), BackgroundTransparency = 1, ZIndex = 6 }, stage)
		local colR = new("Frame", { Size = UDim2.fromOffset(400, 720), Position = UDim2.fromOffset(880, 0), BackgroundTransparency = 1, ZIndex = 6 }, stage)
		local function queueNow()
			if player:GetAttribute("Queued") ~= true and not inMatch() then Net.Queue:FireServer(true) end
		end
		local function pop(inst) -- little bounce when you hover something
			local sc = inst:FindFirstChildOfClass("UIScale") or new("UIScale", {}, inst)
			return sc
		end

		-- logo: flat stencil-style wordmark, orange rule, small tagline
		if MENU_LOGO_ID ~= "" then
			new("ImageLabel", { Size = UDim2.fromOffset(420, 130), Position = UDim2.fromOffset(40, 64), BackgroundTransparency = 1, Image = MENU_LOGO_ID, ScaleType = Enum.ScaleType.Fit, ZIndex = 7 }, colL)
		else
			label(colL, { Size = UDim2.fromOffset(430, 100), Position = UDim2.fromOffset(40, 54), Text = "BLACKOUT", TextSize = 92, Font = FUN, TextColor3 = COL.text,
				TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 8 })
		end
		new("Frame", { Size = UDim2.fromOffset(120, 6), Position = UDim2.fromOffset(46, 160), BackgroundColor3 = COL.accent, BorderSizePixel = 0, ZIndex = 8 }, colL)
		label(colL, { Size = UDim2.fromOffset(330, 20), Position = UDim2.fromOffset(176, 153), Text = "NEAR-FUTURE WARFARE  //  SEASON 01", TextSize = 13, TextColor3 = COL.dim,
			TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 8 })

		-- the big PLAY slab: one click and you are in a match
		local playB = new("TextButton", { Text = "", AutoButtonColor = false, Size = UDim2.fromOffset(380, 96), Position = UDim2.fromOffset(40, 190), BackgroundColor3 = COL.accent,
			BorderSizePixel = 0, ZIndex = 8, ClipsDescendants = true }, colL)
		corner(playB, 4)
		local playStroke = stroke(playB, COL.text, 2, 0.75)
		label(playB, { Size = UDim2.fromOffset(240, 66), Position = UDim2.fromOffset(20, 4), Text = "PLAY", TextSize = 74, Font = FUN, TextColor3 = RGB(12, 14, 14), TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 10 })
		local playSub = label(playB, { Size = UDim2.fromOffset(300, 20), Position = UDim2.fromOffset(24, 68), Text = "FIND A MATCH", TextSize = 14, TextColor3 = RGB(60, 28, 0), Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 10 })
		label(playB, { Size = UDim2.fromOffset(70, 96), Position = UDim2.new(1, -86, 0, 0), Text = ">>", TextSize = 56, Font = FUN, TextColor3 = RGB(12, 14, 14), ZIndex = 10 })
		playB.MouseEnter:Connect(function() uiHover() tween(playB, 0.12, { BackgroundColor3 = COL.accent2 }) end)
		playB.MouseLeave:Connect(function() tween(playB, 0.12, { BackgroundColor3 = COL.accent }) end)
		playB.Activated:Connect(function() uiClick() queueNow() end)
		-- menu rows
		local function row(y, text, sub, cb, hot, tagText)
			local b = new("TextButton", { Text = "", AutoButtonColor = false, Size = UDim2.fromOffset(380, 48), Position = UDim2.fromOffset(40, y), BackgroundColor3 = COL.panel,
				BackgroundTransparency = 0.35, BorderSizePixel = 0, ZIndex = 8, ClipsDescendants = true }, colL)
			corner(b, 3)
			local bar2 = new("Frame", { Size = UDim2.fromOffset(5, 48), BackgroundColor3 = hot and COL.accent or COL.line, BorderSizePixel = 0, ZIndex = 10 }, b)
			local tl = label(b, { Size = UDim2.fromOffset(200, 48), Position = UDim2.fromOffset(20, 0), Text = text, TextSize = 28, Font = FUN, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 10 })
			label(b, { Size = UDim2.fromOffset(150, 48), Position = UDim2.new(1, -164, 0, 0), Text = sub or "", TextSize = 12, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 10 })
			if tagText then
				local tg = label(b, { Size = UDim2.fromOffset(52, 20), Position = UDim2.new(1, -64, 0.5, -10), Text = tagText, TextSize = 14, Font = FUN, TextColor3 = RGB(12, 14, 14), BackgroundColor3 = COL.accent, BackgroundTransparency = 0, ZIndex = 10 })
				corner(tg, 3)
			end
			b.MouseEnter:Connect(function()
				uiHover()
				tween(b, 0.12, { BackgroundColor3 = COL.panel2, BackgroundTransparency = 0 })
				tween(bar2, 0.12, { Size = UDim2.fromOffset(380, 48), BackgroundTransparency = 0.82, BackgroundColor3 = COL.accent })
				tl.TextColor3 = COL.accent2
			end)
			b.MouseLeave:Connect(function()
				tween(b, 0.12, { BackgroundColor3 = COL.panel, BackgroundTransparency = 0.35 })
				tween(bar2, 0.12, { Size = UDim2.fromOffset(5, 48), BackgroundTransparency = 0, BackgroundColor3 = hot and COL.accent or COL.line })
				tl.TextColor3 = COL.text
			end)
			b.Activated:Connect(function() uiClick() if cb then cb() end end)
			return b
		end
		local loBtn = row(312, "LOADOUT", "", function() showPage("LOADOUT") end)
		row(364, "OPERATORS", "skins and camos", function() showPage("OPERATORS") end)
		row(416, "STORE", "", function() showPage("STORE") end, true, "HOT")
		row(468, "BATTLE PASS", "100 levels", function() showPage("PASS") end, true, "NEW")
		row(520, "CAREER", "", function() showPage("CAREER") end)
		row(572, "SETTINGS", "", function() showPage("SETTINGS") end)
		local loSub = label(loBtn, { Size = UDim2.fromOffset(150, 48), Position = UDim2.new(1, -164, 0, 0), Text = "", TextSize = 12, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 10 })

		-- bottom centre: status + invite
		local bar = new("Frame", { Size = UDim2.fromOffset(560, 46), AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.fromOffset(640, 636), BackgroundTransparency = 1, ZIndex = 8 }, stage)
		local function pill(x, w, text, cb, fill)
			local b = new("TextButton", { Text = text, AutoButtonColor = false, Size = UDim2.fromOffset(w, 42), Position = UDim2.fromOffset(x, 0), BackgroundColor3 = fill or COL.panel2, TextColor3 = RGB(255, 255, 255),
				Font = FUN, TextSize = 22, BorderSizePixel = 0, ZIndex = 9 }, bar)
			corner(b, 21)
			stroke(b, RGB(255, 255, 255), 2, 0.55)
			b.MouseEnter:Connect(function() uiHover() tween(b, 0.1, { BackgroundColor3 = PINK }) end)
			b.MouseLeave:Connect(function() tween(b, 0.1, { BackgroundColor3 = fill or COL.panel2 }) end)
			b.Activated:Connect(function() uiClick() cb() end)
			return b
		end
		pill(0, 270, "INVITE FRIENDS", function() if XT.mmInvite then XT.mmInvite() end end)
		exploreBtn = pill(290, 270, "EXPLORE THE HUB", function()
			UI.explore = true
			UI.refresh()
		end)

		-- status chip (top centre)
		local chip0 = new("Frame", { Size = UDim2.fromOffset(420, 36), AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.fromOffset(640, 22), BackgroundColor3 = COL.panel, BackgroundTransparency = 0.15, BorderSizePixel = 0, ZIndex = 8 }, stage)
		corner(chip0, 18)
		stroke(chip0, COL.line, 2, 0.4)
		local live = new("Frame", { Size = UDim2.fromOffset(12, 12), Position = UDim2.fromOffset(16, 12), BackgroundColor3 = COL.green, BorderSizePixel = 0, ZIndex = 9 }, chip0)
		corner(live, 6)
		TweenService:Create(live, TweenInfo.new(0.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), { BackgroundTransparency = 0.7 }):Play()
		PC.playStatus = label(chip0, { Size = UDim2.fromOffset(370, 36), Position = UDim2.fromOffset(40, 0), Text = "", TextSize = 16, TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left,
			TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 9 })

		-- right column: player card
		local pc = new("Frame", { Size = UDim2.fromOffset(340, 96), Position = UDim2.fromOffset(20, 40), BackgroundColor3 = COL.panel, BackgroundTransparency = 0.08, BorderSizePixel = 0, ZIndex = 8 }, colR)
		corner(pc, 18)
		stroke(pc, COL.line, 2, 0.3)
		local av = new("ImageLabel", { Size = UDim2.fromOffset(68, 68), Position = UDim2.fromOffset(14, 14), BackgroundColor3 = COL.panel2, BorderSizePixel = 0, ZIndex = 9 }, pc)
		corner(av, 16)
		stroke(av, YEL, 3, 0)
		task.spawn(function()
			local ok, img = pcall(function() return Players:GetUserThumbnailAsync(player.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size100x100) end)
			if ok and img then av.Image = img end
		end)
		label(pc, { Size = UDim2.fromOffset(200, 30), Position = UDim2.fromOffset(94, 10), Text = string.upper(player.DisplayName), TextSize = 26, Font = FUN, TextXAlignment = Enum.TextXAlignment.Left,
			TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 9 })
		PC.pcLevel = label(pc, { Size = UDim2.fromOffset(100, 22), Position = UDim2.fromOffset(94, 40), Text = "LEVEL 1", TextSize = 17, Font = FUN, TextColor3 = YEL, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 9 })
		PC.pcXp = label(pc, { Size = UDim2.fromOffset(120, 22), Position = UDim2.fromOffset(206, 40), Text = "0 / 0 XP", TextSize = 13, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 9 })
		local pcBg = new("Frame", { Size = UDim2.fromOffset(232, 12), Position = UDim2.fromOffset(94, 68), BackgroundColor3 = RGB(14, 15, 15), BorderSizePixel = 0, ZIndex = 9 }, pc)
		corner(pcBg, 6)
		pcBar = new("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = PINK, BorderSizePixel = 0, ZIndex = 10 }, pcBg)
		corner(pcBar, 6)
		new("UIGradient", { Color = ColorSequence.new(PINK, YEL) }, pcBar)
		PC.pcSave = label(pc, { Size = UDim2.fromOffset(10, 10), Position = UDim2.fromOffset(0, 0), Text = "", TextSize = 1, Visible = false })

		-- right column: daily reward (real, claim once a day)
		local DAILY_XP = { 100, 150, 200, 250, 300, 400, 750 }
		local dc = new("Frame", { Size = UDim2.fromOffset(340, 236), Position = UDim2.fromOffset(20, 156), BackgroundColor3 = COL.panel, BackgroundTransparency = 0.08, BorderSizePixel = 0, ZIndex = 8 }, colR)
		corner(dc, 18)
		stroke(dc, COL.line, 2, 0.3)
		label(dc, { Size = UDim2.fromOffset(300, 30), Position = UDim2.fromOffset(16, 8), Text = "DAILY REWARD", TextSize = 28, Font = FUN, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 9 })
		local dNote = label(dc, { Size = UDim2.fromOffset(310, 18), Position = UDim2.fromOffset(17, 38), Text = "", TextSize = 13, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 9 })
		local coins = {}
		for d = 1, 7 do
			local row2 = d > 4
			local cx = row2 and (16 + 43 + (d - 5) * 86) or (16 + (d - 1) * 86)
			local cy = row2 and 150 or 66
			local c = new("TextButton", { Text = "", AutoButtonColor = false, Size = UDim2.fromOffset(70, 70), Position = UDim2.fromOffset(cx, cy), BackgroundColor3 = COL.panel2, BorderSizePixel = 0, ZIndex = 9 }, dc)
			corner(c, 35)
			local cs = stroke(c, COL.line, 2, 0.2)
			local dl = label(c, { Size = UDim2.new(1, 0, 0, 14), Position = UDim2.fromOffset(0, 10), Text = "DAY " .. d, TextSize = 11, TextColor3 = COL.dim, ZIndex = 10 })
			local amt = label(c, { Size = UDim2.new(1, 0, 0, 26), Position = UDim2.fromOffset(0, 24), Text = "+" .. DAILY_XP[d], TextSize = 24, Font = FUN, TextColor3 = YEL, ZIndex = 10 })
			local st = label(c, { Size = UDim2.new(1, 0, 0, 14), Position = UDim2.fromOffset(0, 50), Text = "XP", TextSize = 11, TextColor3 = COL.dim, ZIndex = 10 })
			c.Activated:Connect(function() uiClick() Net.Daily:FireServer() end)
			coins[d] = { f = c, st = cs, dl = dl, amt = amt, tx = st }
		end
		local function refreshDaily()
			local today = os.time() // 86400
			local dl = P.daily or { last = 0, streak = 0 }
			local claimed = dl.last == today
			local cur = claimed and dl.streak or ((today - dl.last == 1) and (dl.streak % 7 + 1) or 1)
			dNote.Text = claimed and "Claimed. Come back tomorrow for the next one." or "Tap the glowing coin to claim today's XP."
			dNote.TextColor3 = claimed and COL.dim or YEL
			for d, c in ipairs(coins) do
				if d < cur or (d == cur and claimed) then
					c.f.BackgroundColor3, c.st.Color, c.tx.Text, c.tx.TextColor3, c.amt.TextColor3 = RGB(24, 92, 70), COL.green, "CLAIMED", COL.green, RGB(255, 255, 255)
				elseif d == cur then
					c.f.BackgroundColor3, c.st.Color, c.tx.Text, c.tx.TextColor3, c.amt.TextColor3 = PINK, RGB(255, 255, 255), "CLAIM", RGB(255, 255, 255), RGB(255, 255, 255)
				else
					c.f.BackgroundColor3, c.st.Color, c.tx.Text, c.tx.TextColor3, c.amt.TextColor3 = COL.panel2, COL.line, "XP", COL.dim, YEL
				end
				c.st.Thickness = (d == cur and not claimed) and 4 or 2
			end
		end

		-- right column: shop promo
		local sp = new("TextButton", { Text = "", AutoButtonColor = false, Size = UDim2.fromOffset(340, 118), Position = UDim2.fromOffset(20, 412), BackgroundColor3 = PINK, BorderSizePixel = 0, ZIndex = 8, ClipsDescendants = true }, colR)
		corner(sp, 18)
		stroke(sp, RGB(255, 255, 255), 3, 0.2)
		new("UIGradient", { Rotation = 20, Color = ColorSequence.new(RGB(255, 106, 10), RGB(96, 108, 116)) }, sp)
		label(sp, { Size = UDim2.fromOffset(150, 24), Position = UDim2.fromOffset(16, 12), Text = "NEW IN THE SHOP", TextSize = 16, Font = FUN, TextColor3 = INK, BackgroundColor3 = YEL, BackgroundTransparency = 0, ZIndex = 10 })
		local spName = label(sp, { Size = UDim2.fromOffset(310, 40), Position = UDim2.fromOffset(16, 40), Text = "", TextSize = 36, Font = FUN, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 10 })
		local spSub = label(sp, { Size = UDim2.fromOffset(310, 36), Position = UDim2.fromOffset(17, 80), Text = "", TextSize = 14, TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true, ZIndex = 10 })
		sp.MouseEnter:Connect(function() uiHover() tween(sp, 0.12, { Position = UDim2.fromOffset(12, 412) }, Enum.EasingStyle.Back) end)
		sp.MouseLeave:Connect(function() tween(sp, 0.12, { Position = UDim2.fromOffset(20, 412) }) end)
		sp.Activated:Connect(function() uiClick() showPage("STORE") end)

		-- right column: coming-soon stickers
		for i, t in ipairs({ { "CAMPAIGN", "story mode" }, { "ZOMBIES", "co-op survival" } }) do
			local s = new("Frame", { Size = UDim2.fromOffset(160, 76), Position = UDim2.fromOffset(20 + (i - 1) * 180, 552), BackgroundColor3 = COL.panel, BackgroundTransparency = 0.1, BorderSizePixel = 0, Rotation = (i == 1) and -3 or 3, ZIndex = 8 }, colR)
			corner(s, 14)
			stroke(s, YEL, 2, 0.1)
			label(s, { Size = UDim2.new(1, 0, 0, 30), Position = UDim2.fromOffset(0, 8), Text = t[1], TextSize = 26, Font = FUN, ZIndex = 9 })
			label(s, { Size = UDim2.new(1, 0, 0, 16), Position = UDim2.fromOffset(0, 38), Text = t[2], TextSize = 12, TextColor3 = COL.dim, ZIndex = 9 })
			local cs = label(s, { Size = UDim2.new(1, -24, 0, 18), Position = UDim2.fromOffset(12, 54), Text = "COMING SOON", TextSize = 14, Font = FUN, TextColor3 = INK, BackgroundColor3 = YEL, BackgroundTransparency = 0, ZIndex = 9 })
			corner(cs, 5)
		end

		-- operator picker under the soldiers (name + arrows)
		nameplate = label(stage, { Size = UDim2.fromOffset(280, 46), AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.fromOffset(640, 70), Text = "RECON", TextSize = 46, Font = FUN, TextColor3 = YEL, ZIndex = 9 })
		subplate = label(stage, { Size = UDim2.fromOffset(360, 18), AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.fromOffset(640, 118), Text = "", TextSize = 14, TextColor3 = COL.dim, ZIndex = 9 })
		do
			local function cycle(dir)
				local owned = {}
				for _, id in ipairs(OPERATOR_ORDER) do
					if operatorOwned(id, P.owned, P) then table.insert(owned, id) end
				end
				local cur = P.equip and P.equip.operator or "OP_STANDARD"
				local idx = 1
				for i, id in ipairs(owned) do if id == cur then idx = i end end
				idx = (idx - 1 + dir) % #owned + 1
				P.equip = P.equip or {}
				P.equip.operator = owned[idx]
				Net.EquipItem:FireServer("operator", owned[idx])
				build()
			end
			build()
			for _, a in ipairs({ { "<", 500, -1 }, { ">", 736, 1 } }) do
				local b = button(stage, a[1], UDim2.fromOffset(44, 44), UDim2.fromOffset(a[2], 71), PINK, function() cycle(a[3]) end, 30)
				b.ZIndex = 9
				b.Font = FUN
				corner(b, 22)
				stroke(b, RGB(255, 255, 255), 2, 0.3)
			end
		end

		-- live text
		local qStart = 0
		local ssI = 0
		local ssT = 0
		task.spawn(function()
			while true do
				local pd = WEAPONS[P.loadout and P.loadout.primary or "VX44"] or WEAPONS.VX44
				loSub.Text = pd.name
				local q, n = 0, #Players:GetPlayers()
				for _, pl in ipairs(Players:GetPlayers()) do if pl:GetAttribute("Queued") then q += 1 end end
				PC.playStatus.Text = string.format("%d online   -   %d in queue   -   %s", n, q, string.lower(tostring(State:GetAttribute("Status") or "")))
				if player:GetAttribute("Queued") == true then
					if qStart == 0 then qStart = os.clock() end
					local e = math.floor(os.clock() - qStart)
					playSub.Text = string.format("SEARCHING   %d:%02d", e // 60, e % 60)
				else
					qStart = 0
					playSub.Text = "FIND A MATCH"
				end
				if ssT % 4 == 0 then
					ssI = ssI % #STORE_ITEMS + 1
					local it = STORE_ITEMS[ssI]
					spName.Text = string.upper(it.name)
					spSub.Text = it.desc .. "   R$ " .. it.robux
				end
				ssT += 1
				pcall(refreshDaily)
				task.wait(1)
			end
		end)

		-- slide the two columns in whenever the home screen shows up
		local function intro()
			colL.Position = UDim2.fromOffset(-140, 0)
			colR.Position = UDim2.fromOffset(1020, 0)
			tween(colL, 0.55, { Position = UDim2.fromOffset(0, 0) }, Enum.EasingStyle.Back)
			tween(colR, 0.55, { Position = UDim2.fromOffset(880, 0) }, Enum.EasingStyle.Back)
		end
		homeUi:GetPropertyChangedSignal("Visible"):Connect(function() if homeUi.Visible then intro() end end)

		-- ---------- page tab strip (only while a page like Loadout or Store is open) ----------
		local nav = new("Frame", { Name = "Nav", Size = UDim2.new(1, 0, 0, 64), BackgroundTransparency = 1, Visible = false, ZIndex = 8 }, menuHolder)
		MS.nav = nav
		XT.queueNow = queueNow
		local tabs = { { "HOME", "HOME" }, { "LOADOUT", "LOADOUT" }, { "OPERATORS", "OPERATORS" }, { "STORE", "STORE" }, { "PASS", "PASS" }, { "CAREER", "CAREER" }, { "SETTINGS", "SETTINGS" } }
		local strip = new("Frame", { Size = UDim2.fromOffset(#tabs * 112 + 12, 46), AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 12), BackgroundColor3 = COL.panel,
			BackgroundTransparency = 0.05, BorderSizePixel = 0, ZIndex = 8 }, nav)
		corner(strip, 23)
		stroke(strip, COL.line, 2, 0.3)
		for i, tdef in ipairs(tabs) do
			local tb = button(strip, tdef[1], UDim2.fromOffset(106, 36), UDim2.fromOffset(6 + (i - 1) * 112, 5), COL.panel, function()
				if tdef[2] == "HOME" then showPage(nil) elseif tdef[2] == "DEPLOY" then queueNow() else showPage(tdef[2]) end
			end, 20)
			tb.BackgroundTransparency = 1
			tb.Font = FUN
			tb.ZIndex = 9
			corner(tb, 18)
			new("Frame", { Name = "Under", Size = UDim2.new(0, 0, 0, 4), AnchorPoint = Vector2.new(0.5, 1), Position = UDim2.new(0.5, 0, 1, -1), BackgroundColor3 = YEL, BorderSizePixel = 0, ZIndex = 10 }, tb)
			UI.navBtns[tdef[2]] = tb
		end
		XT.skinFonts(gui)
	end

	-- ---------- item scenes (store cards + preview) ----------
	function XT.itemSpec(it)
		local spec = {}
		local list = it.grants or { it.id }
		for _, g in ipairs(list) do
			local gi = STORE_BY_ID[g]
			if string.sub(g, 1, 3) == "OP_" then spec.opId = g
			elseif string.sub(g, 1, 2) == "C_" then spec.camoId = g
			elseif gi and gi.weapon then spec.wId = gi.weapon
			elseif gi and gi.attach then spec.attIds = { gi.attach } end
		end
		local bp = (P.loadout and WEAPONS[P.loadout.primary] and P.loadout.primary) or "VX44"
		if WEAPONS[bp].premium then bp = "VX44" end
		spec.gunId = spec.wId or bp
		if it.kind == "OPERATORS" then spec.camoId = P.equip and P.equip.camo end
		spec.focus = it.kind == "ATTACHMENTS"
		if spec.focus and WEAPONS[spec.gunId].scope then spec.gunId = "VX44" end -- snipers cannot carry optics
		return spec
	end
	-- add(inst, base, static) parents the part into a viewport; returns look-at centre + camera distance
	function XT.buildScene(spec, add)
		local def = camoDef(effDef(WEAPONS[spec.gunId], spec.attIds or {}), spec.camoId)
		if spec.opId then
			local m = XT.soldier(spec.opId)
			add(m, m:GetPivot(), false)
			add(XT.gun(def), m:GetPivot() * XT.gunHold, false)
			local ra, rb = XT.ring(OPERATORS[spec.opId].accent)
			add(ra, nil, true)
			add(rb, nil, true)
			return Vector3.new(0, 3, -0.3), 13.5
		end
		local g = XT.gun(def)
		if spec.focus then
			-- attachments: show ONLY the attachment (the gun it sits on is removed), in its real colours, lit and centred
			local hold = Instance.new("Model")
			for _, d in ipairs(g:GetChildren()) do
				if d:IsA("BasePart") and string.sub(d.Name, 1, 3) == "Att" then
					d.Parent = hold
					d.Anchored = true
					if d.Material ~= Enum.Material.Neon then
						d.Color = d.Color:Lerp(RGB(190, 205, 235), 0.45)
						d.Material = Enum.Material.SmoothPlastic
					end
				end
			end
			g:Destroy()
			if #hold:GetChildren() > 0 then
				local bcf2, sz2 = hold:GetBoundingBox()
				add(hold, hold:GetPivot(), false)
				return bcf2.Position, math.max(sz2.X, sz2.Y, sz2.Z, 0.6) * 2.4 + 1.4
			end
			hold:Destroy()
			g = XT.gun(def)
			add(g, g:GetPivot(), false)
		else
			add(g, g:GetPivot(), false)
		end
		local bcf, size = g:GetBoundingBox()
		return bcf.Position, (math.max(size.X, size.Y, size.Z) / 2) / math.tan(math.rad(17)) * 1.25 + size.Z / 2 + 0.6
	end
	XT.kindLabel = { BUNDLES = "BUNDLE", OPERATORS = "OPERATOR", WEAPONS = "WEAPON", CAMOS = "WEAPON CAMO", ATTACHMENTS = "ATTACHMENT" }

	-- ---------- STORE page (Robux) ----------
	local stPage = makePage("STORE", "STORE", "Tap any item to see it in 3D. Purchases are permanent and save to your account.")
	do
		local tabsRow = new("Frame", { Size = UDim2.fromOffset(780, 40), Position = UDim2.fromOffset(25, 92), BackgroundTransparency = 1 }, stPage)
		local grid = new("ScrollingFrame", { Size = UDim2.fromOffset(780, 470), Position = UDim2.fromOffset(25, 140), BackgroundTransparency = 1, BorderSizePixel = 0,
			ScrollBarThickness = 6, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, ScrollBarImageColor3 = COL.accent }, stPage)
		new("UIGridLayout", { CellSize = UDim2.fromOffset(250, 214), CellPadding = UDim2.fromOffset(15, 15), SortOrder = Enum.SortOrder.LayoutOrder }, grid)
		local toast = label(stPage, { Size = UDim2.fromOffset(780, 26), Position = UDim2.fromOffset(25, 616), Text = "", TextSize = 14, TextColor3 = COL.gold, Font = Enum.Font.GothamBold })
		local tabBtns, curTab = {}, "BUNDLES"
		local render, openPreview
		local thumbs = {}
		local function owns(it)
			if it.grants then
				for _, g in ipairs(it.grants) do if not (P.owned and P.owned[g]) then return false end end
				return true
			end
			return P.owned ~= nil and P.owned[it.id] == true
		end
		function render()
			for _, c in ipairs(grid:GetChildren()) do
				if c:IsA("GuiObject") then c:Destroy() end
			end
			table.clear(thumbs)
			for id, b in pairs(tabBtns) do
				b.BackgroundColor3 = (id == curTab) and COL.accent or RGB(14, 20, 40)
				b.TextColor3 = (id == curTab) and RGB(20, 14, 8) or COL.text
			end
			local order = 0
			for _, it in ipairs(STORE_ITEMS) do
				if it.kind == curTab then
					order += 1
					local have = owns(it)
					local isEq = P.equip and (P.equip.operator == it.id or P.equip.camo == it.id)
					local card = button(grid, "", UDim2.fromOffset(250, 214), UDim2.new(), RGB(12, 18, 38), function() openPreview(it) end)
					card.LayoutOrder = order
					corner(card, 4)
					stroke(card, it.accent, 1.5, 0.4)
					local vpf = new("ViewportFrame", { Size = UDim2.fromOffset(250, 130), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 2,
						Ambient = RGB(150, 158, 200), LightColor = RGB(255, 210, 170), LightDirection = Vector3.new(-1, -0.6, -0.9) }, card)
					-- a UIGradient on a ViewportFrame tints (blackens) the 3D content, so the backdrop is a separate frame
					local bgf = new("Frame", { Size = vpf.Size, Position = vpf.Position, BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 1 }, card)
					new("UIGradient", { Rotation = 90, Color = ColorSequence.new({ ColorSequenceKeypoint.new(0, RGB(30, 36, 76)), ColorSequenceKeypoint.new(1, RGB(8, 10, 24)) }) }, bgf)
					vpf.BackgroundTransparency = 1
					local vcam = new("Camera", { FieldOfView = 34 }, vpf)
					vpf.CurrentCamera = vcam
					local entry = { items = {}, off = order * 0.9 }
					local c, d = XT.buildScene(XT.itemSpec(it), function(inst, base, static)
						inst.Parent = vpf
						if not static then table.insert(entry.items, { inst = inst, base = base }) end
					end)
					entry.c = c
					vcam.CFrame = CFrame.lookAt(c + Vector3.new(0, c.Y > 1 and 0.5 or 0.2, d * 0.95), c)
					table.insert(thumbs, entry)
					label(card, { Size = UDim2.fromOffset(110, 16), Position = UDim2.fromOffset(8, 6), Text = XT.kindLabel[it.kind] or it.kind, TextSize = 10, TextColor3 = RGB(255, 255, 255),
						BackgroundColor3 = it.accent, BackgroundTransparency = 0.35, ZIndex = 3 })
					label(card, { Size = UDim2.fromOffset(230, 26), Position = UDim2.fromOffset(10, 134), Text = it.name, TextSize = 19, Font = Enum.Font.GothamBlack, TextColor3 = it.accent,
						TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 3 })
					label(card, { Size = UDim2.fromOffset(230, 16), Position = UDim2.fromOffset(10, 160), Text = it.desc, TextSize = 10, TextColor3 = COL.dim, Font = Enum.Font.Gotham,
						TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 3 })
					if have then
						label(card, { Size = UDim2.fromOffset(110, 26), Position = UDim2.fromOffset(10, 182), Text = isEq and "EQUIPPED" or "OWNED", TextSize = 14, TextColor3 = COL.green,
							TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 3 })
					else
						label(card, { Size = UDim2.fromOffset(110, 26), Position = UDim2.fromOffset(10, 182), Text = "R$ " .. it.robux, TextSize = 17, TextColor3 = RGB(255, 205, 70),
							TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 3 })
					end
					label(card, { Size = UDim2.fromOffset(110, 26), Position = UDim2.fromOffset(130, 182), Text = "VIEW >", TextSize = 13, TextColor3 = COL.text,
						TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 3 })
				end
			end
			local soon = new("Frame", { LayoutOrder = 9999, BackgroundColor3 = RGB(12, 16, 34), BackgroundTransparency = 0.3, BorderSizePixel = 0 }, grid)
			corner(soon, 4)
			stroke(soon, COL.line, 1.5, 0.3)
			label(soon, { Size = UDim2.fromScale(1, 0.3), Position = UDim2.fromScale(0, 0.3), Text = "MORE COMING SOON!", TextSize = 20, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent })
			label(soon, { Size = UDim2.fromScale(1, 0.2), Position = UDim2.fromScale(0, 0.56), Text = "NEW ITEMS EVERY SEASON", TextSize = 11, TextColor3 = COL.dim })
		end
		for i, tname in ipairs(STORE_TABS) do
			local tb = button(tabsRow, tname, UDim2.fromOffset(148, 36), UDim2.fromOffset((i - 1) * 154, 2), RGB(14, 20, 40), function()
				curTab = tname
				render()
			end, 13)
			tb.Font = Enum.Font.GothamBlack
			tabBtns[tname] = tb
		end

		-- ---------- preview overlay (3D view of the item + buy / equip) ----------
		local pv = new("Frame", { Size = UDim2.fromOffset(780, 536), Position = UDim2.fromOffset(25, 92), BackgroundColor3 = RGB(6, 9, 20), BorderSizePixel = 0, Visible = false, ZIndex = 30 }, stPage)
		corner(pv, 4)
		local pvStroke = stroke(pv, COL.accent, 2, 0.2)
		local pvp = new("ViewportFrame", { Size = UDim2.fromOffset(430, 516), Position = UDim2.fromOffset(10, 10), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 31,
			Ambient = RGB(150, 158, 200), LightColor = RGB(255, 210, 170), LightDirection = Vector3.new(-1, -0.6, -0.9) }, pv)
		-- a UIGradient on a ViewportFrame tints (blackens) the 3D content, so the backdrop is a separate frame
		local bgf = new("Frame", { Size = pvp.Size, Position = pvp.Position, BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 30 }, pv)
		new("UIGradient", { Rotation = 90, Color = ColorSequence.new({ ColorSequenceKeypoint.new(0, RGB(34, 40, 84)), ColorSequenceKeypoint.new(1, RGB(8, 10, 24)) }) }, bgf)
		pvp.BackgroundTransparency = 1
		local pcam = new("Camera", { FieldOfView = 34 }, pvp)
		pvp.CurrentCamera = pcam
		local pvName = label(pv, { Size = UDim2.fromOffset(310, 40), Position = UDim2.fromOffset(455, 14), Text = "", TextSize = 28, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left,
			TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 32 })
		local pvKind = label(pv, { Size = UDim2.fromOffset(310, 18), Position = UDim2.fromOffset(455, 54), Text = "", RichText = true, TextSize = 12, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 32 })
		local pvDesc = label(pv, { Size = UDim2.fromOffset(310, 110), Position = UDim2.fromOffset(455, 80), Text = "", RichText = true, TextSize = 14, TextColor3 = COL.text, Font = Enum.Font.Gotham,
			TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true, ZIndex = 32 })
		local pvInfo = label(pv, { Size = UDim2.fromOffset(310, 120), Position = UDim2.fromOffset(455, 196), Text = "", RichText = true, TextSize = 13, TextColor3 = COL.accent2, Font = Enum.Font.GothamMedium,
			TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true, ZIndex = 32 })
		local pvPrice = label(pv, { Size = UDim2.fromOffset(310, 36), Position = UDim2.fromOffset(455, 336), Text = "", TextSize = 26, Font = Enum.Font.GothamBlack, TextColor3 = RGB(255, 205, 70),
			TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 32 })
		local pvNote = label(pv, { Size = UDim2.fromOffset(310, 30), Position = UDim2.fromOffset(455, 372), Text = "", TextSize = 11, TextColor3 = COL.dim, Font = Enum.Font.Gotham,
			TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true, ZIndex = 32 })
		local pvBuy = button(pv, "BUY", UDim2.fromOffset(310, 56), UDim2.fromOffset(455, 414), COL.accent, nil, 24)
		pvBuy.Font = Enum.Font.GothamBlack
		pvBuy.TextColor3 = RGB(20, 14, 8)
		pvBuy.ZIndex = 32
		local pvBack = button(pv, "BACK TO STORE", UDim2.fromOffset(310, 40), UDim2.fromOffset(455, 480), RGB(16, 22, 44), function() pv.Visible = false end, 15)
		pvBack.ZIndex = 32
		-- purchase confirmation
		local cf = new("Frame", { Size = UDim2.fromOffset(420, 220), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), BackgroundColor3 = RGB(10, 14, 30), BorderSizePixel = 0,
			Visible = false, ZIndex = 50 }, pv)
		corner(cf, 4)
		stroke(cf, COL.gold, 2, 0.1)
		label(cf, { Size = UDim2.fromOffset(400, 30), Position = UDim2.fromOffset(10, 12), Text = "CONFIRM PURCHASE", TextSize = 22, Font = Enum.Font.GothamBlack, TextColor3 = COL.gold, ZIndex = 51 })
		local cfName = label(cf, { Size = UDim2.fromOffset(400, 30), Position = UDim2.fromOffset(10, 50), Text = "", TextSize = 20, ZIndex = 51 })
		local cfPrice = label(cf, { Size = UDim2.fromOffset(400, 30), Position = UDim2.fromOffset(10, 82), Text = "", TextSize = 24, Font = Enum.Font.GothamBlack, TextColor3 = RGB(255, 205, 70), ZIndex = 51 })
		local cfNote = label(cf, { Size = UDim2.fromOffset(390, 40), Position = UDim2.fromOffset(15, 116), Text = "", TextSize = 12, TextColor3 = COL.dim, Font = Enum.Font.Gotham, TextWrapped = true, ZIndex = 51 })
		local cur
		local cfYes = button(cf, "CONFIRM", UDim2.fromOffset(190, 46), UDim2.fromOffset(15, 160), COL.green, function()
			cf.Visible = false
			if cur then Net.Buy:FireServer(cur.id) end
		end, 18)
		cfYes.ZIndex = 51
		cfYes.TextColor3 = RGB(10, 20, 10)
		local cfNo = button(cf, "CANCEL", UDim2.fromOffset(190, 46), UDim2.fromOffset(215, 160), COL.red, function() cf.Visible = false end, 18)
		cfNo.ZIndex = 51

		local stage = { items = {}, c = Vector3.new() }
		local function clearStage()
			for _, it in ipairs(stage.items) do it.inst:Destroy() end
			stage.items = {}
		end
		local function addStage(inst, base)
			inst.Parent = pvp
			table.insert(stage.items, { inst = inst, base = base })
		end
		local function nameOf(id)
			local x = STORE_BY_ID[id]
			return x and x.name or id
		end
		local function stateUi()
			if not cur then return end
			local it = cur
			local have = owns(it)
			local isEq = P.equip and (P.equip.operator == it.id or P.equip.camo == it.id)
			pvBuy.Visible = true
			if have then
				pvPrice.Text = "OWNED"
				pvPrice.TextColor3 = COL.green
				pvNote.Text = ""
				if it.kind == "OPERATORS" or it.kind == "CAMOS" then
					pvBuy.Text = isEq and "EQUIPPED" or "EQUIP"
					pvBuy.BackgroundColor3 = isEq and COL.line or COL.green
				else
					pvBuy.Text = "GO TO LOADOUT"
					pvBuy.BackgroundColor3 = COL.accent2
				end
			else
				pvPrice.Text = "R$ " .. it.robux
				pvPrice.TextColor3 = RGB(255, 205, 70)
				pvBuy.Text = "BUY  R$ " .. it.robux
				pvBuy.BackgroundColor3 = COL.accent
				pvNote.Text = (STORE_PRODUCTS[it.id] or 0) == 0 and "Test mode: this item is free in Studio until you set its Developer Product ID in STORE_PRODUCTS." or "Paid with Robux through Roblox. Permanent and saved to your account."
			end
		end
		pvBuy.Activated:Connect(function()
			if not cur then return end
			local it = cur
			if owns(it) then
				if it.kind == "OPERATORS" or it.kind == "CAMOS" then
					P.equip = P.equip or {}
					local slot = it.kind == "OPERATORS" and "operator" or "camo"
					P.equip[slot] = it.id
					Net.EquipItem:FireServer(slot, it.id)
					stateUi()
					render()
					for _, f in ipairs(profileHooks) do pcall(f) end
				else
					pv.Visible = false
					showPage("LOADOUT")
				end
				return
			end
			cfName.Text = it.name
			cfPrice.Text = "R$ " .. it.robux
			cfNote.Text = (STORE_PRODUCTS[it.id] or 0) == 0 and "Test purchase (no Robux charged). Set the Developer Product ID to enable real Robux purchases." or "Roblox will ask you to confirm and charge your Robux balance."
			cf.Visible = true
		end)
		-- slide controls (bundles show one included item per slide)
		local slides, slideI = {}, 1
		TweenService:Create(pvStroke, TweenInfo.new(1.3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), { Transparency = 0.75 }):Play()
		local function rarityOf(it)
			local r = it.robux or 0
			if r >= 900 then return "LEGENDARY", RGB(255, 205, 70) end
			if r >= 500 then return "EPIC", RGB(96, 108, 116) end
			if r >= 250 then return "RARE", RGB(80, 170, 255) end
			return "UNCOMMON", RGB(110, 255, 140)
		end
		local pvBadge = label(pv, { Size = UDim2.fromOffset(120, 24), Position = UDim2.fromOffset(18, 18), Text = "", TextSize = 13, Font = Enum.Font.GothamBlack, BackgroundColor3 = COL.red,
			BackgroundTransparency = 0.05, ZIndex = 35, Visible = false })
		corner(pvBadge, 4)
		local tiles = {}
		XT.kindShort = { OPERATORS = "OPERATOR", CAMOS = "CAMO", WEAPONS = "WEAPON", ATTACHMENTS = "ATTACH", BUNDLES = "ALL" }
		local slideLbl = label(pv, { Size = UDim2.fromOffset(430, 24), Position = UDim2.fromOffset(10, 486), Text = "", TextSize = 14, Font = Enum.Font.GothamBold, ZIndex = 33 })
		local arrows = {}
		local statRows = {}
		for i, nm in ipairs({ "DAMAGE", "RANGE", "FIRE RATE", "MOBILITY" }) do
			local y = 196 + (i - 1) * 24
			local l = label(pv, { Size = UDim2.fromOffset(90, 16), Position = UDim2.fromOffset(455, y), Text = nm, TextSize = 11, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 32 })
			local bg = new("Frame", { Size = UDim2.fromOffset(210, 8), Position = UDim2.fromOffset(555, y + 4), BackgroundColor3 = RGB(24, 30, 54), BorderSizePixel = 0, ZIndex = 32 }, pv)
			local fill = new("Frame", { Size = UDim2.fromScale(0.5, 1), BackgroundColor3 = COL.accent2, BorderSizePixel = 0, ZIndex = 33 }, bg)
			statRows[i] = { l = l, bg = bg, fill = fill }
		end
		local function infoFor(sub)
			local k = sub.kind
			if k == "OPERATORS" then
				return "<b>COSMETIC OPERATOR</b>\nOther players see you in this armour.\nNo stat changes, pure style.", nil
			elseif k == "CAMOS" then
				return "<b>COSMETIC FINISH</b>\nShown here on your primary weapon.\nApplies to every gun you own.", nil
			elseif k == "ATTACHMENTS" then
				local a = ATTACH_BY_ID[sub.attach]
				return "<b>SLOT: " .. (a and a.cat or "?") .. "</b>\n" .. (a and a.desc or "") .. "\nUnlocks it in the Gunsmith.", nil
			elseif k == "WEAPONS" then
				local w = WEAPONS[sub.weapon]
				if w then
					return string.format("<b>MAG</b> %d / %d     <b>RELOAD</b> %.1fs\n<b>%s</b>", w.mag, w.reserve, w.reload, w.burst and "BURST" or (w.auto and "FULL-AUTO" or "SEMI-AUTO")), w
				end
			end
			return "", nil
		end
		local function showSlide(i)
			slideI = i
			local sl = slides[i]
			local sub = sl.sub
			clearStage()
			pvStroke.Color = cur.accent
			pvName.Text = sl.overview and cur.name or sub.name
			pvName.TextColor3 = sub.accent
			do
				local rn, rc = rarityOf(cur.grants and cur or sub)
				pvKind.Text = string.format('%s     <font color="#%s"><b>%s</b></font>', (sl.overview and (XT.kindLabel[cur.kind] or cur.kind) or (XT.kindLabel[sub.kind] or sub.kind)), rc:ToHex(), rn)
			end
			if cur.grants then
				local worth = 0
				for _, g in ipairs(cur.grants) do worth += (STORE_BY_ID[g] and STORE_BY_ID[g].robux or 0) end
				local pct = worth > 0 and math.floor((1 - cur.robux / worth) * 100 + 0.5) or 0
				pvBadge.Visible = pct > 0
				pvBadge.Text = "SAVE " .. pct .. "%"
			else
				pvBadge.Visible = false
			end
			for ti, tb in ipairs(tiles) do
				local slx = slides[ti]
				tb.Visible = slx ~= nil and #slides > 1
				if slx then
					tb.Text = slx.overview and "FULL\nPACK" or ((XT.kindShort[slx.sub.kind] or "") .. "\n" .. string.sub(slx.sub.name, 1, 9))
					tb.TextColor3 = slx.sub.accent
					local st = tb:FindFirstChildOfClass("UIStroke")
					if st then
						st.Color = (ti == i) and slx.sub.accent or COL.line
						st.Transparency = (ti == i) and 0 or 0.4
						st.Thickness = (ti == i) and 3 or 1.5
					end
				end
			end
			pvDesc.Text = sub.lore or sub.desc or ""
			local info, w = "", nil
			if sl.overview then
				local names, worth = {}, 0
				for _, g in ipairs(cur.grants or {}) do
					local gi = STORE_BY_ID[g]
					worth += gi and gi.robux or 0
					table.insert(names, string.format('<font color="#%s"><b>%s</b></font>   <font color="#8C98AC">%s</font>   <s>R$ %d</s>', (gi and gi.accent or COL.text):ToHex(), gi and gi.name or g,
						XT.kindLabel[gi and gi.kind or ""] or "", gi and gi.robux or 0))
				end
				info = "<b>THIS PACK INCLUDES</b>\n" .. table.concat(names, "\n") .. string.format('\n<b>VALUE R$ %d</b>   NOW <font color="#FFCD46"><b>R$ %d</b></font>', worth, cur.robux)
			else
				info, w = infoFor(sub)
			end
			pvInfo.Text = info
			for r = 1, 4 do
				local row = statRows[r]
				local on = w ~= nil and w.stats ~= nil
				row.l.Visible, row.bg.Visible = on, on
				if on then row.fill.Size = UDim2.fromScale(math.clamp(w.stats[r] or 0.5, 0.05, 1), 1) end
			end
			pvInfo.Position = (w ~= nil and w.stats ~= nil) and UDim2.fromOffset(455, 296) or UDim2.fromOffset(455, 190)
			pvInfo.Size = (w ~= nil and w.stats ~= nil) and UDim2.fromOffset(310, 40) or UDim2.fromOffset(310, 140)
			local spec = XT.itemSpec(sl.overview and cur or sub)
			if not sl.overview and cur.grants and (sub.kind == "CAMOS" or sub.kind == "ATTACHMENTS") then
				local comp = XT.itemSpec(cur)
				if comp.wId then spec.gunId = comp.wId end
			end
			local center, dist = XT.buildScene(spec, function(inst, base, static)
				inst.Parent = pvp
				table.insert(stage.items, { inst = inst, base = base, static = static })
			end)
			stage.c, stage.dist = center, dist
			pcam.CFrame = CFrame.lookAt(center + Vector3.new(0, center.Y > 1 and 0.4 or 0.25, dist), center)
			slideLbl.Text = (#slides > 1) and string.format("%d / %d     %s", i, #slides, sl.overview and "FULL PACK" or sub.name) or ""
			pvDesc.Size = cur.grants and UDim2.fromOffset(310, 76) or UDim2.fromOffset(310, 110)
			slideLbl.TextColor3 = sub.accent
			for _, ar in ipairs(arrows) do ar.Visible = #slides > 1 end
		end
		for _, a in ipairs({ { "<", 18, -1 }, { ">", 372, 1 } }) do
			local ab = button(pv, a[1], UDim2.fromOffset(50, 60), UDim2.fromOffset(a[2], 230), RGB(10, 16, 40), function()
				if #slides > 1 then showSlide((slideI - 1 + a[3]) % #slides + 1) end
			end, 28)
			ab.ZIndex = 34
			ab.Font = Enum.Font.GothamBlack
			ab.BackgroundTransparency = 0.25
			stroke(ab, RGB(70, 150, 255), 2, 0.3)
			table.insert(arrows, ab)
		end
		for i = 1, 6 do
			local tb = button(pv, "", UDim2.fromOffset(64, 40), UDim2.fromOffset(18 + (i - 1) * 70, 436), RGB(10, 16, 40), function()
				if slides[i] then showSlide(i) end
			end, 10)
			tb.ZIndex = 34
			tb.BackgroundTransparency = 0.15
			tb.Font = Enum.Font.GothamBlack
			tb.Visible = false
			stroke(tb, COL.line, 2, 0.2)
			tiles[i] = tb
		end
		function openPreview(it)
			cur = it
			cf.Visible = false
			slides = {}
			if it.grants then
				table.insert(slides, { sub = it, overview = true })
				for _, g in ipairs(it.grants) do
					if STORE_BY_ID[g] then table.insert(slides, { sub = STORE_BY_ID[g] }) end
				end
			else
				slides = { { sub = it } }
			end
			showSlide(1)
			stateUi()
			pv.Visible = true
		end
		XT.openStoreItem = function(it)
			curTab = it.kind
			render()
			openPreview(it)
		end
		RunService.RenderStepped:Connect(function()
			if stPage.Visible and not pv.Visible then
				local tt = os.clock()
				for _, th in ipairs(thumbs) do
					local rot = CFrame.new(th.c) * CFrame.Angles(0, tt * 0.7 + th.off, 0) * CFrame.new(-th.c)
					for _, it in ipairs(th.items) do it.inst:PivotTo(rot * it.base) end
				end
			end
			if not pv.Visible or not stPage.Visible then return end
			local yaw = os.clock() * 0.9
			local rot = CFrame.new(stage.c) * CFrame.Angles(0, yaw, 0) * CFrame.new(-stage.c)
			for _, it in ipairs(stage.items) do
				if not it.static then it.inst:PivotTo(rot * it.base) end
			end
		end)
		stPage:GetPropertyChangedSignal("Visible"):Connect(function()
			if stPage.Visible then
				render()
			else
				pv.Visible = false
				cf.Visible = false
			end
		end)
		table.insert(profileHooks, function()
			if stPage.Visible then
				render()
				if pv.Visible then stateUi() end
			end
		end)
		render()
		Net.StoreMsg.OnClientEvent:Connect(function(text, ok)
			toast.Text = tostring(text or "")
			toast.TextColor3 = ok and COL.green or COL.red
			task.delay(4, function() if toast.Text == tostring(text or "") then toast.Text = "" end end)
		end)
	end

	-- ---------- OPERATORS page ----------
	local opPage = makePage("OPERATORS", "OPERATORS", "Choose who you fight as. Earn some by levelling up and finishing challenges. Buy the rest in the store.")
	do
		local grid = new("ScrollingFrame", { Size = UDim2.fromOffset(780, 536), Position = UDim2.fromOffset(25, 92), BackgroundTransparency = 1, BorderSizePixel = 0,
			ScrollBarThickness = 6, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, ScrollBarImageColor3 = COL.accent }, opPage)
		new("UIGridLayout", { CellSize = UDim2.fromOffset(250, 262), CellPadding = UDim2.fromOffset(15, 15), SortOrder = Enum.SortOrder.LayoutOrder }, grid)
		local thumbs = {}
		local msg = label(opPage, { Size = UDim2.fromOffset(780, 22), Position = UDim2.fromOffset(25, 632), Text = "", TextSize = 13, TextColor3 = COL.gold })
		local function howText(id)
			local op = OPERATORS[id]
			if op.free then return "STARTER OPERATOR" end
			local r = op.req
			if r then
				if r.kind == "level" then return "REACH LEVEL " .. r.n end
				if r.kind == "kills" then return "GET " .. r.n .. " KILLS" end
				if r.kind == "headshots" then return "LAND " .. r.n .. " HEADSHOTS" end
				if r.kind == "wins" then return "WIN " .. r.n .. " MATCHES" end
			end
			local it = STORE_BY_ID[id]
			return "R$ " .. (it and it.robux or "?") .. "  IN THE STORE"
		end
		local function render()
			for _, c in ipairs(grid:GetChildren()) do
				if c:IsA("GuiObject") then c:Destroy() end
			end
			table.clear(thumbs)
			local eqId = P.equip and P.equip.operator or "OP_STANDARD"
			for i, id in ipairs(OPERATOR_ORDER) do
				local op = OPERATORS[id]
				local have = operatorOwned(id, P.owned, P)
				local card = button(grid, "", UDim2.fromOffset(250, 262), UDim2.new(), RGB(12, 18, 38), function()
					if have then
						P.equip = P.equip or {}
						P.equip.operator = id
						Net.EquipItem:FireServer("operator", id)
						for _, f in ipairs(profileHooks) do pcall(f) end
						msg.Text = op.name .. " EQUIPPED"
					elseif op.req then
						msg.Text = op.name .. " IS LOCKED: " .. howText(id)
					else
						showPage("STORE")
						if XT.openStoreItem and STORE_BY_ID[id] then XT.openStoreItem(STORE_BY_ID[id]) end
					end
				end)
				card.LayoutOrder = i
				stroke(card, op.accent, eqId == id and 3 or 1.5, eqId == id and 0 or 0.45)
				local vpf = new("ViewportFrame", { Size = UDim2.fromOffset(250, 150), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 2,
					Ambient = RGB(150, 158, 200), LightColor = RGB(255, 210, 170), LightDirection = Vector3.new(-1, -0.6, -0.9) }, card)
				-- a UIGradient on a ViewportFrame tints (blackens) the 3D content, so the backdrop is a separate frame
				local bgf = new("Frame", { Size = vpf.Size, Position = vpf.Position, BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 1 }, card)
				new("UIGradient", { Rotation = 90, Color = ColorSequence.new({ ColorSequenceKeypoint.new(0, RGB(30, 36, 76)), ColorSequenceKeypoint.new(1, RGB(8, 10, 24)) }) }, bgf)
				vpf.BackgroundTransparency = 1
				local vcam = new("Camera", { FieldOfView = 34 }, vpf)
				vpf.CurrentCamera = vcam
				local m = XT.soldier(id)
				m.Parent = vpf
				local ra, rb = XT.ring(op.accent)
				ra.Parent, rb.Parent = vpf, vpf
				vcam.CFrame = CFrame.lookAt(Vector3.new(0, 3.6, 13.8), Vector3.new(0, 2.9, 0))
				table.insert(thumbs, { m = m, off = i * 0.7, locked = not have })
				if not have then
					for _, d in ipairs(m:GetDescendants()) do
						if d:IsA("BasePart") then d.Color = d.Color:Lerp(RGB(0, 0, 0), 0.55) end
					end
				end
				label(card, { Size = UDim2.fromOffset(226, 28), Position = UDim2.fromOffset(12, 156), Text = op.name, TextSize = 22, Font = Enum.Font.GothamBlack,
					TextColor3 = have and op.accent or COL.dim, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 3 })
				label(card, { Size = UDim2.fromOffset(226, 18), Position = UDim2.fromOffset(12, 184), Text = op.desc, TextSize = 11, TextColor3 = COL.dim, Font = Enum.Font.Gotham,
					TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 3 })
				local st, stCol
				if eqId == id then st, stCol = "EQUIPPED", COL.green
				elseif have then st, stCol = "UNLOCKED   TAP TO EQUIP", COL.green
				else st, stCol = "LOCKED   " .. howText(id), COL.red end
				label(card, { Size = UDim2.fromOffset(226, 20), Position = UDim2.fromOffset(12, 208), Text = st, TextSize = 12, TextColor3 = stCol, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 3 })
				if op.req and not have then
					local cur, need = operatorProgress(op, P)
					local bg = new("Frame", { Size = UDim2.fromOffset(226, 8), Position = UDim2.fromOffset(12, 234), BackgroundColor3 = RGB(20, 26, 48), BorderSizePixel = 0, ZIndex = 3 }, card)
					new("Frame", { Size = UDim2.fromScale(math.clamp(cur / math.max(1, need), 0, 1), 1), BackgroundColor3 = op.accent, BorderSizePixel = 0, ZIndex = 4 }, bg)
					label(card, { Size = UDim2.fromOffset(226, 14), Position = UDim2.fromOffset(12, 244), Text = cur .. " / " .. need, TextSize = 10, TextColor3 = COL.dim,
						TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 3 })
				end
			end
		end
		opPage:GetPropertyChangedSignal("Visible"):Connect(function() if opPage.Visible then render() end end)
		table.insert(profileHooks, function() if opPage.Visible then render() end end)
		RunService.RenderStepped:Connect(function()
			if not opPage.Visible then return end
			local t = os.clock()
			for _, th in ipairs(thumbs) do
				if th.m.Parent then th.m:PivotTo(CFrame.new(0, 3.2, 0) * CFrame.Angles(0, math.pi + math.sin(t * 0.8 + th.off) * 0.9 + t * 0.35, 0)) end
			end
		end)
		-- announce operators that unlock by progress
		table.insert(profileHooks, function()
			if not XT.gotProfile then return end
			local now = {}
			for _, id in ipairs(OPERATOR_ORDER) do now[id] = operatorOwned(id, P.owned, P) end
			if XT.known and XT.banner then
				for id, v in pairs(now) do
					if v and not XT.known[id] then XT.banner("OPERATOR UNLOCKED   " .. OPERATORS[id].name, "good") end
				end
			end
			XT.known = now
		end)
	end

	-- ---------- DEPLOY page ----------
	local dep = makePage("DEPLOY", "MULTIPLAYER", "Queue up with your squad. Everyone in the queue votes on the map and mode.")
	local statusCard = new("Frame", { Size = UDim2.fromOffset(780, 116), Position = UDim2.fromOffset(25, 94), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0 }, dep)
	new("UIGradient", { Rotation = 0, Color = ColorSequence.new(RGB(46, 60, 61), RGB(18, 24, 25)) }, statusCard)
	corner(statusCard, 8)
	stroke(statusCard, COL.accent, 1.5, 0.5)
	new("Frame", { Size = UDim2.fromOffset(6, 116), BackgroundColor3 = COL.accent2, BorderSizePixel = 0 }, statusCard)
	TS.stTitle = label(statusCard, { Size = UDim2.fromOffset(500, 36), Position = UDim2.fromOffset(24, 14), Text = "READY TO DEPLOY", TextSize = 28, Font = Enum.Font.GothamBlack,
		TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left })
	TS.stSub = label(statusCard, { Size = UDim2.fromOffset(480, 50), Position = UDim2.fromOffset(24, 52), Text = "", TextSize = 13,
		TextColor3 = COL.accent, Font = Enum.Font.GothamMedium, TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top,
		TextWrapped = true })
	local queueBtn = button(statusCard, "QUEUE FOR MATCH", UDim2.fromOffset(240, 56), UDim2.fromOffset(520, 12), COL.accent2, function()
		local want = not (player:GetAttribute("Queued") == true)
		Net.Queue:FireServer(want)
	end, 18)
	queueBtn.TextColor3 = RGB(14, 18, 18)
	queueBtn.Font = Enum.Font.GothamBlack
	local invBtn = button(statusCard, "INVITE FRIENDS", UDim2.fromOffset(240, 32), UDim2.fromOffset(520, 74), RGB(30, 40, 41), function() if XT.mmInvite then XT.mmInvite() end end, 13)
	invBtn.Font = Enum.Font.GothamBlack
	stroke(invBtn, COL.accent, 1.5, 0.4)

	-- game modes: icon art + rule chip
	label(dep, { Size = UDim2.fromOffset(780, 22), Position = UDim2.fromOffset(25, 224), Text = "GAME MODES", TextSize = 14, TextColor3 = COL.accent2, Font = Enum.Font.GothamBlack,
		TextXAlignment = Enum.TextXAlignment.Left })
	local modeChip = { TDM = "FIRST TO 50", FFA = "FIRST TO 25", GUN = "FINISH THE LADDER", FREE = "NO COMBAT" }
	for i, mid in ipairs(MODE_ORDER) do
		local m = MODES[mid]
		local c = new("Frame", { Size = UDim2.fromOffset(186, 112), Position = UDim2.fromOffset(25 + (i - 1) * 192, 248), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ClipsDescendants = true }, dep)
		new("UIGradient", { Rotation = 90, Color = ColorSequence.new(RGB(54, 66, 67), RGB(18, 23, 24)) }, c)
		corner(c, 8)
		stroke(c, COL.line, 1, 0.3)
		-- mini icon (top right)
		local ix = 186 - 50
		if mid == "TDM" then
			new("Frame", { Size = UDim2.fromOffset(34, 8), Position = UDim2.fromOffset(ix, 44), BackgroundColor3 = RGB(255, 140, 30), BorderSizePixel = 0 }, c)
			new("Frame", { Size = UDim2.fromOffset(34, 8), Position = UDim2.fromOffset(ix, 58), BackgroundColor3 = RGB(0, 200, 255), BorderSizePixel = 0 }, c)
			new("Frame", { Size = UDim2.fromOffset(22, 8), Position = UDim2.fromOffset(ix, 72), BackgroundColor3 = RGB(255, 140, 30), BorderSizePixel = 0 }, c)
		elseif mid == "FFA" then
			for k = 0, 5 do
				local d = new("Frame", { Size = UDim2.fromOffset(10, 10), Position = UDim2.fromOffset(ix + (k % 3) * 14, 44 + math.floor(k / 3) * 16), BackgroundColor3 = COL.accent2, BorderSizePixel = 0 }, c)
				new("UICorner", { CornerRadius = UDim.new(1, 0) }, d)
			end
		elseif mid == "GUN" then
			for k = 0, 3 do new("Frame", { Size = UDim2.fromOffset(8, 10 + k * 8), AnchorPoint = Vector2.new(0, 1), Position = UDim2.fromOffset(ix + k * 12, 80), BackgroundColor3 = COL.accent2, BorderSizePixel = 0 }, c) end
		else
			-- free roam: a compass ring with a needle
			local ring = new("Frame", { Size = UDim2.fromOffset(34, 34), Position = UDim2.fromOffset(ix, 44), BackgroundTransparency = 1, BorderSizePixel = 0 }, c)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, ring)
			new("UIStroke", { Color = COL.accent2, Thickness = 2 }, ring)
			local nd = new("Frame", { Size = UDim2.fromOffset(4, 22), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(17, 17), BackgroundColor3 = COL.accent2, BorderSizePixel = 0, Rotation = 35 }, ring)
		end
		label(c, { Size = UDim2.fromOffset(170, 26), Position = UDim2.fromOffset(14, 10), Text = m.name, TextSize = 14, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left })
		label(c, { Size = UDim2.fromOffset(120, 40), Position = UDim2.fromOffset(14, 38), Text = m.desc, TextSize = 11, TextColor3 = COL.dim, Font = Enum.Font.Gotham,
			TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true })
		local ch = label(c, { Size = UDim2.fromOffset(#modeChip[mid] * 8 + 16, 20), Position = UDim2.fromOffset(14, 82), Text = modeChip[mid], TextSize = 10, Font = Enum.Font.GothamBlack,
			TextColor3 = RGB(14, 18, 18), BackgroundColor3 = COL.accent2, BackgroundTransparency = 0 })
		corner(ch, 4)
	end
	-- maps: real previews
	label(dep, { Size = UDim2.fromOffset(780, 22), Position = UDim2.fromOffset(25, 372), Text = "MAPS", TextSize = 14, TextColor3 = COL.accent2, Font = Enum.Font.GothamBlack,
		TextXAlignment = Enum.TextXAlignment.Left })
	XT.deployThumbs = {}
	for i, mid in ipairs(MAP_ORDER) do
		local m = MAPS[mid]
		local c = new("Frame", { Size = UDim2.fromOffset(250, 196), Position = UDim2.fromOffset(25 + (i - 1) * 265, 396), BackgroundColor3 = RGB(18, 23, 24), BorderSizePixel = 0, ClipsDescendants = true }, dep)
		corner(c, 8)
		stroke(c, m.accent, 1.5, 0.5)
		local th = new("Frame", { Size = UDim2.fromOffset(250, 118), BackgroundColor3 = RGB(8, 10, 10), BorderSizePixel = 0, ClipsDescendants = true }, c)
		XT.deployThumbs[mid] = th
		local sh = new("Frame", { Size = UDim2.fromOffset(250, 118), BackgroundColor3 = RGB(0, 0, 0), BorderSizePixel = 0, ZIndex = 50 }, c)
		new("UIGradient", { Rotation = 90, Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0.35) }) }, sh)
		label(c, { Size = UDim2.fromOffset(230, 26), Position = UDim2.fromOffset(12, 92), Text = m.name, TextSize = 18, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 51 })
		label(c, { Size = UDim2.fromOffset(230, 16), Position = UDim2.fromOffset(12, 124), Text = m.tag, TextSize = 10, TextColor3 = m.accent, Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Left })
		label(c, { Size = UDim2.fromOffset(230, 44), Position = UDim2.fromOffset(12, 142), Text = m.desc, TextSize = 11, TextColor3 = COL.dim, Font = Enum.Font.Gotham,
			TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true })
	end

	local TIPS = {
		"Headshots deal bonus damage. Aim for the head.",
		"Hold your aim button (ADS) for pinpoint accuracy - but you move slower.",
		"Sprint then tap Slide to slide under fire and break line of sight.",
		"Shotguns are brutal up close; the sniper rewards patience.",
		"Health regenerates after a few seconds out of combat. Break contact!",
		"Level up to unlock new weapons in the LOADOUT tab.",
		"Gun Game: every kill upgrades your weapon. Melee kills downgrade victims!",
		"You can rebind every key and move the mobile buttons in SETTINGS.",
		"Bots fill empty slots - but they give less XP. Real players give full XP.",
		"3 kills = UAV radar. 5 kills = SUPER UAV that shows enemies through walls.",
		"Press G to throw a frag, H to use a stim. Rebind them in SETTINGS.",
		"Open GUNSMITH in LOADOUT to spin your gun 360 degrees and test-fire it.",
	}
	TS.tipLabel = label(dep, { Size = UDim2.fromOffset(780, 30), Position = UDim2.fromOffset(25, 606), Text = "", TextSize = 14, TextColor3 = COL.gold,
		Font = Enum.Font.GothamMedium, TextXAlignment = Enum.TextXAlignment.Left, TextWrapped = true })
	task.spawn(function()
		local i = math.random(1, #TIPS)
		while true do
			TS.tipLabel.Text = "TIP: " .. TIPS[i]
			i = i % #TIPS + 1
			task.wait(8)
		end
	end)

	-- ---------- LOADOUT page (slot cards on the left, big weapon showcase on the right, stat bars bottom-right) ----------
	local lo = makePage("LOADOUT", "LOADOUT", "")
	for _, c in ipairs(lo:GetChildren()) do c:Destroy() end
	local LO = { primary = "VX44", secondary = "MK2", focus = "VX44", rows = {}, cat = "primary" }
	for wid in pairs(WEAPONS) do LO.rows[wid] = true end
	local equipBtn
	local refreshLoadout
	function XT.wlock(def)
		return def.unlock > P.level and not (def.premium and P.owned and P.owned["W_" .. def.id])
	end
	do
		local LIGHT, EDGE, BLUE = RGB(232, 236, 242), RGB(74, 78, 88), RGB(120, 170, 235)
		local ui = new("Frame", { Size = UDim2.fromOffset(830, 650), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 1 }, lo)
		new("UIGradient", { Rotation = 0, Color = ColorSequence.new({ ColorSequenceKeypoint.new(0, RGB(28, 26, 26)), ColorSequenceKeypoint.new(0.45, RGB(13, 13, 15)),
			ColorSequenceKeypoint.new(1, RGB(24, 21, 20)) }) }, ui)
		for k = 1, 13 do -- faint brick-wall courses
			new("Frame", { Size = UDim2.new(1, 0, 0, 1), Position = UDim2.fromOffset(0, k * 50), BackgroundColor3 = RGB(255, 255, 255), BackgroundTransparency = 0.94, BorderSizePixel = 0 }, ui)
		end
		for _, g in ipairs({ { 0, 96, 92 }, { 0, 440, 70 }, { 824, 70, 120 }, { 824, 400, 90 } }) do -- orange edge glows
			local gl = new("Frame", { Size = UDim2.fromOffset(6, g[3]), Position = UDim2.fromOffset(g[1], g[2]), BackgroundColor3 = RGB(255, 150, 40), BackgroundTransparency = 0.25, BorderSizePixel = 0, ZIndex = 2 }, ui)
			new("UIGradient", { Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.5, 0), NumberSequenceKeypoint.new(1, 1) }), Rotation = 90 }, gl)
		end
		label(ui, { Size = UDim2.fromOffset(330, 34), Position = UDim2.fromOffset(24, 14), Text = "CUSTOM LOADOUT 1", TextSize = 26, Font = Enum.Font.GothamBlack,
			TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5 })
		-- brand box (top right, next to the close button)
		local brand = new("Frame", { Size = UDim2.fromOffset(190, 64), Position = UDim2.fromOffset(506, 0), BackgroundColor3 = RGB(6, 6, 8), BorderSizePixel = 0, ZIndex = 4 }, ui)
		label(brand, { Size = UDim2.fromOffset(180, 16), Position = UDim2.fromOffset(8, 6), Text = "BLACKOUT", TextSize = 13, TextColor3 = RGB(255, 150, 40), Font = Enum.Font.GothamBlack,
			TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5 })
		label(brand, { Size = UDim2.fromOffset(182, 36), Position = UDim2.fromOffset(8, 22), Text = "LOADOUTS", TextSize = 30, Font = Enum.Font.GothamBlack,
			TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5 })

		-- ----- helpers for little 3D icons -----
		local function newVp(parent, pos, size, z)
			local vp = new("ViewportFrame", { Size = size, Position = pos, BackgroundTransparency = 1, ZIndex = z, Ambient = RGB(190, 194, 205), LightColor = RGB(255, 255, 255),
				LightDirection = Vector3.new(-0.5, -1, -0.6) }, parent)
			local cam = new("Camera", { FieldOfView = 28 }, vp)
			vp.CurrentCamera = cam
			return vp, cam
		end
		local function frame3d(cam, model, aspect, mode)
			local bcf, size = model:GetBoundingBox()
			local th = math.tan(math.rad(cam.FieldOfView / 2))
			local along = (mode == "side") and size.Z or size.X
			local d = math.max((along / 2 * 1.12) / (th * aspect), (size.Y / 2 * 1.3) / th) + ((mode == "side") and size.X or size.Z) / 2
			if mode == "side" then
				cam.CFrame = CFrame.lookAt(bcf.Position + Vector3.new(-d, 0.05, 0), bcf.Position)
			else
				cam.CFrame = CFrame.lookAt(bcf.Position + Vector3.new(0, 0.05, d), bcf.Position)
			end
			return bcf.Position
		end
		local function whiten(m)
			for _, d in ipairs(m:GetDescendants()) do
				if d:IsA("BasePart") then
					d.Color = RGB(222, 227, 235):Lerp(d.Color, 0.14)
					d.Material = Enum.Material.SmoothPlastic
					d.CastShadow = false
				end
			end
		end
		local function mkPart(m, size, off, col, mat, shape)
			local x = Instance.new("Part")
			x.Anchored, x.CanCollide, x.Size, x.Color = true, false, size, col
			x.Material = mat or Enum.Material.SmoothPlastic
			if shape then x.Shape = shape end
			x.CFrame = CFrame.new(off)
			x.Parent = m
			return x
		end
		local function knifeModel()
			local m = Instance.new("Model")
			mkPart(m, Vector3.new(0.06, 0.3, 1.3), Vector3.new(0, 0.02, -1.0), RGB(226, 231, 239), Enum.Material.Metal)
			mkPart(m, Vector3.new(0.05, 0.18, 0.4), Vector3.new(0, -0.03, -1.85), RGB(226, 231, 239), Enum.Material.Metal)
			mkPart(m, Vector3.new(0.14, 0.44, 0.1), Vector3.new(0, 0, -0.3), RGB(70, 74, 82))
			mkPart(m, Vector3.new(0.16, 0.26, 0.85), Vector3.new(0, -0.03, 0.2), RGB(58, 60, 66))
			mkPart(m, Vector3.new(0.18, 0.3, 0.12), Vector3.new(0, -0.03, 0.68), RGB(200, 205, 214))
			return m
		end
		local function stimModel()
			local m = Instance.new("Model")
			local cyl = Enum.PartType.Cylinder
			-- clear barrel with glowing green serum, orange plunger cap, steel needle, grip rings
			mkPart(m, Vector3.new(1.5, 0.42, 0.42), Vector3.new(0, 0, 0), RGB(200, 232, 255), Enum.Material.Glass, cyl).Transparency = 0.45
			mkPart(m, Vector3.new(1.1, 0.3, 0.3), Vector3.new(0.05, 0, 0), RGB(70, 255, 140), Enum.Material.Neon, cyl)
			mkPart(m, Vector3.new(0.16, 0.56, 0.56), Vector3.new(-0.78, 0, 0), RGB(255, 120, 20), Enum.Material.SmoothPlastic, cyl)
			mkPart(m, Vector3.new(0.5, 0.3, 0.3), Vector3.new(-1.1, 0, 0), RGB(255, 120, 20), Enum.Material.SmoothPlastic, cyl)
			mkPart(m, Vector3.new(0.12, 0.48, 0.48), Vector3.new(0.76, 0, 0), RGB(60, 64, 72), Enum.Material.Metal, cyl)
			mkPart(m, Vector3.new(0.5, 0.06, 0.06), Vector3.new(1.1, 0, 0), RGB(225, 230, 238), Enum.Material.Metal, cyl)
			for i = -1, 1 do
				mkPart(m, Vector3.new(0.06, 0.5, 0.5), Vector3.new(-0.3 + i * 0.22, 0, 0), RGB(40, 44, 50), Enum.Material.SmoothPlastic, cyl)
			end
			mkPart(m, Vector3.new(0.16, 0.7, 0.16), Vector3.new(-0.2, 0, 0), RGB(60, 64, 72), Enum.Material.SmoothPlastic)
			return m
		end
		local function fragModel()
			local m = Instance.new("Model")
			-- olive body with cast ridges, steel top, safety lever and pull-ring
			mkPart(m, Vector3.new(1.0, 1.25, 1.0), Vector3.new(0, 0, 0), RGB(78, 92, 60), Enum.Material.SmoothPlastic, Enum.PartType.Ball)
			for i = -2, 2 do
				mkPart(m, Vector3.new(1.03 - math.abs(i) * 0.12, 0.05, 1.03 - math.abs(i) * 0.12), Vector3.new(0, i * 0.24, 0), RGB(52, 62, 42), Enum.Material.SmoothPlastic, Enum.PartType.Cylinder).CFrame = CFrame.new(0, i * 0.24, 0) * CFrame.Angles(0, 0, math.rad(90))
			end
			mkPart(m, Vector3.new(0.38, 0.24, 0.38), Vector3.new(0, 0.68, 0), RGB(150, 156, 166), Enum.Material.Metal)
			mkPart(m, Vector3.new(0.14, 0.1, 0.7), Vector3.new(0.12, 0.84, 0.22), RGB(205, 210, 220), Enum.Material.Metal)
			mkPart(m, Vector3.new(0.5, 0.06, 0.06), Vector3.new(-0.22, 0.86, 0), RGB(205, 210, 220), Enum.Material.Metal)
			local ring = mkPart(m, Vector3.new(0.05, 0.36, 0.36), Vector3.new(-0.5, 0.86, 0), RGB(215, 220, 228), Enum.Material.Metal, Enum.PartType.Cylinder)
			ring.CFrame = CFrame.new(-0.5, 0.86, 0)
			return m
		end

		XT.stimModel, XT.fragModel = stimModel, fragModel

		-- ----- slot cards -----
		local cards = {}
		local function slotCard(key, x, y, w, h, header, onClick)
			local c = new("TextButton", { Size = UDim2.fromOffset(w, h), Position = UDim2.fromOffset(x, y), BackgroundColor3 = RGB(24, 26, 30), BackgroundTransparency = 0.08,
				AutoButtonColor = false, Text = "", BorderSizePixel = 0, ZIndex = 4 }, ui)
			local st = new("UIStroke", { Color = EDGE, Thickness = 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, c)
			c.MouseEnter:Connect(function() uiHover() end)
			c.Activated:Connect(function() uiClick() if onClick then onClick() end end)
			local hd = label(c, { Size = UDim2.fromOffset(w - 10, 12), Position = UDim2.fromOffset(7, 4), Text = header, TextSize = 9, TextColor3 = RGB(150, 154, 162),
				Font = Enum.Font.GothamMedium, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 6 })
			local card = { btn = c, stroke = st, header = hd, w = w, h = h }
			cards[key] = card
			return card
		end
		local function cardName(card, text, sub)
			card.name = label(card.btn, { Size = UDim2.fromOffset(card.w - 60, 18), Position = UDim2.fromOffset(8, card.h - 22), Text = text, TextSize = 13, Font = Enum.Font.GothamBlack,
				TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 6 })
			card.sub = label(card.btn, { Size = UDim2.fromOffset(52, 14), Position = UDim2.fromOffset(card.w - 58, card.h - 20), Text = sub or "", TextSize = 9, TextColor3 = RGB(190, 194, 202),
				Font = Enum.Font.GothamMedium, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 6 })
		end
		local function gunIcon(card, y0, h0)
			card.vp, card.cam = newVp(card.btn, UDim2.fromOffset(10, y0), UDim2.fromOffset(card.w - 20, h0), 5)
			card.aspect = (card.w - 20) / h0
		end
		local function setGunIcon(card, def)
			if card.iconId == def.id then return end
			card.iconId = def.id
			if card.model then card.model:Destroy() end
			local m = XT.gun(def)
			whiten(m)
			m.Parent = card.vp
			card.model = m
			frame3d(card.cam, m, card.aspect, "side")
		end
		local function dots(card, n)
			card.dots = {}
			for i = 1, 5 do
				local d = new("Frame", { Size = UDim2.fromOffset(6, 6), Position = UDim2.fromOffset(card.w - 66 + (i - 1) * 11, 6), BackgroundColor3 = RGB(255, 255, 255),
					BackgroundTransparency = 0.8, BorderSizePixel = 0, ZIndex = 6 }, card.btn)
				new("UICorner", { CornerRadius = UDim.new(1, 0) }, d)
				card.dots[i] = d
			end
		end
		local function chip(card, slot)
			local b = new("TextButton", { Size = UDim2.fromOffset(88, 17), Position = UDim2.fromOffset(card.w - 94, 19), BackgroundColor3 = RGB(240, 242, 246), AutoButtonColor = false,
				Text = "", BorderSizePixel = 0, ZIndex = 8 }, card.btn)
			local xc = new("Frame", { Size = UDim2.fromOffset(11, 11), Position = UDim2.fromOffset(4, 3), BackgroundColor3 = RGB(14, 14, 16), BorderSizePixel = 0, ZIndex = 9 }, b)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, xc)
			label(xc, { Size = UDim2.fromScale(1, 1), Text = "X", TextSize = 8, ZIndex = 10 })
			label(b, { Size = UDim2.fromOffset(66, 17), Position = UDim2.fromOffset(19, 0), Text = "GUNSMITH", TextSize = 9, TextColor3 = RGB(14, 14, 16), Font = Enum.Font.GothamBlack, ZIndex = 9 })
			b.Activated:Connect(function()
				uiClick()
				LO.focus = LO[slot]
				LO.cat = slot
				if LO.openGS then LO.openGS() end
			end)
		end

		local picker -- weapon list popup (forward)
		local function openPicker(slot)
			if picker then picker.show(slot) end
		end
		local function selectSlot(slot)
			if LO.cat == slot and picker and picker.frame.Visible and picker.slot == slot then
				picker.frame.Visible = false
				return
			end
			LO.cat = slot
			LO.focus = LO[slot]
			openPicker(slot)
			refreshLoadout()
		end

		local cp = slotCard("primary", 24, 60, 336, 126, "", function() selectSlot("primary") end)
		gunIcon(cp, 22, 78)
		cardName(cp, "", "")
		dots(cp)
		chip(cp, "primary")
		local cs = slotCard("secondary", 24, 192, 165, 102, "", function() selectSlot("secondary") end)
		gunIcon(cs, 20, 56)
		cardName(cs, "", "")
		dots(cs)
		local ck = slotCard("melee", 195, 192, 165, 102, "MELEE   KNIFE", nil)
		do
			ck.vp, ck.cam = newVp(ck.btn, UDim2.fromOffset(10, 20), UDim2.fromOffset(145, 56), 5)
			local m = knifeModel()
			whiten(m)
			m.Parent = ck.vp
			frame3d(ck.cam, m, 145 / 56, "side")
			cardName(ck, "KNIFE", "LVL 1")
		end
		local function smallCard(key, x, y, header, name, model)
			local c = slotCard(key, x, y, 108, 98, header, nil)
			if model then
				c.vp, c.cam = newVp(c.btn, UDim2.fromOffset(4, 16), UDim2.fromOffset(100, 62), 5)
				local m = model()
				m.Parent = c.vp
				if key == "tactical" then m:PivotTo(m:GetPivot() * CFrame.Angles(0, 0, math.rad(-24))) end
				frame3d(c.cam, m, 100 / 62, "front")
				c.cam.FieldOfView = 34
				local base = m:GetPivot()
				local t0 = os.clock()
				RunService.RenderStepped:Connect(function()
					if not c.vp.Parent or not c.btn.Visible then return end
					local a = (os.clock() - t0) * 0.9
					if key == "tactical" then
						m:PivotTo(base * CFrame.Angles(a, 0, 0))
					else
						m:PivotTo(base * CFrame.Angles(0, a, 0))
					end
				end)
			end
			cardName(c, name, "")
			return c
		end
		smallCard("tactical", 24, 300, "TACTICAL", "STIM", stimModel)
		smallCard("lethal", 138, 300, "LETHAL", "FRAG", fragModel)
		-- operator / camo / scorestreak
		local cop = smallCard("operator", 24, 404, "OPERATOR", "", nil)
		cop.btn.Activated:Connect(function() showPage("OPERATORS") end)
		cop.helm = new("Frame", { Size = UDim2.fromOffset(40, 40), Position = UDim2.fromOffset(34, 24), BackgroundColor3 = RGB(46, 50, 58), BorderSizePixel = 0, ZIndex = 5 }, cop.btn)
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, cop.helm)
		cop.visor = new("Frame", { Size = UDim2.fromOffset(30, 9), Position = UDim2.fromOffset(5, 16), BackgroundColor3 = RGB(255, 140, 30), BorderSizePixel = 0, ZIndex = 6 }, cop.helm)
		new("UICorner", { CornerRadius = UDim.new(0, 4) }, cop.visor)
		local ccm = smallCard("camo", 138, 404, "CAMO", "", nil)
		ccm.btn.Activated:Connect(function() showPage("STORE") end)
		ccm.sw = new("Frame", { Size = UDim2.fromOffset(72, 40), Position = UDim2.fromOffset(18, 26), BackgroundColor3 = RGB(60, 64, 74), BorderSizePixel = 0, ZIndex = 5 }, ccm.btn)
		new("UICorner", { CornerRadius = UDim.new(0, 5) }, ccm.sw)
		ccm.s1 = new("Frame", { Size = UDim2.fromOffset(10, 40), Position = UDim2.fromOffset(14, 0), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 6 }, ccm.sw)
		ccm.s2 = new("Frame", { Size = UDim2.fromOffset(10, 40), Position = UDim2.fromOffset(32, 0), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 6 }, ccm.sw)
		ccm.s3 = new("Frame", { Size = UDim2.fromOffset(10, 40), Position = UDim2.fromOffset(50, 0), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, ZIndex = 6 }, ccm.sw)
		local cst = smallCard("streak", 252, 404, "SCORESTREAK", "AIRSTRIKE", nil)
		do
			local strokes = {}
			local ring = new("Frame", { Size = UDim2.fromOffset(44, 44), Position = UDim2.fromOffset(32, 24), BackgroundTransparency = 1, ZIndex = 5 }, cst.btn)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, ring)
			table.insert(strokes, new("UIStroke", { Color = RGB(120, 255, 170), Thickness = 2 }, ring))
			local r2 = new("Frame", { Size = UDim2.fromOffset(22, 22), Position = UDim2.fromOffset(11, 11), BackgroundTransparency = 1, ZIndex = 5 }, ring)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, r2)
			table.insert(strokes, new("UIStroke", { Color = RGB(120, 255, 170), Thickness = 1, Transparency = 0.4 }, r2))
			local dot = new("Frame", { Size = UDim2.fromOffset(6, 6), Position = UDim2.fromOffset(28, 10), BackgroundColor3 = RGB(120, 255, 170), BorderSizePixel = 0, ZIndex = 6 }, ring)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, dot)
			local hint = label(cst.btn, { Size = UDim2.fromOffset(100, 12), Position = UDim2.fromOffset(4, 70), Text = "TAP TO SWAP", TextSize = 8, TextColor3 = RGB(150, 154, 162), ZIndex = 6 })
			local function chosen() return (P.equip and P.equip.streak == "BOMBER") and "BOMBER" or "AIRSTRIKE" end
			local function paint()
				local bomber = chosen() == "BOMBER"
				local col = bomber and RGB(255, 90, 70) or RGB(120, 255, 170)
				cst.name.Text = bomber and "BOMBER" or "AIRSTRIKE"
				for _, st in ipairs(strokes) do st.Color = col end
				dot.BackgroundColor3 = col
				dot.Position = bomber and UDim2.fromOffset(19, 19) or UDim2.fromOffset(28, 10)
				dot.Size = bomber and UDim2.fromOffset(6, 6) or UDim2.fromOffset(6, 6)
			end
			cst.btn.Activated:Connect(function()
				P.equip = P.equip or {}
				local nxt = (chosen() == "BOMBER") and "AIRSTRIKE" or "BOMBER"
				P.equip.streak = nxt
				Net.EquipItem:FireServer("streak", nxt)
				paint()
			end)
			task.spawn(function()
				while cst.btn.Parent do
					paint()
					task.wait(0.6)
				end
			end)
			paint()
		end
		local cw = slotCard("wild", 24, 508, 222, 88, "WILDCARD", nil)
		label(cw.btn, { Size = UDim2.fromOffset(212, 30), Position = UDim2.fromOffset(5, 26), Text = "MORE COMING SOON!", TextSize = 18, Font = Enum.Font.GothamBlack, TextColor3 = RGB(255, 150, 40), ZIndex = 6 })
		label(cw.btn, { Size = UDim2.fromOffset(212, 16), Position = UDim2.fromOffset(5, 58), Text = "NEW WEAPONS EVERY SEASON", TextSize = 10, TextColor3 = RGB(150, 154, 162), ZIndex = 6 })

		-- ----- right side: weapon showcase -----
		local gvp, gcam = newVp(ui, UDim2.fromOffset(350, 204), UDim2.fromOffset(480, 330), 2)
		gvp.Ambient = RGB(170, 168, 175)
		gvp.LightColor = RGB(255, 236, 214)
		gcam.FieldOfView = 24
		local wName = label(ui, { Size = UDim2.fromOffset(440, 40), Position = UDim2.fromOffset(372, 78), Text = "", TextSize = 36, Font = Enum.Font.GothamBlack,
			TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 5 })
		new("Frame", { Size = UDim2.fromOffset(86, 2), Position = UDim2.fromOffset(374, 122), BackgroundColor3 = RGB(190, 194, 202), BorderSizePixel = 0, ZIndex = 5 }, ui)
		local wLevel = label(ui, { Size = UDim2.fromOffset(440, 14), Position = UDim2.fromOffset(372, 132), Text = "", TextSize = 10, Font = Enum.Font.GothamMedium,
			TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5 })
		local wDesc = label(ui, { Size = UDim2.fromOffset(440, 34), Position = UDim2.fromOffset(372, 150), Text = "", TextSize = 13, Font = Enum.Font.Gotham, TextColor3 = RGB(210, 214, 222),
			TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true, ZIndex = 5 })
		local wNote = label(ui, { Size = UDim2.fromOffset(440, 14), Position = UDim2.fromOffset(372, 186), Text = "", TextSize = 9, TextColor3 = RGB(255, 140, 40), Font = Enum.Font.GothamBold,
			TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5 })
		local GV = { id = nil, key = "", gun = nil, c = Vector3.new(), base = CFrame.new() }
		function XT.gvShow(def, key)
			if GV.key == key then return end
			GV.key = key
			if GV.gun then GV.gun:Destroy() end
			local g = XT.gun(def)
			g.Parent = gvp
			GV.gun, GV.base = g, g:GetPivot()
			GV.c = frame3d(gcam, g, 480 / 330, "side")
		end
		local wasVis = false
		RunService.RenderStepped:Connect(function()
			if lo.Visible and not wasVis then refreshLoadout() end -- pick up operator / camo / profile changes when the page opens
			wasVis = lo.Visible
			if not lo.Visible or not GV.gun then return end
			local t = os.clock()
			local rot = CFrame.new(GV.c) * CFrame.Angles(math.sin(t * 0.7) * 0.05, math.sin(t * 0.55) * 0.42, 0) * CFrame.new(-GV.c)
			GV.gun:PivotTo(rot * GV.base)
		end)
		-- arrows: previous / next weapon in this slot
		local function cycle(dir)
			local list = {}
			for _, id in ipairs(LO.cat == "primary" and PRIMARY_ORDER or SECONDARY_ORDER) do table.insert(list, id) end
			for _, id in ipairs(LO.cat == "primary" and PREMIUM_PRIMARY or PREMIUM_SECONDARY) do table.insert(list, id) end
			local idx = 1
			for i, id in ipairs(list) do if id == LO.focus then idx = i end end
			LO.focus = list[(idx - 1 + dir) % #list + 1]
			refreshLoadout()
		end
		for _, a in ipairs({ { "<", 358, -1 }, { ">", 788, 1 } }) do
			local ab = new("TextButton", { Size = UDim2.fromOffset(36, 70), Position = UDim2.fromOffset(a[2], 330), BackgroundTransparency = 1, Text = a[1], TextColor3 = RGB(230, 232, 238),
				TextSize = 44, Font = Enum.Font.Gotham, AutoButtonColor = false, ZIndex = 7 }, ui)
			ab.MouseEnter:Connect(function() ab.TextColor3 = RGB(255, 150, 40) end)
			ab.MouseLeave:Connect(function() ab.TextColor3 = RGB(230, 232, 238) end)
			ab.Activated:Connect(function() uiClick() cycle(a[3]) end)
		end
		-- equip + state
		equipBtn = button(ui, "EQUIP", UDim2.fromOffset(160, 38), UDim2.fromOffset(372, 548), RGB(255, 150, 40), function()
			local def = WEAPONS[LO.focus]
			if not def or XT.wlock(def) then return end
			if def.slot == "primary" then LO.primary = def.id else LO.secondary = def.id end
			Net.Loadout:FireServer(LO.primary, LO.secondary)
			P.loadout.primary, P.loadout.secondary = LO.primary, LO.secondary
			refreshLoadout()
		end, 17)
		equipBtn.ZIndex = 7
		equipBtn.TextColor3 = RGB(16, 12, 8)
		equipBtn.Font = Enum.Font.GothamBlack
		local wState = label(ui, { Size = UDim2.fromOffset(300, 16), Position = UDim2.fromOffset(372, 590), Text = "", TextSize = 11, TextColor3 = COL.gold, Font = Enum.Font.GothamBold,
			TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 6 })
		-- attachments counter + stats panel
		local attDots = {}
		for i = 1, 5 do
			local d = new("Frame", { Size = UDim2.fromOffset(8, 8), Position = UDim2.fromOffset(712 + (i - 1) * 13, 536), BackgroundColor3 = RGB(255, 255, 255), BackgroundTransparency = 0.8,
				BorderSizePixel = 0, ZIndex = 6 }, ui)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, d)
			attDots[i] = d
		end
		local attTxt = label(ui, { Size = UDim2.fromOffset(40, 14), Position = UDim2.fromOffset(780, 533), Text = "0 / 5", TextSize = 12, Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 6 })
		local sp = new("Frame", { Size = UDim2.fromOffset(264, 76), Position = UDim2.fromOffset(556, 552), BackgroundColor3 = RGB(22, 24, 28), BackgroundTransparency = 0.05, BorderSizePixel = 0, ZIndex = 5 }, ui)
		new("UIStroke", { Color = EDGE, Thickness = 1 }, sp)
		local sBars = {}
		for i, nm in ipairs({ "FIREPOWER", "ACCURACY", "MOBILITY", "HANDLING" }) do
			local y = 6 + (i - 1) * 16
			label(sp, { Size = UDim2.fromOffset(64, 12), Position = UDim2.fromOffset(8, y), Text = nm, TextSize = 8, TextColor3 = RGB(200, 204, 212), Font = Enum.Font.GothamMedium,
				TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 6 })
			local bg = new("Frame", { Size = UDim2.fromOffset(112, 8), Position = UDim2.fromOffset(74, y + 2), BackgroundColor3 = RGB(46, 48, 54), BorderSizePixel = 0, ZIndex = 6 }, sp)
			sBars[i] = new("Frame", { Size = UDim2.fromScale(0.5, 1), BackgroundColor3 = RGB(228, 232, 238), BorderSizePixel = 0, ZIndex = 7 }, bg)
		end
		local function statBox(x, y, title)
			new("Frame", { Size = UDim2.fromOffset(62, 32), Position = UDim2.fromOffset(x, y), BackgroundColor3 = RGB(34, 36, 42), BorderSizePixel = 0, ZIndex = 6 }, sp)
			label(sp, { Size = UDim2.fromOffset(56, 10), Position = UDim2.fromOffset(x + 4, y + 2), Text = title, TextSize = 7, TextColor3 = RGB(190, 194, 202), TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 7 })
			return label(sp, { Size = UDim2.fromOffset(56, 18), Position = UDim2.fromOffset(x + 4, y + 11), Text = "", TextSize = 16, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 7 })
		end
		local vMags = statBox(194, 6, "MAGS")
		local vMagSize = statBox(194, 40, "MAG SIZE")
		label(ui, { Size = UDim2.fromOffset(60, 14), Position = UDim2.fromOffset(590, 630), Text = "STATS", TextSize = 12, Font = Enum.Font.GothamBold, ZIndex = 6 })
		new("Frame", { Size = UDim2.fromOffset(94, 2), Position = UDim2.fromOffset(573, 645), BackgroundColor3 = RGB(230, 232, 238), BorderSizePixel = 0, ZIndex = 6 }, ui)
		local gsTab = new("TextButton", { Size = UDim2.fromOffset(110, 18), Position = UDim2.fromOffset(700, 628), BackgroundTransparency = 1, Text = "GUNSMITH", TextColor3 = RGB(170, 174, 182),
			TextSize = 12, Font = Enum.Font.GothamBold, AutoButtonColor = false, ZIndex = 6 }, ui)
		gsTab.MouseEnter:Connect(function() gsTab.TextColor3 = RGB(255, 150, 40) end)
		gsTab.MouseLeave:Connect(function() gsTab.TextColor3 = RGB(170, 174, 182) end)
		gsTab.Activated:Connect(function() uiClick() if LO.openGS then LO.openGS() end end)

		-- ----- weapon picker popup (opens when you click the primary / sidearm card) -----
		do
			local pf = new("Frame", { Size = UDim2.fromOffset(452, 340), Position = UDim2.fromOffset(372, 200), BackgroundColor3 = RGB(12, 13, 16), BackgroundTransparency = 0.03,
				BorderSizePixel = 0, Visible = false, ZIndex = 20 }, ui)
			new("UIStroke", { Color = BLUE, Thickness = 2 }, pf)
			local ptitle = label(pf, { Size = UDim2.fromOffset(330, 24), Position = UDim2.fromOffset(12, 6), Text = "", TextSize = 15, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 21 })
			local pclose = new("TextButton", { Size = UDim2.fromOffset(28, 24), Position = UDim2.fromOffset(414, 6), BackgroundTransparency = 1, Text = "X", TextColor3 = RGB(230, 232, 238),
				TextSize = 16, Font = Enum.Font.GothamBold, ZIndex = 21 }, pf)
			pclose.Activated:Connect(function() pf.Visible = false end)
			local sc = new("ScrollingFrame", { Size = UDim2.fromOffset(440, 296), Position = UDim2.fromOffset(6, 36), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 4,
				CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, ZIndex = 21 }, pf)
			new("UIGridLayout", { CellSize = UDim2.fromOffset(140, 92), CellPadding = UDim2.fromOffset(6, 6), SortOrder = Enum.SortOrder.LayoutOrder }, sc)
			local tiles = {}
			picker = { frame = pf, slot = nil }
			local function tileTags()
				for id, t in pairs(tiles) do
					local def = WEAPONS[id]
					local locked = XT.wlock(def)
					local eq = (LO.primary == id) or (LO.secondary == id)
					t.tag.Text = locked and (def.premium and "STORE" or ("LVL " .. def.unlock)) or (eq and "EQUIPPED" or "")
					t.tag.TextColor3 = locked and RGB(255, 90, 90) or RGB(120, 255, 170)
					t.name.TextColor3 = locked and RGB(150, 154, 162) or RGB(236, 238, 244)
					t.stroke.Color = (LO.focus == id) and BLUE or EDGE
					t.stroke.Thickness = (LO.focus == id) and 2 or 1
				end
			end
			picker.tags = tileTags
			function picker.show(slot)
				picker.slot = slot
				ptitle.Text = slot == "primary" and "SELECT PRIMARY WEAPON" or "SELECT SIDEARM"
				for _, c in ipairs(sc:GetChildren()) do if c:IsA("GuiObject") then c:Destroy() end end
				table.clear(tiles)
				local list = {}
				for _, id in ipairs(slot == "primary" and PRIMARY_ORDER or SECONDARY_ORDER) do table.insert(list, id) end
				for _, id in ipairs(slot == "primary" and PREMIUM_PRIMARY or PREMIUM_SECONDARY) do table.insert(list, id) end
				for n, id in ipairs(list) do
					local def = WEAPONS[id]
					local t = new("TextButton", { BackgroundColor3 = RGB(24, 26, 30), AutoButtonColor = false, Text = "", BorderSizePixel = 0, LayoutOrder = n, ZIndex = 22 }, sc)
					local st = new("UIStroke", { Color = EDGE, Thickness = 1 }, t)
					t.Activated:Connect(function()
						uiClick()
						LO.focus = id
						pf.Visible = false
						refreshLoadout()
					end)
					local tv, tc = newVp(t, UDim2.fromOffset(6, 14), UDim2.fromOffset(128, 50), 23)
					local m = XT.gun(def)
					whiten(m)
					m.Parent = tv
					frame3d(tc, m, 128 / 50, "side")
					label(t, { Size = UDim2.fromOffset(130, 12), Position = UDim2.fromOffset(6, 2), Text = def.class, TextSize = 8, TextColor3 = RGB(150, 154, 162), TextXAlignment = Enum.TextXAlignment.Left,
						TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 23 })
					local nm = label(t, { Size = UDim2.fromOffset(92, 18), Position = UDim2.fromOffset(6, 70), Text = def.name, TextSize = 11, Font = Enum.Font.GothamBlack,
						TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 23 })
					local tg = label(t, { Size = UDim2.fromOffset(60, 14), Position = UDim2.fromOffset(76, 72), Text = "", TextSize = 8, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 23 })
					tiles[id] = { name = nm, tag = tg, stroke = st }
				end
				local soon = new("Frame", { BackgroundColor3 = RGB(18, 19, 23), BorderSizePixel = 0, LayoutOrder = 9999, ZIndex = 22 }, sc)
				new("UIStroke", { Color = EDGE, Thickness = 1, Transparency = 0.4 }, soon)
				label(soon, { Size = UDim2.fromScale(1, 0.4), Position = UDim2.fromScale(0, 0.22), Text = "MORE COMING SOON!", TextSize = 13, Font = Enum.Font.GothamBlack, TextColor3 = RGB(255, 150, 40), ZIndex = 23 })
				label(soon, { Size = UDim2.fromScale(1, 0.2), Position = UDim2.fromScale(0, 0.62), Text = "NEW WEAPONS EVERY SEASON", TextSize = 8, TextColor3 = RGB(150, 154, 162), ZIndex = 23 })
				pf.Visible = true
				tileTags()
			end
		end

		-- ----- refresh everything from the current state -----
		function refreshLoadout()
			local pDef, sDef = WEAPONS[LO.primary] or WEAPONS.VX44, WEAPONS[LO.secondary] or WEAPONS.MK2
			local function attCount(id)
				local att = P.loadout and P.loadout.att and P.loadout.att[id]
				return att and #cleanAttach(att, P.level, P.owned or {}) or 0
			end
			for _, sl in ipairs({ { cp, pDef }, { cs, sDef } }) do
				local card, d = sl[1], sl[2]
				card.header.Text = d.class .. "   " .. d.name
				card.name.Text = d.name
				card.sub.Text = "LVL " .. d.unlock
				if d.premium then card.sub.Text = "PREMIUM" end
				setGunIcon(card, d)
				local n = attCount(d.id)
				for i, dt in ipairs(card.dots) do dt.BackgroundTransparency = (i <= n) and 0 or 0.8 end
			end
			cp.stroke.Color = (LO.cat == "primary") and BLUE or EDGE
			cp.stroke.Thickness = (LO.cat == "primary") and 2 or 1
			cs.stroke.Color = (LO.cat == "secondary") and BLUE or EDGE
			cs.stroke.Thickness = (LO.cat == "secondary") and 2 or 1
			local opId = P.equip and P.equip.operator
			local op = (opId and OPERATORS[opId]) or OPERATORS.OP_STANDARD
			if op then
				cop.name.Text = op.name or "OPERATOR"
				cop.visor.BackgroundColor3 = op.accent or RGB(255, 140, 30)
			end
			local cm = P.equip and P.equip.camo and CAMOS[P.equip.camo]
			if cm and cm.body then
				ccm.name.Text = cm.name
				ccm.sw.BackgroundColor3 = cm.body
				ccm.s1.BackgroundColor3, ccm.s2.BackgroundColor3, ccm.s3.BackgroundColor3 = cm.c1 or cm.accent, cm.c2 or cm.accent, cm.accent
				ccm.s1.BackgroundTransparency, ccm.s2.BackgroundTransparency, ccm.s3.BackgroundTransparency = 0, 0, 0
			else
				ccm.name.Text = "STANDARD"
				ccm.sw.BackgroundColor3 = RGB(60, 64, 74)
				ccm.s1.BackgroundTransparency, ccm.s2.BackgroundTransparency, ccm.s3.BackgroundTransparency = 1, 1, 1
			end
			-- showcase
			local def = WEAPONS[LO.focus] or pDef
			local list = {}
			do
				local att = P.loadout and P.loadout.att and P.loadout.att[def.id]
				list = att and cleanAttach(att, P.level, P.owned or {}) or {}
			end
			local camoId = P.equip and P.equip.camo or "C_NONE"
			local eDef = effDef(def, list)
			XT.gvShow(camoDef(eDef, camoId), def.id .. ":" .. table.concat(list, ",") .. ":" .. camoId)
			wName.Text = def.name
			wLevel.Text = def.premium and "PREMIUM WEAPON" or ("WEAPON LEVEL " .. def.unlock)
			wDesc.Text = def.desc
			local locked = XT.wlock(def)
			if def.premium then
				wNote.Text = locked and "AVAILABLE IN THE STORE" or "PERMANENTLY UNLOCKED BY PURCHASING IN THE STORE"
			else
				wNote.Text = locked and ("UNLOCKS AT LEVEL " .. def.unlock) or "UNLOCKED"
			end
			local fin = {
				math.clamp((def.stats[1] or 0.5) * (eDef.dmg / def.dmg), 0.05, 1),
				eDef.scope and math.clamp(0.9 - eDef.kickP / 20, 0.08, 1) or math.clamp(1.08 - eDef.hip / 8.5 - eDef.kickP / 13, 0.08, 1),
				math.clamp((def.stats[4] or 0.5) * (eDef.speed / def.speed) * math.sqrt(def.adsTime / eDef.adsTime), 0.05, 1),
				math.clamp(1.15 - eDef.adsTime * 1.3 - eDef.reload / 10, 0.08, 1),
			}
			for i, bar in ipairs(sBars) do tween(bar, 0.2, { Size = UDim2.fromScale(fin[i], 1) }) end
			vMags.Text = tostring(math.max(1, math.floor(eDef.reserve / eDef.mag + 0.5)))
			vMagSize.Text = tostring(eDef.mag)
			for i, dt in ipairs(attDots) do dt.BackgroundTransparency = (i <= #list) and 0 or 0.8 end
			attTxt.Text = #list .. " / 5"
			local isEq = (def.slot == "primary" and LO.primary == def.id) or (def.slot == "secondary" and LO.secondary == def.id)
			equipBtn.Text = locked and "LOCKED" or (isEq and "EQUIPPED" or "EQUIP")
			equipBtn.BackgroundColor3 = (locked or isEq) and RGB(70, 72, 80) or RGB(255, 150, 40)
			wState.Text = locked and (def.premium and "PREMIUM - buy it in the STORE" or ("LOCKED - reach level " .. def.unlock)) or ""
			if picker then picker.tags() end
		end
	end

	-- ---------- GUNSMITH (max 5 attachments per gun) ----------
	do
		local GS = { cat = ATTACH_CATS[1], open = false }
		local panel = new("Frame", { Size = UDim2.fromOffset(830, 650), Position = UDim2.fromOffset(0, 0), BackgroundColor3 = COL.bg, BorderSizePixel = 0,
			Visible = false, ZIndex = 6 }, lo)
		corner(panel, 10)
		stroke(panel, COL.line, 1)
		local title = label(panel, { Size = UDim2.fromOffset(500, 28), Position = UDim2.fromOffset(16, 10), Text = "GUNSMITH", TextSize = 22, TextColor3 = COL.accent,
			TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 7 })
		local slotsTxt = label(panel, { Size = UDim2.fromOffset(260, 24), Position = UDim2.fromOffset(554, 12), Text = "", TextSize = 16, TextColor3 = COL.accent2,
			TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 7 })
		local statTxt = label(panel, { Size = UDim2.fromOffset(560, 40), Position = UDim2.fromOffset(16, 46), Text = "", TextSize = 12, TextColor3 = COL.dim,
			Font = Enum.Font.Gotham, TextXAlignment = Enum.TextXAlignment.Left, TextWrapped = true, ZIndex = 7 })
		statTxt.Visible = false
		-- stat bar: grey = stock gun, green = better than stock, red = worse. d is -1..1 (positive = better)
		local function deltaBar(parent, x, y, w, h, z)
			local bg = new("Frame", { Size = UDim2.fromOffset(w, h), Position = UDim2.fromOffset(x, y), BackgroundColor3 = RGB(20, 26, 46), BorderSizePixel = 0, ZIndex = z }, parent)
			new("Frame", { Size = UDim2.fromScale(0.5, 1), BackgroundColor3 = RGB(120, 130, 150), BorderSizePixel = 0, ZIndex = z + 1 }, bg)
			local seg = new("Frame", { Size = UDim2.fromScale(0, 1), Position = UDim2.fromScale(0.5, 0), BackgroundColor3 = COL.green, BorderSizePixel = 0, ZIndex = z + 2 }, bg)
			return function(d)
				d = math.clamp(d, -1, 1)
				local amt = math.abs(d) * 0.5
				seg.Size = UDim2.fromScale(amt, 1)
				seg.Position = UDim2.fromScale(d >= 0 and 0.5 or 0.5 - amt, 0)
				seg.BackgroundColor3 = d >= 0 and COL.green or COL.red
			end
		end
		label(panel, { Size = UDim2.fromOffset(300, 14), Position = UDim2.fromOffset(350, 18), Text = "GREY = STOCK GUN     GREEN = BETTER     RED = WORSE", TextSize = 10,
			TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 7 })
		local GB = {}
		for i, g in ipairs({ { "RECOIL", "kickP", -1 }, { "HIP SPREAD", "hip", -1 }, { "RANGE", "rFar", 1 }, { "MAGAZINE", "mag", 1 }, { "RELOAD", "reload", -1 }, { "AIM SPEED", "adsTime", -1 } }) do
			local x = 16 + (i - 1) * 96
			label(panel, { Size = UDim2.fromOffset(90, 12), Position = UDim2.fromOffset(x, 42), Text = g[1], TextSize = 10, TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 7 })
			local set = deltaBar(panel, x, 57, 88, 10, 7)
			local val = label(panel, { Size = UDim2.fromOffset(90, 14), Position = UDim2.fromOffset(x, 69), Text = "", TextSize = 11, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 7 })
			GB[i] = { set = set, val = val, def = g }
		end
		local catList = new("Frame", { Size = UDim2.fromOffset(170, 400), Position = UDim2.fromOffset(14, 92), BackgroundTransparency = 1, ZIndex = 7 }, panel)
		new("UIListLayout", { Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder }, catList)
		local optList = new("ScrollingFrame", { Size = UDim2.fromOffset(232, 540), Position = UDim2.fromOffset(590, 92), BackgroundTransparency = 1, BorderSizePixel = 0,
			ScrollBarThickness = 4, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, ZIndex = 7 }, panel)
		new("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }, optList)
		local catBtns, catSub = {}, {}
		local function curList()
			P.loadout.att = P.loadout.att or {}
			return cleanAttach(P.loadout.att[LO.focus], P.level, P.owned or {}, LO.focus)
		end
		local function has(list, id)
			for _, v in ipairs(list) do if v == id then return true end end
			return false
		end
		-- readable list of what an attachment changes, e.g. "RECOIL -22%" (green = good, red = trade-off)
		local FX = {
			kickP = { "RECOIL", -1 }, kickY = { "RECOIL", -1, true }, rNear = { "RANGE", 1 }, rFar = { "RANGE", 1, true }, adsTime = { "AIM TIME", -1 },
			hip = { "HIP SPREAD", -1 }, moveSpread = { "MOVE SPREAD", -1 }, mag = { "MAGAZINE", 1 }, reload = { "RELOAD TIME", -1 }, reloadEmpty = { "", -1, true },
			dmg = { "DAMAGE", 1 }, dmgFar = { "LONG DAMAGE", 1 }, ads = { "AIM SPREAD", -1 },
		}
		local function fxList(a)
			local seen, out = {}, {}
			local function add(label, pct, good)
				table.insert(out, { label = label, pct = pct, good = good })
			end
			for k, mult in pairs(a.mul or {}) do
				if k == "zoom" then
					if not seen.zoom then
						seen.zoom = true
						add("ZOOM", math.floor((1 / mult - 1) * 100 + 0.5), mult < 1)
					end
				else
					local f = FX[k]
					if f and not f[3] and not seen[f[1]] then
						seen[f[1]] = true
						local pct = math.floor((mult - 1) * 100 + 0.5)
						if pct ~= 0 then add(f[1], pct, (pct > 0) == (f[2] > 0)) end
					end
				end
			end
			for k, v in pairs(a.add or {}) do
				if k == "speed" then add("MOVE SPEED", math.floor(v * 100 + 0.5), v > 0)
				elseif k == "headMult" then add("HEADSHOT", math.floor(v * 100 + 0.5), v > 0) end
			end
			table.sort(out, function(x, y) return x.label < y.label end)
			return out
		end
		local render
		local function commit(list)
			P.loadout.att = P.loadout.att or {}
			P.loadout.att[LO.focus] = list
			Net.Attach:FireServer(LO.focus, list)
			playSound(SND.click, 1.2, 0.4 * S.g.sfx)
			render()
		end
		function render()
			local def = WEAPONS[LO.focus]
			if not def then return end
			local list = curList()
			if GS.rebuild then GS.rebuild(def, list) end
			title.Text = "GUNSMITH   " .. def.name
			slotsTxt.Text = string.format("ATTACHMENTS  %d / %d", #list, ATTACH_MAX)
			local e = effDef(def, list)
			for _, gb in ipairs(GB) do
				local key = gb.def[2]
				local ratio = (def[key] and def[key] ~= 0) and (e[key] / def[key]) or 1
				local score = (ratio - 1) * gb.def[3] -- positive = better for the player
				local col = score > 0.001 and COL.green or (score < -0.001 and COL.red or COL.dim)
				gb.set(score * 1.5)
				local pct = math.floor(score * 100 + 0.5)
				gb.val.Text = (pct == 0) and "STOCK" or ((pct > 0 and "+" or "") .. pct .. "% BETTER")
				if pct < 0 then gb.val.Text = pct .. "% WORSE" end
				gb.val.TextColor3 = col
			end
			for cat, b in pairs(catBtns) do
				local equippedName
				for _, id in ipairs(list) do if ATTACH_BY_ID[id].cat == cat then equippedName = ATTACH_BY_ID[id].name end end
				b.BackgroundColor3 = (GS.cat == cat) and COL.line or COL.panel2
				catSub[cat].Text = equippedName or "EMPTY"
				catSub[cat].TextColor3 = equippedName and COL.green or COL.dim
			end
			for _, c in ipairs(optList:GetChildren()) do if c:IsA("GuiObject") then c:Destroy() end end
			local n = 0
			for _, a in ipairs(ATTACHMENTS) do
				if a.cat == GS.cat and (not a.only or a.only == LO.focus) then
					n += 1
					local locked = a.unlock > P.level or (a.premium and not (P.owned and P.owned["A_" .. a.id]))
					local on = has(list, a.id)
					local card = button(optList, "", UDim2.fromOffset(222, 130), UDim2.new(), on and COL.line or COL.panel2, function()
						if locked then return end
						local nl = {}
						for _, id in ipairs(list) do if id ~= a.id then table.insert(nl, id) end end
						if not on then
							-- one per category: replace the old one
							for i = #nl, 1, -1 do if ATTACH_BY_ID[nl[i]].cat == a.cat then table.remove(nl, i) end end
							if #nl >= ATTACH_MAX then return end
							table.insert(nl, a.id)
						end
						commit(nl)
					end)
					card.LayoutOrder = n
					card.ZIndex = 8
					label(card, { Size = UDim2.fromOffset(120, 22), Position = UDim2.fromOffset(10, 5), Text = a.name, TextSize = 16, TextColor3 = locked and COL.dim or COL.text,
						TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 9 })
					label(card, { Size = UDim2.fromOffset(204, 30), Position = UDim2.fromOffset(10, 28), Text = a.desc, TextSize = 11, TextColor3 = COL.dim, Font = Enum.Font.Gotham,
						TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true, ZIndex = 9 })
					for ri, fx in ipairs(fxList(a)) do
						if ri > 4 then break end
						local ry = 62 + (ri - 1) * 16
						local col = fx.good and COL.green or COL.red
						label(card, { Size = UDim2.fromOffset(80, 12), Position = UDim2.fromOffset(10, ry), Text = fx.label, TextSize = 10, TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 9 })
						deltaBar(card, 94, ry + 2, 78, 8, 9)(((fx.good and 1) or -1) * math.clamp(math.abs(fx.pct) / 40, 0.1, 1))
						label(card, { Size = UDim2.fromOffset(44, 12), Position = UDim2.fromOffset(174, ry), Text = (fx.pct >= 0 and "+" or "") .. fx.pct .. "%", TextSize = 10, TextColor3 = col,
							TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 9 })
					end
					label(card, { Size = UDim2.fromOffset(86, 22), Position = UDim2.fromOffset(130, 5), Text = locked and (a.premium and "STORE" or ("LVL " .. a.unlock)) or (on and "EQUIPPED" or "EQUIP"),
						TextSize = 13, TextColor3 = locked and COL.red or (on and COL.green or COL.accent), TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 9 })
				end
			end
		end
		for i, cat in ipairs(ATTACH_CATS) do
			local b = button(catList, "", UDim2.fromOffset(170, 46), UDim2.new(), COL.panel2, function()
				GS.cat = cat
				render()
			end)
			b.LayoutOrder = i
			b.ZIndex = 8
			label(b, { Size = UDim2.fromOffset(150, 18), Position = UDim2.fromOffset(10, 4), Text = cat, TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 9 })
			catSub[cat] = label(b, { Size = UDim2.fromOffset(150, 16), Position = UDim2.fromOffset(10, 25), Text = "EMPTY", TextSize = 11, TextColor3 = COL.dim,
				TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 9 })
			catBtns[cat] = b
		end
		-- 360 degree preview + test fire
		do
			local vp = new("ViewportFrame", { Size = UDim2.fromOffset(392, 440), Position = UDim2.fromOffset(190, 92), BackgroundColor3 = RGB(58, 66, 80),
				BorderSizePixel = 0, ZIndex = 7, Ambient = RGB(215, 215, 225), LightColor = RGB(255, 255, 255), LightDirection = Vector3.new(-0.6, -1, -0.5) }, panel)
			corner(vp, 8)
			stroke(vp, COL.line, 1)
			local cam = new("Camera", {}, vp)
			vp.CurrentCamera = cam
			label(panel, { Size = UDim2.fromOffset(392, 16), Position = UDim2.fromOffset(190, 534), Text = "DRAG TO ROTATE   BLUE PARTS ON THE GUN ARE YOUR ATTACHMENTS", TextSize = 10, TextColor3 = COL.dim, ZIndex = 8 })
			local V = { zoomOn = true, curC = nil, curD = 5, yaw = 0.6, pitch = 0.2, dist = 5, model = nil, kick = 0, drag = false, last = nil, center = Vector3.zero, idle = 0 }
			local function target() return WEAPONS[LO.focus] end
			function GS.rebuild(def, list)
				if V.model then V.model:Destroy() end
				local e = effDef(def, list)
				local m = buildGunModel(e, true)
				for _, d in ipairs(m:GetDescendants()) do
					if d:IsA("BasePart") then
						d.Color = d.Color:Lerp(RGB(255, 255, 255), 0.28)
						if d.Material == Enum.Material.Metal then d.Material = Enum.Material.SmoothPlastic end
						if string.sub(d.Name, 1, 3) == "Att" and d.Material ~= Enum.Material.Neon then
							d.Color = RGB(70, 190, 255) -- attachments glow blue so you can see exactly what you added
							d.Material = Enum.Material.SmoothPlastic
						end
					end
				end
				m.Parent = vp
				local cf, size = m:GetBoundingBox()
				V.center = cf.Position
				V.size = size.Magnitude
				V.dist = math.max(3.0, size.Magnitude * 1.05 + 0.6)
				V.model = m
				V.muzzle = m:FindFirstChild("AttMuzzle") or m:FindFirstChild("Muzzle")
				-- bounds of each equipped attachment category, so the camera can fly in on it
				V.cb = {}
				for _, d in ipairs(m:GetChildren()) do
					if d:IsA("BasePart") then
						local nm = d.Name
						local c2 = (string.sub(nm, 1, 8) == "AttOptic" or nm == "AttLens") and "OPTIC" or string.sub(nm, 1, 9) == "AttMuzzle" and "MUZZLE" or string.sub(nm, 1, 9) == "AttBarrel" and "BARREL"
							or string.sub(nm, 1, 7) == "AttGrip" and "UNDERBARREL" or string.sub(nm, 1, 6) == "AttMag" and "MAGAZINE" or string.sub(nm, 1, 8) == "AttStock" and "STOCK"
							or string.sub(nm, 1, 8) == "AttLaser" and "LASER" or string.sub(nm, 1, 7) == "AttAmmo" and "AMMO" or nil
						if c2 then
							local h = d.Size / 2
							local a1, a2 = d.Position - h, d.Position + h
							local e0 = V.cb[c2]
							if e0 then
								e0.lo = Vector3.new(math.min(e0.lo.X, a1.X), math.min(e0.lo.Y, a1.Y), math.min(e0.lo.Z, a1.Z))
								e0.hi = Vector3.new(math.max(e0.hi.X, a2.X), math.max(e0.hi.Y, a2.Y), math.max(e0.hi.Z, a2.Z))
							else
								V.cb[c2] = { lo = a1, hi = a2 }
							end
						end
					end
				end
				V.home = m:GetPivot()
				V.def = e
			end
			vp.InputBegan:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
					V.drag, V.last, V.idle = true, input.Position, os.clock()
				end
			end)
			UIS.InputChanged:Connect(function(input)
				if V.drag and V.last and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
					local d = input.Position - V.last
					V.last = input.Position
					V.yaw -= d.X * 0.012
					V.pitch = math.clamp(V.pitch + d.Y * 0.01, -1.3, 1.3)
					V.idle = os.clock()
				end
			end)
			UIS.InputEnded:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then V.drag = false end
			end)
			local views = { { "FRONT", 0, 0.1 }, { "SIDE", 1.57, 0.1 }, { "BACK", 3.14, 0.1 }, { "TOP", 1.57, 1.25 } }
			for i, v in ipairs(views) do
				local b = button(panel, v[1], UDim2.fromOffset(92, 30), UDim2.fromOffset(190 + (i - 1) * 100, 556), COL.panel2, function()
					V.yaw, V.pitch, V.idle = v[2], v[3], os.clock()
				end, 11)
				b.ZIndex = 8
			end
			local zbtn = button(panel, "ZOOM: ATTACHMENT", UDim2.fromOffset(150, 26), UDim2.fromOffset(198, 100), COL.panel2, function()
				V.zoomOn = not V.zoomOn
			end, 11)
			zbtn.ZIndex = 9
			local shotsLeft = 0
			local flash
			local function fireOnceView()
				local e = V.def
				if not e or not V.model then return end
				V.kick = 1
				playGunShot(e.id, nil, 0.7 * S.g.sfx)
				if V.muzzle then
					if flash then flash:Destroy() end
					flash = Instance.new("Part")
					flash.Size = Vector3.new(0.35, 0.35, 0.35)
					flash.Shape = Enum.PartType.Ball
					flash.Material = Enum.Material.Neon
					flash.Color = e.color
					flash.Anchored = true
					flash.CanCollide = false
					flash.CFrame = V.muzzle.CFrame * CFrame.new(0, 0, -0.35)
					flash.Parent = vp
					Debris:AddItem(flash, 0.06)
				end
			end
			local testBtn = button(panel, "TEST FIRE", UDim2.fromOffset(392, 44), UDim2.fromOffset(190, 594), COL.accent, function()
				local e = V.def
				if not e then return end
				task.spawn(function()
					local n = e.burst or ((e.auto and math.min(6, math.max(2, e.mag))) or 1)
					for _ = 1, n do
						fireOnceView()
						task.wait(60 / e.rpm)
					end
				end)
			end, 14)
			testBtn.ZIndex = 8
			testBtn.TextColor3 = RGB(20, 20, 20)
			RunService.RenderStepped:Connect(function(dt)
				if not panel.Visible or not V.model then return end
				if not V.drag and os.clock() - V.idle > 2.5 then V.yaw += dt * 0.5 end
				V.kick = math.max(0, V.kick - dt * 9)
				V.model:PivotTo(V.home * CFrame.new(0, V.kick * 0.03, V.kick * 0.22) * CFrame.Angles(V.kick * 0.05, 0, 0))
				local rot = CFrame.Angles(0, V.yaw, 0) * CFrame.Angles(-V.pitch, 0, 0)
				local tc, td = V.center, V.dist
				local eb = V.zoomOn and V.cb and V.cb[GS.cat]
				if eb then
					local sz = eb.hi - eb.lo
					tc = (eb.lo + eb.hi) / 2
					td = math.min(V.dist, math.max(sz.X, sz.Y, sz.Z) * 2.4 + 1.8)
				end
				zbtn.Text = V.zoomOn and (eb and "ZOOM: ATTACHMENT" or "ZOOM: (NOTHING EQUIPPED)") or "ZOOM: FULL GUN"
				local k = math.min(1, dt * 7)
				V.curC = (V.curC or tc):Lerp(tc, k)
				V.curD = V.curD + (td - V.curD) * k
				cam.CFrame = CFrame.lookAt(V.curC + rot:VectorToWorldSpace(Vector3.new(0, 0, V.curD)), V.curC)
			end)
		end
		local clear = button(panel, "CLEAR ALL", UDim2.fromOffset(110, 30), UDim2.fromOffset(590, 54), COL.panel2, function() commit({}) end, 13)
		clear.ZIndex = 8
		local back = button(panel, "BACK", UDim2.fromOffset(100, 30), UDim2.fromOffset(712, 54), COL.accent, function()
			panel.Visible = false
			refreshLoadout()
		end, 13)
		back.ZIndex = 8
		back.TextColor3 = RGB(20, 20, 20)
		LO.openGS = function()
			local def = WEAPONS[LO.focus]
			if not def or XT.wlock(def) then return end
			panel.Visible = true
			render()
		end
		table.insert(profileHooks, function() if panel.Visible then render() end end)
	end

	-- ---------- CAREER page ----------
	-- ---------- BATTLE PASS page ----------
	local bp = makePage("PASS", "BATTLE PASS", "Season 01.  Earn XP in matches to climb 100 levels. Every level has a reward.")
	do
		local banner = new("Frame", { Size = UDim2.fromOffset(780, 76), Position = UDim2.fromOffset(25, 92), BackgroundColor3 = RGB(24, 26, 26), BorderSizePixel = 0 }, bp)
		corner(banner, 6)
		stroke(banner, COL.line, 1, 0.3)
		new("UIGradient", { Rotation = 0, Color = ColorSequence.new(RGB(70, 44, 14), RGB(24, 26, 26)) }, banner)
		local bpLevel = label(banner, { Size = UDim2.fromOffset(220, 40), Position = UDim2.fromOffset(16, 6), Text = "LEVEL 1", TextSize = 34, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent2,
			TextXAlignment = Enum.TextXAlignment.Left })
		local bpTrack = new("Frame", { Size = UDim2.fromOffset(390, 12), Position = UDim2.fromOffset(18, 52), BackgroundColor3 = RGB(12, 14, 14), BorderSizePixel = 0 }, banner)
		corner(bpTrack, 6)
		local bpFill = new("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = COL.accent, BorderSizePixel = 0 }, bpTrack)
		corner(bpFill, 6)
		local bpXp = label(banner, { Size = UDim2.fromOffset(200, 18), Position = UDim2.fromOffset(210, 22), Text = "", TextSize = 12, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Right })
		local buyB = new("TextButton", { Size = UDim2.fromOffset(330, 52), Position = UDim2.fromOffset(436, 12), BackgroundColor3 = COL.accent2, AutoButtonColor = false, Text = "",
			BorderSizePixel = 0 }, banner)
		corner(buyB, 5)
		local buyT = label(buyB, { Size = UDim2.new(1, 0, 0, 30), Position = UDim2.fromOffset(0, 3), Text = "UNLOCK PREMIUM PASS", TextSize = 22, Font = Enum.Font.GothamBlack, TextColor3 = RGB(18, 20, 20) })
		local buyS = label(buyB, { Size = UDim2.new(1, 0, 0, 16), Position = UDim2.fromOffset(0, 32), Text = "R$ 799  -  50+ items, a reward on every level", TextSize = 12, TextColor3 = RGB(60, 40, 0) })
		buyB.Activated:Connect(function()
			uiClick()
			if not (P.owned and P.owned.PASS_PREMIUM) then Net.Pass:FireServer("buy") end
		end)
		local scroll = new("ScrollingFrame", { Size = UDim2.fromOffset(780, 440), Position = UDim2.fromOffset(25, 180), BackgroundTransparency = 1, BorderSizePixel = 0,
			ScrollBarThickness = 8, ScrollBarImageColor3 = COL.accent, CanvasSize = UDim2.fromOffset(86 + PASS_MAX * 132, 0), ScrollingDirection = Enum.ScrollingDirection.X }, bp)
		label(scroll, { Size = UDim2.fromOffset(80, 190), Position = UDim2.fromOffset(0, 6), Text = "FREE", TextSize = 20, Font = Enum.Font.GothamBlack, TextColor3 = COL.text })
		label(scroll, { Size = UDim2.fromOffset(80, 190), Position = UDim2.fromOffset(0, 206), Text = "PREMIUM", TextSize = 16, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent2 })
		local tiles = {}
		local built = false
		local TYPE_TAG = { OP_ = "OPERATOR", C_ = "CAMO", W_ = "WEAPON", A_ = "ATTACHMENT" }
		local function tagOf(id)
			for pre, t in pairs(TYPE_TAG) do if string.sub(id, 1, #pre) == pre then return t end end
			return "ITEM"
		end
		local function makeTile(track, lvl)
			local r = PASS[track][lvl]
			local y = track == "free" and 6 or 206
			local prem = track == "prem"
			local accent = prem and COL.accent2 or COL.accent
			local f = new("Frame", { Size = UDim2.fromOffset(124, 190), Position = UDim2.fromOffset(86 + (lvl - 1) * 132, y), BackgroundColor3 = RGB(22, 24, 24), BorderSizePixel = 0 }, scroll)
			corner(f, 5)
			local st = stroke(f, accent, 1, 0.55)
			local art = new("Frame", { Size = UDim2.fromOffset(124, 104), BackgroundColor3 = prem and RGB(48, 38, 14) or RGB(24, 28, 30), BorderSizePixel = 0, ZIndex = 2 }, f)
			if prem then new("UIGradient", { Rotation = 90, Color = ColorSequence.new(RGB(110, 84, 22), RGB(26, 20, 8)) }, art) end
			corner(art, 5)
			local isItem = r.kind == "item"
			if isItem then
				local it = STORE_BY_ID[r.ids[1]]
				local big = #r.ids > 1
				local vpf = new("ViewportFrame", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ZIndex = 3, Ambient = RGB(255, 255, 255), LightColor = RGB(255, 255, 255),
					LightDirection = Vector3.new(0.3, -0.5, -1) }, art)
				local vcam = new("Camera", { FieldOfView = 22 }, vpf)
				vpf.CurrentCamera = vcam
				local ok = pcall(function()
					local id = r.ids[1]
					if string.sub(id, 1, 3) == "OP_" then
						local m = XT.soldier(id)
						m.Parent = vpf
						vcam.CFrame = CFrame.lookAt(Vector3.new(0, 4.5, 11.5), Vector3.new(0, 3.9, 0))
					else
						local gunId, camoId = "VX44", nil
						if string.sub(id, 1, 2) == "W_" and it and it.weapon then gunId = it.weapon
						elseif string.sub(id, 1, 2) == "C_" then camoId = id end
						if string.sub(id, 1, 2) == "A_" then
							local c, d = XT.buildScene(XT.itemSpec(it), function(inst) inst.Parent = vpf end)
							vcam.FieldOfView = 34
							vcam.CFrame = CFrame.lookAt(c + Vector3.new(0, 0.3, d * 0.95), c)
						else
							local def = camoDef(effDef(WEAPONS[gunId], {}), camoId)
							local m = XT.gun(def)
							m.Parent = vpf
							local bcf, size = m:GetBoundingBox()
							local th = math.tan(math.rad(11))
							local dd = math.max((size.Z / 2 * 1.12) / (th * (124 / 104)), (size.Y / 2 * 1.5) / th) + size.X / 2
							vcam.CFrame = CFrame.lookAt(bcf.Position + Vector3.new(-dd, 0.05, 0), bcf.Position)
						end
					end
				end)
				if not ok then vpf:Destroy() end
				if big then
					label(art, { Size = UDim2.fromOffset(60, 14), Position = UDim2.new(0.5, -30, 1, -18), Text = "+3 MORE", TextSize = 10, Font = Enum.Font.GothamBlack, TextColor3 = accent, ZIndex = 6 })
				end
			elseif r.kind == "xp" then
				label(art, { Size = UDim2.fromScale(1, 0.55), Position = UDim2.fromScale(0, 0.12), Text = "+" .. r.n, TextSize = 34, Font = Enum.Font.GothamBlack, TextColor3 = accent, ZIndex = 3 })
				label(art, { Size = UDim2.fromScale(1, 0.25), Position = UDim2.fromScale(0, 0.62), Text = "BONUS XP", TextSize = 13, TextColor3 = COL.dim, ZIndex = 3 })
			else
				local badge = new("Frame", { Size = UDim2.fromOffset(60, 60), Position = UDim2.fromOffset(32, 22), BackgroundColor3 = RGB(30, 33, 33), Rotation = 45, BorderSizePixel = 0, ZIndex = 3 }, art)
				stroke(badge, accent, 2, 0.1)
				new("Frame", { Size = UDim2.fromOffset(24, 24), Position = UDim2.fromOffset(18, 18), BackgroundColor3 = accent, BorderSizePixel = 0, ZIndex = 4 }, badge)
			end
			local tag = isItem and tagOf(r.ids[1]) or (r.kind == "xp" and "XP" or "EMBLEM")
			label(f, { Size = UDim2.fromOffset(70, 14), Position = UDim2.fromOffset(6, 5), Text = tag, TextSize = 9, TextColor3 = COL.text, BackgroundColor3 = RGB(0, 0, 0), BackgroundTransparency = 0.4, ZIndex = 5 })
			label(f, { Size = UDim2.fromOffset(40, 18), Position = UDim2.new(1, -44, 0, 4), Text = tostring(lvl), TextSize = 14, Font = Enum.Font.GothamBlack, TextColor3 = accent, ZIndex = 5 })
			label(f, { Size = UDim2.fromOffset(116, 36), Position = UDim2.fromOffset(4, 106), Visible = (r.kind == "item" or r.kind == "card"), Text = r.label, TextSize = 13, Font = Enum.Font.GothamBlack, TextWrapped = true, ZIndex = 5 })
			local btn = new("TextButton", { Size = UDim2.fromOffset(108, 30), Position = UDim2.fromOffset(8, 152), BackgroundColor3 = accent, AutoButtonColor = false, Text = "", BorderSizePixel = 0, ZIndex = 5 }, f)
			corner(btn, 4)
			local bt = label(btn, { Size = UDim2.fromScale(1, 1), Text = "CLAIM", TextSize = 15, Font = Enum.Font.GothamBlack, TextColor3 = RGB(18, 20, 20), ZIndex = 6 })
			btn.Activated:Connect(function()
				uiClick()
				Net.Pass:FireServer("claim", track, lvl)
			end)
			tiles[#tiles + 1] = { f = f, st = st, btn = btn, bt = bt, track = track, lvl = lvl, accent = accent }
		end
		local function refreshPass()
			bpLevel.Text = "LEVEL " .. P.level
			bpFill.Size = UDim2.fromScale(math.clamp(P.into / math.max(1, P.need), 0.02, 1), 1)
			bpXp.Text = string.format("%d / %d XP", P.into, P.need)
			local own = P.owned and P.owned.PASS_PREMIUM
			buyB.BackgroundColor3 = own and RGB(40, 120, 70) or COL.accent2
			buyT.Text = own and "PREMIUM ACTIVE" or "UNLOCK PREMIUM PASS"
			buyT.TextColor3 = own and RGB(255, 255, 255) or RGB(18, 20, 20)
			buyS.Text = own and "Every premium reward is yours" or "R$ 799  -  50+ items, a reward on every level"
			buyS.TextColor3 = own and RGB(200, 255, 220) or RGB(60, 40, 0)
			local cl = P.passClaimed or {}
			for _, t in ipairs(tiles) do
				local got = cl[t.track .. t.lvl]
				local reached = P.level >= t.lvl
				local locked = t.track == "prem" and not own
				if got then
					t.bt.Text = "CLAIMED"
					t.btn.BackgroundColor3 = RGB(40, 120, 70)
					t.bt.TextColor3 = RGB(255, 255, 255)
					t.st.Transparency = 0.8
				elseif locked then
					t.bt.Text = reached and "PREMIUM" or "LOCKED"
					t.btn.BackgroundColor3 = RGB(46, 48, 48)
					t.bt.TextColor3 = COL.dim
					t.st.Transparency = 0.7
				elseif reached then
					t.bt.Text = "CLAIM"
					t.btn.BackgroundColor3 = t.accent
					t.bt.TextColor3 = RGB(18, 20, 20)
					t.st.Transparency = 0
				else
					t.bt.Text = "LVL " .. t.lvl
					t.btn.BackgroundColor3 = RGB(46, 48, 48)
					t.bt.TextColor3 = COL.dim
					t.st.Transparency = 0.7
				end
			end
		end
		function XT.buildPass()
			if built then return end
			built = true
			for l = 1, PASS_MAX do
				makeTile("free", l)
				makeTile("prem", l)
			end
			refreshPass()
			XT.skinFonts(bp)
			local lv = math.clamp(P.level - 1, 0, PASS_MAX - 4)
			scroll.CanvasPosition = Vector2.new(lv * 132, 0)
		end
		table.insert(profileHooks, function() if built then refreshPass() else bpLevel.Text = "LEVEL " .. P.level end end)
	end

	local ca = makePage("CAREER", "CAREER", "Your lifetime combat record.")
	local careerVals = {}
	local careerDefs = {
		{ "LEVEL", function() return tostring(P.level) end }, { "KILLS", function() return tostring(P.kills) end },
		{ "DEATHS", function() return tostring(P.deaths) end },
		{ "K/D RATIO", function() return string.format("%.2f", P.kills / math.max(1, P.deaths)) end },
		{ "ASSISTS", function() return tostring(P.assists) end }, { "HEADSHOTS", function() return tostring(P.headshots) end },
		{ "WINS", function() return tostring(P.wins) end }, { "LOSSES", function() return tostring(P.losses) end },
		{ "MATCHES", function() return tostring(P.matches) end },
		{ "ACCURACY", function() return string.format("%d%%", P.shots > 0 and math.floor(P.hits / P.shots * 100 + 0.5) or 0) end },
		{ "WIN RATE", function() return string.format("%d%%", P.matches > 0 and math.floor(P.wins / P.matches * 100 + 0.5) or 0) end },
		{ "TOTAL XP", function() return tostring(P.xp) end },
	}
	local RANKS = { { 1, "ROOKIE" }, { 5, "OPERATIVE" }, { 10, "VETERAN" }, { 20, "ELITE" }, { 30, "LEGEND" }, { 50, "GHOST" } }
	local banner = new("Frame", { Size = UDim2.fromOffset(780, 116), Position = UDim2.fromOffset(25, 94), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0 }, ca)
	new("UIGradient", { Rotation = 0, Color = ColorSequence.new(RGB(255, 106, 10), RGB(96, 108, 116)) }, banner)
	corner(banner, 22)
	stroke(banner, RGB(255, 255, 255), 2, 0.6)
	local lvCircle = new("Frame", { Size = UDim2.fromOffset(88, 88), Position = UDim2.fromOffset(16, 14), BackgroundColor3 = COL.accent2, BorderSizePixel = 0 }, banner)
	new("UICorner", { CornerRadius = UDim.new(1, 0) }, lvCircle)
	stroke(lvCircle, RGB(18, 20, 20), 4, 0)
	local lvBig = label(lvCircle, { Size = UDim2.fromScale(1, 1), Text = "1", TextSize = 52, Font = Enum.Font.GothamBlack, TextColor3 = RGB(18, 20, 20) })
	local rankLbl = label(banner, { Size = UDim2.fromOffset(400, 44), Position = UDim2.fromOffset(122, 10), Text = "ROOKIE", TextSize = 42, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left })
	local xpTxt = label(banner, { Size = UDim2.fromOffset(640, 20), Position = UDim2.fromOffset(124, 56), Text = "", TextSize = 15, Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Left })
	local xpTrack = new("Frame", { Size = UDim2.fromOffset(630, 16), Position = UDim2.fromOffset(124, 84), BackgroundColor3 = RGB(18, 20, 20), BorderSizePixel = 0 }, banner)
	new("UICorner", { CornerRadius = UDim.new(1, 0) }, xpTrack)
	local xpFill = new("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = COL.accent2, BorderSizePixel = 0 }, xpTrack)
	new("UICorner", { CornerRadius = UDim.new(1, 0) }, xpFill)
	table.insert(profileHooks, function()
		lvBig.Text = tostring(P.level)
		local rk = RANKS[1][2]
		for _, r in ipairs(RANKS) do if P.level >= r[1] then rk = r[2] end end
		rankLbl.Text = rk
		xpTxt.Text = string.format("LEVEL %d   -   %d / %d XP to level %d", P.level, P.into, P.need, P.level + 1)
		xpFill.Size = UDim2.fromScale(math.clamp(P.into / math.max(1, P.need), 0.02, 1), 1)
	end)
	local TILE_COL = { COL.accent, COL.accent2, COL.cyan, COL.orange }
	for i, d in ipairs(careerDefs) do
		local col, row = (i - 1) % 4, (i - 1) // 4
		local c = new("Frame", { Size = UDim2.fromOffset(180, 92), Position = UDim2.fromOffset(25 + col * 200, 226 + row * 106), BackgroundColor3 = COL.panel2,
			BorderSizePixel = 0 }, ca)
		corner(c, 16)
		stroke(c, COL.line, 1.5, 0.5)
		local tc = TILE_COL[(col + row) % 4 + 1]
		local bar = new("Frame", { Size = UDim2.fromOffset(6, 60), Position = UDim2.fromOffset(0, 16), BackgroundColor3 = tc, BorderSizePixel = 0 }, c)
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, bar)
		label(c, { Size = UDim2.fromOffset(150, 20), Position = UDim2.fromOffset(18, 10), Text = d[1], TextSize = 13, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left })
		careerVals[i] = label(c, { Size = UDim2.fromOffset(150, 50), Position = UDim2.fromOffset(18, 32), Text = "0", TextSize = 40, TextColor3 = tc, Font = Enum.Font.GothamBlack,
			TextXAlignment = Enum.TextXAlignment.Left })
	end
	label(ca, { Size = UDim2.fromOffset(780, 40), Position = UDim2.fromOffset(25, 548), TextSize = 13, TextColor3 = COL.dim, Font = Enum.Font.Gotham,
		Text = "Stats save automatically to your account. If API access is off in Studio, progress is temporary.",
		TextXAlignment = Enum.TextXAlignment.Left, TextWrapped = true })

	-- ---------- SIDE PANEL: your operator + stats, shown next to every page so the screen is always full ----------
	do
		local sp2 = new("Frame", { Name = "SidePanel", Size = UDim2.fromOffset(400, 650), Position = UDim2.fromOffset(850, 0), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, Visible = false, ZIndex = 6 }, menuRoot)
		new("UIGradient", { Rotation = 90, Color = ColorSequence.new(RGB(44, 52, 56), RGB(16, 19, 21)) }, sp2)
		corner(sp2, 4)
		stroke(sp2, COL.line, 1.5, 0.4)
		new("Frame", { Size = UDim2.new(1, 0, 0, 5), BackgroundColor3 = COL.accent, BorderSizePixel = 0, ZIndex = 7 }, sp2)
		XT.sidePanel = sp2
		label(sp2, { Size = UDim2.fromOffset(300, 24), Position = UDim2.fromOffset(22, 18), Text = "YOUR OPERATOR", TextSize = 20, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 8 })
		local glow = new("Frame", { Size = UDim2.fromOffset(300, 300), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(200, 250), BackgroundColor3 = COL.accent, BackgroundTransparency = 0.8, BorderSizePixel = 0, ZIndex = 7 }, sp2)
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, glow)
		local v = new("ViewportFrame", { Size = UDim2.fromOffset(400, 410), Position = UDim2.fromOffset(0, 44), BackgroundTransparency = 1, ZIndex = 8, Ambient = RGB(170, 168, 168), LightColor = RGB(255, 228, 200),
			LightDirection = Vector3.new(-1, -0.6, -0.9) }, sp2)
		local vcam = new("Camera", { FieldOfView = 30 }, v)
		v.CurrentCamera = vcam
		local opName = label(sp2, { Size = UDim2.fromOffset(360, 50), Position = UDim2.fromOffset(22, 458), Text = "", TextSize = 46, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 8 })
		local opDesc = label(sp2, { Size = UDim2.fromOffset(360, 18), Position = UDim2.fromOffset(24, 508), Text = "", TextSize = 13, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 8, TextTruncate = Enum.TextTruncate.AtEnd })
		local lvl = label(sp2, { Size = UDim2.fromOffset(200, 22), Position = UDim2.fromOffset(24, 536), Text = "", TextSize = 18, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent2, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 8 })
		local barBg = new("Frame", { Size = UDim2.fromOffset(352, 10), Position = UDim2.fromOffset(24, 562), BackgroundColor3 = RGB(8, 10, 10), BorderSizePixel = 0, ZIndex = 8 }, sp2)
		local barFill = new("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = COL.accent, BorderSizePixel = 0, ZIndex = 9 }, barBg)
		local statL = {}
		for i, nm in ipairs({ "KILLS", "K/D", "WINS" }) do
			local t = new("Frame", { Size = UDim2.fromOffset(110, 52), Position = UDim2.fromOffset(24 + (i - 1) * 121, 582), BackgroundColor3 = COL.panel2, BorderSizePixel = 0, ZIndex = 8 }, sp2)
			corner(t, 3)
			label(t, { Size = UDim2.fromOffset(100, 16), Position = UDim2.fromOffset(8, 4), Text = nm, TextSize = 11, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 9 })
			statL[i] = label(t, { Size = UDim2.fromOffset(100, 30), Position = UDim2.fromOffset(8, 18), Text = "0", TextSize = 26, Font = Enum.Font.GothamBlack, TextColor3 = COL.text, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 9 })
		end
		local m, g
		local function build()
			if m then m:Destroy() end
			if g then g:Destroy() end
			local id = (P.equip and P.equip.operator) or "OP_STANDARD"
			local op = OPERATORS[id] or OPERATORS.OP_STANDARD
			local wd = WEAPONS[(P.loadout and P.loadout.primary) or "VX44"] or WEAPONS.VX44
			m = XT.soldier(id)
			m.Parent = v
			g = XT.gun(wd)
			g.Parent = v
			opName.Text = op.name
			opName.TextColor3 = op.accent
			opDesc.Text = op.desc or ""
			lvl.Text = string.format("LEVEL %d   -   %d / %d XP", P.level or 1, P.into or 0, P.need or 1)
			barFill.Size = UDim2.fromScale(math.clamp((P.into or 0) / math.max(1, P.need or 1), 0.02, 1), 1)
			statL[1].Text = tostring(P.kills or 0)
			statL[2].Text = string.format("%.2f", (P.kills or 0) / math.max(1, P.deaths or 0))
			statL[3].Text = tostring(P.wins or 0)
		end
		table.insert(profileHooks, function() if sp2.Visible then pcall(build) end end)
		sp2:GetPropertyChangedSignal("Visible"):Connect(function() if sp2.Visible then pcall(build) end end)
		RunService.RenderStepped:Connect(function()
			if not sp2.Visible or not m then return end
			local t = os.clock()
			local cf = CFrame.new(0, 0.15 + math.sin(t * 1.6) * 0.04, 0) * CFrame.Angles(0, math.pi - 0.3 + math.sin(t * 0.6) * 0.35, 0) * CFrame.new(0, 3, 0)
			m:PivotTo(cf)
			g:PivotTo(cf * XT.gunHold)
			vcam.CFrame = CFrame.lookAt(Vector3.new(0, 4.3, 17), Vector3.new(0, 3.4, 0))
		end)
	end

	-- ---------- profile + status refresh ----------
	local function refreshProfileUI()
		PC.pcLevel.Text = "LEVEL " .. P.level
		pcBar.Size = UDim2.fromScale(math.clamp(P.into / math.max(1, P.need), 0, 1), 1)
		PC.pcXp.Text = string.format("%d / %d XP", P.into, P.need)
		PC.pcSave.Text = P.saved and "Progress saves to your account" or "Temporary profile (not saved)"
		for i, d in ipairs(careerDefs) do careerVals[i].Text = d[2]() end
		LO.primary, LO.secondary = P.loadout.primary, P.loadout.secondary
		if not LO.rows[LO.focus] then LO.focus = LO.primary end
		refreshLoadout()
		for _, f in ipairs(profileHooks) do pcall(f) end
	end
	Net.Profile.OnClientEvent:Connect(function(t)
		if type(t) ~= "table" then return end
		XT.gotProfile = true
		for k, v in pairs(t) do
			if k ~= "settings" and k ~= "loadout" then P[k] = v end
		end
		if type(t.loadout) == "table" and WEAPONS[t.loadout.primary] and WEAPONS[t.loadout.secondary] then
			P.loadout = t.loadout
			if type(P.loadout.att) ~= "table" then P.loadout.att = {} end
		end
		if type(t.settings) == "table" and not settingsLoaded then
			settingsLoaded = true
			mergeSettings(t.settings)
			settingsChanged()
		end
		refreshProfileUI()
	end)

	local function refreshDeploy()
		local ph = phase()
		local q = player:GetAttribute("Queued") == true
		local status = State:GetAttribute("Status") or ""
		TS.stTitle.Text = inMatch() and "IN MATCH" or (q and "SEARCHING FOR A MATCH" or "READY TO DEPLOY")
		TS.stSub.Text = string.upper(tostring(status)) .. (q and "   -   PRESS CANCEL TO LEAVE THE QUEUE" or "")
		if inMatch() then
			queueBtn.Text = "LEAVE MATCH"
			queueBtn.BackgroundColor3 = COL.red
		elseif q then
			queueBtn.Text = "CANCEL QUEUE"
			queueBtn.BackgroundColor3 = COL.line
		else
			queueBtn.Text = "QUEUE FOR MATCH"
			queueBtn.BackgroundColor3 = COL.accent
		end
	end
	State:GetAttributeChangedSignal("Phase"):Connect(function()
		refreshDeploy()
		if UI.refresh then UI.refresh() end
	end)
	State:GetAttributeChangedSignal("Status"):Connect(refreshDeploy)
	player:GetAttributeChangedSignal("Queued"):Connect(refreshDeploy)

	-- =================================================================
	-- MATCHMAKING SCREEN: shown while queued. Animated map art + minimap, 3D squad line-up, lobby list, CANCEL and INVITE FRIENDS
	-- =================================================================
	do
		local SocialService = game:GetService("SocialService")
		local WHITE = RGB(255, 255, 255)
		local LEFT, CENTER, RIGHT = Enum.TextXAlignment.Left, Enum.TextXAlignment.Center, Enum.TextXAlignment.Right
		local INKC = RGB(18, 20, 20)
		local mm = new("Frame", { Name = "Matchmaking", Size = UDim2.fromScale(1, 1), BackgroundColor3 = WHITE, BorderSizePixel = 0, Visible = false, Active = true, ZIndex = 24, ClipsDescendants = true }, gui)
		new("UIGradient", { Rotation = 62, Color = ColorSequence.new({ ColorSequenceKeypoint.new(0, RGB(70, 77, 77)), ColorSequenceKeypoint.new(0.45, RGB(33, 36, 36)), ColorSequenceKeypoint.new(1, RGB(12, 13, 13)) }) }, mm)
		-- backdrop slabs + a sweeping light stripe
		local function slab(x, y, w, h, rot, col, tr)
			local s = new("Frame", { Size = UDim2.fromOffset(w, h), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(x, y), Rotation = rot, BackgroundColor3 = col, BackgroundTransparency = tr, BorderSizePixel = 0, ZIndex = 24 }, mm)
			return s
		end
		slab(0.12, 0.92, 2400, 220, -14, COL.accent, 0.86)
		slab(0.9, 0.08, 2000, 160, -14, RGB(96, 108, 116), 0.8)
		slab(0.5, 1.02, 2600, 40, -14, COL.accent2, 0.82)
		local sweep = slab(-0.3, 0.5, 120, 1600, 22, COL.accent2, 0.92)
		TweenService:Create(sweep, TweenInfo.new(9, Enum.EasingStyle.Linear, Enum.EasingDirection.Out, -1), { Position = UDim2.fromScale(1.3, 0.5) }):Play()

		local root = new("Frame", { Size = UDim2.fromOffset(1280, 720), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), BackgroundTransparency = 1, ZIndex = 25 }, mm)
		local rs = new("UIScale", {}, root)
		local function fit()
			local v = camera.ViewportSize
			rs.Scale = math.clamp(math.min(v.X / 1280, v.Y / 720), 0.35, 1.7)
		end
		fit()
		camera:GetPropertyChangedSignal("ViewportSize"):Connect(fit)
		local function fr(parent, x, y, w, h, col, tr, z, round)
			local f = new("Frame", { Position = UDim2.fromOffset(x, y), Size = UDim2.fromOffset(w, h), BackgroundColor3 = col, BackgroundTransparency = tr or 0, BorderSizePixel = 0, ZIndex = z or 27 }, parent)
			if round then new("UICorner", { CornerRadius = UDim.new(round, 0) }, f) end
			return f
		end
		local function card(parent, x, y, w, h, z)
			local f = fr(parent, x, y, w, h, COL.panel, 0.08, z or 27)
			new("UICorner", { CornerRadius = UDim.new(0, 18) }, f)
			stroke(f, COL.line, 2, 0.45)
			return f
		end

		-- ---------- TOP BAR ----------
		local leaveBtn = button(root, "LEAVE QUEUE", UDim2.fromOffset(200, 44), UDim2.fromOffset(1050, 662), COL.accent, function() XT.autoVote = false Net.Queue:FireServer(false) end, 22)
		leaveBtn.Font = Enum.Font.GothamBlack
		local startBtn = button(root, "START NOW", UDim2.fromOffset(190, 44), UDim2.fromOffset(850, 662), COL.panel2, function()
			Net.Vote:FireServer("start", "now")
		end, 20)
		startBtn.Font = Enum.Font.GothamBlack
		startBtn.ZIndex = 30
		stroke(startBtn, COL.accent2, 2, 0.3)
		leaveBtn.ZIndex = 30
		stroke(leaveBtn, WHITE, 2, 0.6)
		label(root, { Size = UDim2.fromOffset(300, 56), Position = UDim2.fromOffset(240, 8), Text = "LOBBY", TextSize = 60, Font = Enum.Font.GothamBlack, TextXAlignment = LEFT, ZIndex = 28 })
		local mmStatus = label(root, { Size = UDim2.fromOffset(330, 20), Position = UDim2.fromOffset(244, 62), Text = "", TextSize = 14, TextColor3 = COL.accent2, Font = Enum.Font.GothamBold, TextXAlignment = LEFT, ZIndex = 28, TextTruncate = Enum.TextTruncate.AtEnd })
		-- 3-step progress
		local steps = {}
		local STEP_NAMES = { "MATCHMAKING", "LOCKED IN", "DEPLOY" }
		for i = 1, 3 do
			local p = fr(root, 590 + (i - 1) * 168, 28, 160, 44, COL.panel2, 0.1, 28)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, p)
			local num = label(p, { Size = UDim2.fromOffset(30, 30), Position = UDim2.fromOffset(8, 7), Text = tostring(i), TextSize = 22, Font = Enum.Font.GothamBlack, BackgroundTransparency = 0, BackgroundColor3 = COL.panel, ZIndex = 29 })
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, num)
			local t = label(p, { Size = UDim2.fromOffset(110, 44), Position = UDim2.fromOffset(44, 0), Text = STEP_NAMES[i], TextSize = 18, Font = Enum.Font.GothamBlack, TextXAlignment = LEFT, ZIndex = 29 })
			steps[i] = { f = p, num = num, t = t }
		end
		local mmTimer = label(root, { Size = UDim2.fromOffset(200, 48), AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromOffset(1250, 18), Text = "00:00", TextSize = 50, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent2, TextXAlignment = RIGHT, ZIndex = 28 })
		local timerCap = label(root, { Size = UDim2.fromOffset(200, 16), AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromOffset(1250, 66), Text = "TIME IN LOBBY", TextSize = 12, TextColor3 = COL.dim, TextXAlignment = RIGHT, ZIndex = 28 })

		-- ---------- MAP STAGE (centre) ----------
		local stage = fr(root, 350, 100, 580, 200, RGB(9, 10, 10), 0, 27)
		new("UICorner", { CornerRadius = UDim.new(0, 20) }, stage)
		stroke(stage, COL.accent2, 3, 0.2)
		stage.ClipsDescendants = true
		local artWrap = fr(stage, 0, -90, 720, 430, RGB(0, 0, 0), 1, 26)
		new("UIScale", { Scale = 580 / 720 }, artWrap)
		local artHolder = fr(artWrap, 0, 0, 720, 430, RGB(10, 13, 14), 1, 26)
		artHolder.ClipsDescendants = true
		local function sky(parent, c0, c1, c2)
			local s = fr(parent, 0, 0, 720, 430, RGB(255, 255, 255), 0, 26)
			new("UIGradient", { Rotation = 90, Color = ColorSequence.new({ ColorSequenceKeypoint.new(0, c0), ColorSequenceKeypoint.new(0.55, c1), ColorSequenceKeypoint.new(1, c2) }) }, s)
			return s
		end
		local ART = {}
		ART.NEON = function(p)
			sky(p, RGB(84, 100, 124), RGB(170, 180, 192), RGB(232, 214, 184))
			fr(p, 430, 110, 260, 260, RGB(255, 236, 196), 0.6, 26, 0.5)
			fr(p, 490, 170, 140, 140, RGB(255, 246, 220), 0.4, 26, 0.5)
			local rng = Random.new(33)
			-- far skyline (hazy)
			for i = 0, 15 do
				local h = rng:NextNumber(110, 250)
				fr(p, i * 48 - 8, 320 - h, 44, h, RGB(138, 150, 166), 0, 26)
			end
			-- mid towers with lit windows
			for i = 0, 6 do
				local w, h = rng:NextNumber(70, 112), rng:NextNumber(150, 290)
				local x = i * 108 - 24
				local tone = (i % 2 == 0) and RGB(66, 82, 100) or RGB(96, 100, 106)
				fr(p, x, 340 - h, w, h, tone, 0, 27)
				fr(p, x + 4, 340 - h - 6, w - 8, 8, RGB(52, 60, 72), 0, 27)
				for wy = 0, math.floor(h / 22) - 1 do
					for wx = 0, math.floor(w / 15) - 1 do
						local lit = rng:NextNumber() < 0.28
						fr(p, x + 7 + wx * 15, 340 - h + 10 + wy * 22, 8, 12, lit and RGB(250, 226, 160) or RGB(40, 54, 68), lit and 0.1 or 0.2, 28)
					end
				end
			end
			-- overpass + street level
			fr(p, 0, 322, 720, 14, RGB(104, 106, 110), 0, 29)
			for i = 0, 8 do fr(p, 24 + i * 84, 336, 12, 40, RGB(96, 98, 100), 0, 29) end
			fr(p, 0, 372, 720, 58, RGB(46, 48, 52), 0, 30)
			for i = 0, 8 do fr(p, i * 90 + 10, 398, 46, 4, RGB(206, 176, 70), 0, 31) end
			for i = 0, 3 do fr(p, 60 + i * 170, 354 + (i % 2) * 6, 64, 22, RGB(30, 34, 38), 0, 31) end
			local fog = fr(p, 0, 250, 720, 130, RGB(214, 218, 224), 0.7, 32)
			new("UIGradient", { Rotation = 90, Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0.4) }) }, fog)
		end
		ART.STATION = function(p)
			sky(p, RGB(92, 112, 150), RGB(206, 168, 150), RGB(246, 200, 140))
			fr(p, 500, 60, 150, 150, RGB(255, 226, 170), 0.45, 26, 0.5)
			for i = 0, 9 do
				local h = 60 + (i * 53 % 80)
				fr(p, i * 78 - 10, 250 - h, 60, h + 30, RGB(92 + (i % 3) * 8, 88 + (i % 4) * 6, 104), 0, 27)
			end
			-- grand hall facade
			fr(p, 150, 120, 420, 150, RGB(190, 178, 154), 0, 28)
			fr(p, 130, 108, 460, 16, RGB(160, 148, 128), 0, 29)
			for _, ax in ipairs({ 190, 310, 430 }) do
				fr(p, ax, 160, 100, 110, RGB(34, 40, 52), 0, 29)
				fr(p, ax + 6, 150, 88, 20, RGB(236, 206, 150), 0.1, 30)
			end
			fr(p, 250, 70, 220, 56, RGB(206, 196, 172), 0, 29)
			fr(p, 262, 80, 196, 36, RGB(26, 52, 44), 0, 30)
			-- clock tower
			fr(p, 600, 30, 44, 240, RGB(190, 178, 154), 0, 29)
			fr(p, 594, 24, 56, 14, RGB(44, 84, 70), 0, 30)
			fr(p, 612, 60, 20, 20, RGB(240, 238, 226), 0, 31, 0.5)
			-- platform canopy + train
			fr(p, 0, 262, 720, 14, RGB(44, 84, 70), 0, 31)
			for i = 0, 7 do fr(p, 40 + i * 92, 276, 8, 80, RGB(44, 84, 70), 0, 31) end
			fr(p, 0, 340, 720, 100, RGB(70, 68, 66), 0, 32)
			for i = 0, 5 do fr(p, 30 + i * 118, 296, 108, 48, (i == 2) and RGB(44, 70, 120) or RGB(150, 40, 40), 0, 33) end
			for i = 0, 5 do fr(p, 40 + i * 118, 306, 88, 14, RGB(30, 40, 52), 0, 34) end
			fr(p, 0, 342, 720, 8, RGB(232, 178, 36), 0, 34)
			fr(p, 0, 396, 720, 40, RGB(18, 20, 24), 0, 35)
		end
		ART.GRID = function(p)
			sky(p, RGB(8, 12, 14), RGB(18, 26, 28), RGB(30, 40, 42))
			fr(p, 0, 60, 720, 270, RGB(44, 52, 52), 0, 26)          -- back wall
			fr(p, 0, 0, 720, 64, RGB(16, 20, 22), 0, 27)            -- ceiling
			fr(p, 0, 330, 720, 100, RGB(22, 26, 30), 0, 27)         -- carpet
			-- night windows
			for i = 0, 4 do
				fr(p, 40 + i * 138, 96, 104, 84, RGB(14, 26, 44), 0, 28)
				for k = 0, 3 do fr(p, 40 + i * 138, 104 + k * 20 + (i % 2) * 4, 104, 6, RGB(28, 32, 34), 0, 29) end
			end
			-- ceiling lights, two of them dead
			for i = 0, 5 do
				local dead = (i == 1 or i == 4)
				fr(p, 40 + i * 114, 24, 76, 12, dead and RGB(40, 42, 42) or RGB(186, 214, 200), dead and 0 or 0.08, 31)
				if not dead then fr(p, 20 + i * 114, 36, 116, 120, RGB(150, 190, 170), 0.9, 30) end
			end
			-- cubicle rows
			for i = 0, 7 do
				fr(p, i * 94 - 10, 210, 88, 130, RGB(56, 66, 78), 0, 32)
				fr(p, i * 94 - 10, 210, 88, 8, RGB(40, 48, 58), 0, 33)
				fr(p, i * 94 + 12, 270, 46, 6, RGB(88, 66, 48), 0, 34)
				local lc = (i % 3 == 0) and RGB(70, 170, 130) or RGB(80, 150, 205)
				fr(p, i * 94 + 18, 244, 34, 26, lc, 0.1, 35)
				fr(p, i * 94 + 8, 236, 54, 44, lc, 0.88, 34)
			end
			-- foreground desk with a laptop, chair knocked over
			fr(p, 0, 360, 720, 70, RGB(10, 12, 14), 0, 36)
			fr(p, 440, 330, 200, 14, RGB(92, 68, 48), 0, 37)
			fr(p, 470, 270, 120, 62, RGB(40, 44, 48), 0, 38)
			fr(p, 478, 276, 104, 48, RGB(90, 200, 170), 0.15, 39)
			fr(p, 430, 262, 220, 140, RGB(90, 200, 170), 0.9, 37)
			fr(p, 60, 380, 120, 16, RGB(30, 32, 36), 0, 38, 0.5)
			-- red exit sign
			fr(p, 330, 66, 60, 20, RGB(220, 36, 30), 0, 33)
			fr(p, 280, 64, 160, 110, RGB(220, 36, 30), 0.92, 32)
		end
		local MINI = {}
		MINI.NEON = function(p)
			fr(p, 10, 10, 168, 168, RGB(44, 46, 50), 0, 31)
			for gx = 0, 3 do
				for gy = 0, 3 do
					local col = RGB(112, 114, 110)
					if gx == 1 and gy == 1 then col = RGB(150, 140, 120)
					elseif gx == 2 and gy == 1 then col = RGB(92, 110, 126)
					elseif gx == 1 and gy == 2 then col = RGB(100, 100, 104)
					elseif gx == 2 and gy == 2 then col = RGB(70, 110, 62) end
					fr(p, 12 + gx * 42, 12 + gy * 42, 38, 38, col, 0, 32)
				end
			end
			fr(p, 80, 80, 28, 28, RGB(196, 176, 130), 0, 33, 0.5)
			fr(p, 12, 12 + 2 * 42 - 4, 164, 3, RGB(150, 150, 154), 0, 34)
		end
		MINI.STATION = function(p)
			fr(p, 10, 10, 168, 168, RGB(52, 52, 56), 0, 31)
			fr(p, 40, 120, 108, 24, RGB(176, 166, 142), 0, 32)      -- hall
			fr(p, 14, 14, 160, 24, RGB(86, 86, 92), 0, 32)          -- bravo yard
			for i = 0, 3 do fr(p, 14, 62 + i * 12, 160, 6, (i % 2 == 0) and RGB(150, 148, 142) or RGB(96, 90, 80), 0, 33) end
			fr(p, 14, 150, 160, 22, RGB(70, 72, 78), 0, 32)         -- car park
			fr(p, 20, 112, 30, 36, RGB(150, 140, 120), 0, 33)
			fr(p, 138, 112, 30, 36, RGB(120, 80, 64), 0, 33)
			fr(p, 40, 62, 3, 52, RGB(120, 190, 210), 0, 34)
			fr(p, 146, 62, 3, 52, RGB(120, 190, 210), 0, 34)
		end
		MINI.GRID = function(p)
			fr(p, 10, 10, 168, 168, RGB(22, 28, 30), 0, 31)
			fr(p, 14, 14, 30, 160, RGB(60, 70, 72), 0, 32)                   -- west lobby
			fr(p, 142, 14, 30, 160, RGB(48, 54, 62), 0, 32)                  -- east wing
			fr(p, 46, 14, 94, 26, RGB(70, 84, 86), 0, 32)                    -- meeting rooms
			for i = 0, 5 do fr(p, 46 + i * 16, 14, 1, 26, RGB(20, 24, 26), 0, 33) end
			fr(p, 46, 140, 94, 34, RGB(80, 74, 60), 0, 32)                   -- mezzanine
			for gx = 0, 2 do
				for gy = 0, 3 do
					if not (gx == 1 and (gy == 1 or gy == 2)) then fr(p, 52 + gx * 34, 48 + gy * 22, 22, 12, RGB(56, 70, 90), 0, 33) end
				end
			end
			fr(p, 20, 80, 12, 24, RGB(110, 170, 210), 0, 34)                 -- alpha
			fr(p, 154, 80, 12, 24, RGB(230, 150, 70), 0, 34)                 -- bravo
		end

		-- minimap badge (top right of the stage)
		local miniBox = fr(stage, 580 - 128, 14, 114, 114, RGB(9, 10, 10), 0.1, 45)
		new("UICorner", { CornerRadius = UDim.new(0, 14) }, miniBox)
		stroke(miniBox, COL.accent2, 2, 0.2)
		miniBox.ClipsDescendants = true
		local miniWrap = fr(miniBox, 0, 0, 186, 186, RGB(0, 0, 0), 1, 45)
		new("UIScale", { Scale = 114 / 186 }, miniWrap)
		local miniArt = fr(miniWrap, 0, 0, 186, 186, RGB(0, 0, 0), 1, 45)
		local grad = fr(stage, 0, 40, 580, 160, RGB(6, 6, 6), 0, 44)
		new("UIGradient", { Rotation = 90, Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0.05) }) }, grad)
		local mapName = label(stage, { Size = UDim2.fromOffset(440, 54), Position = UDim2.fromOffset(22, 84), Text = "", TextSize = 50, Font = Enum.Font.GothamBlack, TextXAlignment = LEFT, ZIndex = 46 })
		local mapTag = label(stage, { Size = UDim2.fromOffset(540, 18), Position = UDim2.fromOffset(24, 140), Text = "", TextSize = 13, Font = Enum.Font.GothamBold, TextXAlignment = LEFT, ZIndex = 46 })
		local mapDesc = label(stage, { Size = UDim2.fromOffset(536, 34), Position = UDim2.fromOffset(24, 160), Text = "", TextSize = 12, TextColor3 = COL.dim, Font = Enum.Font.Gotham, TextXAlignment = LEFT,
			TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true, ZIndex = 46 })
		local mapBadge = label(stage, { Size = UDim2.fromOffset(210, 28), Position = UDim2.fromOffset(16, 14), Text = "", TextSize = 16, Font = Enum.Font.GothamBlack, TextColor3 = INKC,
			BackgroundColor3 = COL.accent2, BackgroundTransparency = 0, ZIndex = 46 })
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, mapBadge)
		local flash = fr(stage, 0, 0, 580, 200, WHITE, 1, 48)
		local shownMap
		local function showMap(id, locked)
			if shownMap == id then return end
			shownMap = id
			artHolder:ClearAllChildren()
			miniArt:ClearAllChildren()
			local m = MAPS[id]
			if not m or not ART[id] then return end
			ART[id](artHolder)
			MINI[id](miniArt)
			mapName.Text = m.name
			mapTag.Text = m.tag
			mapTag.TextColor3 = m.accent
			mapDesc.Text = (#m.desc > 120) and (string.sub(m.desc, 1, 117) .. "...") or m.desc
			mapBadge.Text = locked and "YOUR PICK" or "MAPS IN ROTATION"
			flash.BackgroundTransparency = 0.5
			tween(flash, 0.5, { BackgroundTransparency = 1 })
		end

		-- ---------- VOTE ROWS ----------
		local voteHdr1 = label(root, { Size = UDim2.fromOffset(400, 22), Position = UDim2.fromOffset(352, 308), Text = "VOTE FOR THE MAP", TextSize = 20, TextColor3 = COL.accent2, Font = Enum.Font.GothamBlack, TextXAlignment = LEFT, ZIndex = 28 })
		local voteHdr2 = label(root, { Size = UDim2.fromOffset(300, 22), Position = UDim2.fromOffset(352, 430), Text = "VOTE FOR THE MODE", TextSize = 20, TextColor3 = COL.accent2, Font = Enum.Font.GothamBlack, TextXAlignment = LEFT, ZIndex = 28 })
		local voteCount = label(root, { Size = UDim2.fromOffset(300, 22), AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromOffset(930, 308), Text = "", TextSize = 20, TextColor3 = WHITE, Font = Enum.Font.GothamBlack, TextXAlignment = RIGHT, ZIndex = 28 })
		label(root, { Size = UDim2.fromOffset(300, 22), Position = UDim2.fromOffset(352, 538), Text = "VOTE FOR THE TEAM SIZE", TextSize = 20, TextColor3 = COL.accent2, Font = Enum.Font.GothamBlack, TextXAlignment = LEFT, ZIndex = 28 })
		local mapRow = fr(root, 350, 332, 580, 92, WHITE, 1, 28)
		local modeRow = fr(root, 350, 454, 580, 76, WHITE, 1, 28)
		local sizeRow = fr(root, 350, 562, 580, 88, WHITE, 1, 28)
		local VB = { maps = {}, modes = {}, sizes = {} }
		local voteOn = false
		local MODE_CHIP = { TDM = "FIRST TO 50", FFA = "FIRST TO 25", GUN = "FINISH THE LADDER", FREE = "NO COMBAT", ["3"] = "6 PLAYERS  -  FAST", ["4"] = "8 PLAYERS  -  BALANCED", ["5"] = "10 PLAYERS  -  BIG FIGHT" }
		local MODE_COL = { TDM = RGB(255, 140, 30), FFA = RGB(255, 106, 10), GUN = RGB(40, 200, 230), FREE = RGB(110, 220, 120), ["3"] = RGB(40, 200, 230), ["4"] = RGB(255, 176, 40), ["5"] = RGB(255, 108, 10) }
		local function voteCard(parent, kind, id, w, h, title, accent, art)
			local b = new("TextButton", { Text = "", AutoButtonColor = false, Size = UDim2.fromOffset(w, h), BackgroundColor3 = WHITE, BorderSizePixel = 0, ClipsDescendants = true, ZIndex = 30 }, parent)
			new("UICorner", { CornerRadius = UDim.new(0, 16) }, b)
			local sh
			if art then
				b.BackgroundColor3 = RGB(9, 10, 10)
				local th = fr(b, 0, 0, 720, 430, RGB(0, 0, 0), 1, 30)
				new("UIScale", { Scale = w / 720 }, th)
				art(th)
				th.Position = UDim2.fromOffset(0, -6)
				sh = fr(b, 0, 0, w, h, RGB(6, 6, 6), 0, 40)
				new("UIGradient", { Rotation = 90, Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.85), NumberSequenceKeypoint.new(1, 0.1) }) }, sh)
			else
				new("UIGradient", { Rotation = 90, Color = ColorSequence.new(COL.panel2, COL.panel) }, b)
				fr(b, 0, 0, w, 6, accent, 0, 41)
			end
			local tl = label(b, { Size = UDim2.fromOffset(w - 12, 26), Position = UDim2.fromOffset(8, art and (h - 32) or 6), Text = title, TextSize = art and 22 or 22, Font = Enum.Font.GothamBlack,
				TextXAlignment = LEFT, ZIndex = 44, TextTruncate = Enum.TextTruncate.AtEnd })
			if not art then
				local ch = label(b, { Size = UDim2.fromOffset(w - 16, 20), Position = UDim2.fromOffset(8, 30), Text = MODE_CHIP[id] or "", TextSize = 11, Font = Enum.Font.GothamBold, TextColor3 = accent, TextXAlignment = LEFT, ZIndex = 44 })
			end
			local vp = label(b, { Size = UDim2.fromOffset(74, 20), Position = art and UDim2.fromOffset(w - 80, 8) or UDim2.fromOffset(8, h - 26), Text = "0 VOTES", TextSize = 12, Font = Enum.Font.GothamBlack, TextColor3 = INKC,
				BackgroundColor3 = COL.accent2, BackgroundTransparency = 0, ZIndex = 45 })
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, vp)
			local win = label(b, { Size = UDim2.fromOffset(86, 22), Position = UDim2.fromOffset(8, 8), Text = "WINNER", TextSize = 13, Font = Enum.Font.GothamBlack, TextColor3 = WHITE,
				BackgroundColor3 = COL.accent, BackgroundTransparency = 0, ZIndex = 46, Visible = false })
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, win)
			local dim = fr(b, 0, 0, w, h, RGB(8, 9, 9), 0.45, 47)
			local st = stroke(b, COL.accent2, 4, 1)
			b.MouseEnter:Connect(function()
				if kind == "map" then showMap(id, true) end
				if voteOn then uiHover() end
			end)
			b.Activated:Connect(function()
				if not voteOn then return end
				uiClick()
				Net.Vote:FireServer(kind, id)
				if kind == "map" then showMap(id, true) end
			end)
			return { btn = b, votes = vp, stroke = st, win = win, dim = dim }
		end
		local mapW = math.floor((580 - 2 * 14) / 3)
		for i, id in ipairs(MAP_ORDER) do
			local m = MAPS[id]
			local c = voteCard(mapRow, "map", id, mapW, 92, m.name, m.accent, ART[id])
			c.btn.Position = UDim2.fromOffset((i - 1) * (mapW + 14), 0)
			VB.maps[id] = c
		end
		local modeW = math.floor((580 - (#MODE_ORDER - 1) * 12) / #MODE_ORDER)
		for i, id in ipairs(MODE_ORDER) do
			local c = voteCard(modeRow, "mode", id, modeW, 76, MODES[id].name, MODE_COL[id] or COL.accent2, nil)
			c.btn.Position = UDim2.fromOffset((i - 1) * (modeW + 12), 0)
			VB.modes[id] = c
		end
		local sizeW = math.floor((580 - 2 * 12) / 3)
		for i, id in ipairs({ "3", "4", "5" }) do
			local c = voteCard(sizeRow, "size", id, sizeW, 88, id .. " v " .. id, MODE_COL[id], nil)
			c.btn.Position = UDim2.fromOffset((i - 1) * (sizeW + 12), 0)
			VB.sizes[id] = c
		end
		local function setVoteOn(on)
			voteOn = on
			XT.mmVoteOn = on
			for _, k in ipairs({ "maps", "modes", "sizes" }) do
				for _, c in pairs(VB[k]) do
					c.dim.Visible = not on and not c.win.Visible
				end
			end
		end
		setVoteOn(false)
		function XT.mmVote(d, show)
			XT.mmVoteOpen = show == true
			if not show or type(d) ~= "table" or type(d.maps) ~= "table" then
				setVoteOn(false)
				return
			end
			local res = d.result
			for i, id in ipairs(d.maps) do
				local c = VB.maps[id]
				if c then c.btn.Position = UDim2.fromOffset((i - 1) * (mapW + 14), 0) end
			end
			for i, id in ipairs(d.modes) do
				local c = VB.modes[id]
				if c then c.btn.Position = UDim2.fromOffset((i - 1) * (modeW + 12), 0) end
			end
			for _, kind in ipairs({ "maps", "modes", "sizes" }) do
				local k1 = (kind == "maps") and "map" or ((kind == "modes") and "mode" or "size")
				for id, c in pairs(VB[kind]) do
					local n = d.tally and d.tally[k1] and d.tally[k1][id] or 0
					c.votes.Text = n .. (n == 1 and " VOTE" or " VOTES")
					c.stroke.Transparency = (d.mine and d.mine[k1] == id) and 0 or 1
					c.win.Visible = res ~= nil and tostring(res[k1]) == id
				end
			end
			setVoteOn(d.open == true)
			local best, bestN = nil, -1
			for _, id in ipairs(d.maps) do
				local n = d.tally and d.tally.map and d.tally.map[id] or 0
				if n > bestN then best, bestN = id, n end
			end
			local pick = (d.mine and d.mine.map) or (bestN > 0 and best) or nil
			if res and res.map then pick = res.map end
			if pick and MAPS[pick] then showMap(pick, true) end
			XT.mmResult = res
			XT.mmHasPick = pick ~= nil
			if d.open and d.endsAt then XT.mmVoteLeft = math.max(0, math.ceil(d.endsAt - serverNow())) else XT.mmVoteLeft = nil end
		end

		-- ---------- YOUR SOLDIER (left) ----------
		local hero = card(root, 30, 100, 300, 548)
		local glow = fr(hero, 30, 60, 240, 240, COL.accent, 0.72, 28, 0.5)
		local youTag = label(hero, { Size = UDim2.fromOffset(56, 24), Position = UDim2.fromOffset(16, 14), Text = "YOU", TextSize = 18, Font = Enum.Font.GothamBlack, TextColor3 = INKC, BackgroundColor3 = COL.accent2, BackgroundTransparency = 0, ZIndex = 32 })
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, youTag)
		local hv = new("ViewportFrame", { Size = UDim2.fromOffset(300, 360), Position = UDim2.fromOffset(0, 20), BackgroundTransparency = 1, ZIndex = 30,
			Ambient = RGB(166, 182, 182), LightColor = RGB(255, 226, 200), LightDirection = Vector3.new(-1, -0.6, -0.9) }, hero)
		local hcam = new("Camera", { FieldOfView = 30 }, hv)
		hv.CurrentCamera = hcam
		fr(hero, 40, 372, 220, 8, RGB(0, 0, 0), 0.6, 29, 0.5)
		local opName = label(hero, { Size = UDim2.fromOffset(268, 42), Position = UDim2.fromOffset(16, 372), Text = "", TextSize = 38, Font = Enum.Font.GothamBlack, TextXAlignment = LEFT, ZIndex = 32 })
		local plName = label(hero, { Size = UDim2.fromOffset(200, 20), Position = UDim2.fromOffset(18, 414), Text = string.upper(player.DisplayName), TextSize = 15, TextColor3 = COL.dim, Font = Enum.Font.GothamBold, TextXAlignment = LEFT, ZIndex = 32, TextTruncate = Enum.TextTruncate.AtEnd })
		local lvChip = label(hero, { Size = UDim2.fromOffset(64, 24), AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromOffset(284, 414), Text = "LV 1", TextSize = 16, Font = Enum.Font.GothamBlack, TextColor3 = WHITE,
			BackgroundColor3 = COL.accent, BackgroundTransparency = 0, ZIndex = 32 })
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, lvChip)
		local function gunRow(y, tag)
			local r = fr(hero, 16, y, 268, 42, COL.panel2, 0.2, 30)
			new("UICorner", { CornerRadius = UDim.new(0, 12) }, r)
			label(r, { Size = UDim2.fromOffset(90, 42), Position = UDim2.fromOffset(12, 0), Text = tag, TextSize = 12, TextColor3 = COL.dim, Font = Enum.Font.GothamBold, TextXAlignment = LEFT, ZIndex = 32 })
			return label(r, { Size = UDim2.fromOffset(160, 42), AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromOffset(258, 0), Text = "", TextSize = 22, Font = Enum.Font.GothamBlack, TextXAlignment = RIGHT, ZIndex = 32, TextTruncate = Enum.TextTruncate.AtEnd })
		end
		local priV, secV = gunRow(452, "PRIMARY"), gunRow(498, "SECONDARY")
		local heroM, heroG
		local function buildHero()
			if heroM then heroM:Destroy() end
			if heroG then heroG:Destroy() end
			local id = (P.equip and P.equip.operator) or "OP_STANDARD"
			local op = OPERATORS[id] or OPERATORS.OP_STANDARD
			local pw = WEAPONS[(P.loadout and P.loadout.primary) or "VX44"] or WEAPONS.VX44
			local sw = WEAPONS[(P.loadout and P.loadout.secondary) or "KR9"] or WEAPONS.KR9
			heroM = XT.soldier(id)
			heroM.Parent = hv
			heroG = XT.gun(pw)
			heroG.Parent = hv
			opName.Text = op.name
			opName.TextColor3 = op.accent
			priV.Text = pw and pw.name or "-"
			secV.Text = sw and sw.name or "-"
			lvChip.Text = "LV " .. tostring(P.level or 1)
			plName.Text = string.upper(player.DisplayName)
		end
		RunService.RenderStepped:Connect(function()
			if not mm.Visible or not heroM then return end
			local t = os.clock()
			local cf = CFrame.new(0, 0.15 + math.sin(t * 1.6) * 0.04, 0) * CFrame.Angles(0, math.pi - 0.25 + math.sin(t * 0.7) * 0.3, 0) * CFrame.new(0, 3, 0)
			heroM:PivotTo(cf)
			heroG:PivotTo(cf * XT.gunHold)
			hcam.CFrame = CFrame.lookAt(Vector3.new(0, 4.3, 17), Vector3.new(0, 3.3, 0))
			glow.Size = UDim2.fromOffset(240 + math.sin(t * 2) * 10, 240 + math.sin(t * 2) * 10)
		end)

		-- ---------- SQUAD (right) ----------
		local sq = card(root, 950, 100, 300, 382)
		label(sq, { Size = UDim2.fromOffset(200, 30), Position = UDim2.fromOffset(18, 10), Text = "YOUR SQUAD", TextSize = 28, Font = Enum.Font.GothamBlack, TextXAlignment = LEFT, ZIndex = 30 })
		local sqCount = label(sq, { Size = UDim2.fromOffset(60, 24), AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromOffset(284, 13), Text = "1/4", TextSize = 16, Font = Enum.Font.GothamBlack, TextColor3 = INKC,
			BackgroundColor3 = COL.accent2, BackgroundTransparency = 0, ZIndex = 30 })
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, sqCount)
		local slots = {}
		for i = 1, 4 do
			local s = fr(sq, 14, 52 + (i - 1) * 80, 272, 72, COL.panel2, 0.15, 30)
			new("UICorner", { CornerRadius = UDim.new(0, 16) }, s)
			local st = stroke(s, COL.line, 2, 0.5)
			local av = new("ImageLabel", { Size = UDim2.fromOffset(54, 54), Position = UDim2.fromOffset(9, 9), BackgroundColor3 = COL.panel, BorderSizePixel = 0, Image = "", ZIndex = 32 }, s)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, av)
			local nm = label(s, { Size = UDim2.fromOffset(190, 28), Position = UDim2.fromOffset(72, 11), Text = "", TextSize = 22, Font = Enum.Font.GothamBlack, TextXAlignment = LEFT, ZIndex = 32, TextTruncate = Enum.TextTruncate.AtEnd })
			local tg = label(s, { Size = UDim2.fromOffset(190, 20), Position = UDim2.fromOffset(72, 40), Text = "", TextSize = 13, Font = Enum.Font.GothamBold, TextXAlignment = LEFT, ZIndex = 32, TextTruncate = Enum.TextTruncate.AtEnd })
			slots[i] = { f = s, st = st, av = av, nm = nm, tg = tg }
		end
		local thumbs, friendOf = {}, {}
		local function thumbFor(pl)
			if thumbs[pl.UserId] == nil then
				thumbs[pl.UserId] = false
				task.spawn(function()
					local ok, img = pcall(function() return Players:GetUserThumbnailAsync(pl.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size100x100) end)
					if ok and img then thumbs[pl.UserId] = img end
				end)
			end
			return thumbs[pl.UserId] or ""
		end
		local statQ, statO
		local function refreshSlots()
			local list = Players:GetPlayers()
			table.sort(list, function(a, b) return a == player or (b ~= player and a.Name < b.Name) end)
			local queued = 0
			for _, pl in ipairs(list) do if pl:GetAttribute("Queued") == true then queued += 1 end end
			for i, s in ipairs(slots) do
				local pl = list[i]
				if pl then
					if friendOf[pl] == nil and pl ~= player then
						friendOf[pl] = false
						task.spawn(function()
							local ok, r2 = pcall(function() return player:IsFriendsWith(pl.UserId) end)
							friendOf[pl] = ok and r2 == true
						end)
					end
					s.av.Image = thumbFor(pl)
					s.av.BackgroundTransparency = 0
					s.nm.Text = pl.DisplayName .. (pl == player and "  (YOU)" or "")
					s.nm.TextColor3 = pl == player and COL.accent2 or COL.text
					local q = pl:GetAttribute("Queued") == true
					s.tg.Text = (friendOf[pl] and "FRIEND  -  " or "") .. (q and "READY" or "IN MENU")
					s.tg.TextColor3 = q and COL.green or COL.dim
					s.st.Color = pl == player and COL.accent2 or COL.line
					s.st.Transparency = pl == player and 0.1 or 0.5
				else
					s.av.Image = ""
					s.av.BackgroundTransparency = 0.5
					s.nm.Text = "OPEN SLOT"
					s.nm.TextColor3 = COL.dim
					s.tg.Text = "Bots fill empty spots"
					s.tg.TextColor3 = COL.dim
					s.st.Color = COL.line
					s.st.Transparency = 0.75 + 0.2 * math.sin(os.clock() * 2 + i)
				end
			end
			sqCount.Text = math.min(#list, 4) .. "/4"
			statQ.Text = tostring(queued)
			statO.Text = tostring(#list)
		end

		-- invite
		local inviteNote = label(root, { Size = UDim2.fromOffset(300, 30), Position = UDim2.fromOffset(950, 550), Text = "Solo is fine: we slot you into an open seat. Bots only fill seats nobody else takes.", TextSize = 11, TextColor3 = COL.dim, TextXAlignment = LEFT, TextWrapped = true, TextYAlignment = Enum.TextYAlignment.Top, ZIndex = 28 })
		local function invite()
			task.spawn(function()
				local ok, can = pcall(function() return SocialService:CanSendGameInviteAsync(player) end)
				if ok and can then
					local ok2 = pcall(function() SocialService:PromptGameInvite(player) end)
					inviteNote.Text = ok2 and "Invite window opened. Friends who accept join this server and queue with you." or "Could not open the invite window."
				else
					inviteNote.Text = "Invites need the published game. Play it from Roblox to invite friends."
				end
				task.delay(6, function() inviteNote.Text = "Solo is fine: we slot you into an open seat. Bots only fill seats nobody else takes." end)
			end)
		end
		local inv = button(root, "INVITE FRIENDS (OPTIONAL)", UDim2.fromOffset(300, 52), UDim2.fromOffset(950, 494), COL.panel2, invite, 20)
		inv.Font = Enum.Font.GothamBlack
		inv.TextColor3 = COL.text
		inv.ZIndex = 30
		stroke(inv, WHITE, 2, 0.5)
		XT.mmInvite = invite
		local function tile(x, title)
			local t = card(root, x, 592, 144, 56)
			label(t, { Size = UDim2.fromOffset(60, 56), Position = UDim2.fromOffset(12, 0), Text = title, TextSize = 12, TextColor3 = COL.dim, Font = Enum.Font.GothamBold, TextXAlignment = LEFT, ZIndex = 30 })
			return label(t, { Size = UDim2.fromOffset(60, 56), Position = UDim2.fromOffset(72, 0), Text = "0", TextSize = 34, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent2, TextXAlignment = RIGHT, ZIndex = 30 })
		end
		statQ = tile(950, "IN QUEUE")
		statO = tile(1106, "ONLINE")

		-- ---------- TIP BAR ----------
		local tipBar = card(root, 30, 662, 800, 44)
		local tipTag = label(tipBar, { Size = UDim2.fromOffset(100, 28), Position = UDim2.fromOffset(10, 8), Text = "PRO TIP", TextSize = 18, Font = Enum.Font.GothamBlack, TextColor3 = WHITE, BackgroundColor3 = COL.accent, BackgroundTransparency = 0, ZIndex = 30 })
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, tipTag)
		local tipLbl = label(tipBar, { Size = UDim2.fromOffset(660, 44), Position = UDim2.fromOffset(126, 0), Text = "", TextSize = 16, TextColor3 = COL.text, Font = Enum.Font.GothamBold, TextXAlignment = LEFT, ZIndex = 30, TextTruncate = Enum.TextTruncate.AtEnd })

		XT.mapArt = ART
		for id, holder in pairs(XT.deployThumbs or {}) do
			if ART[id] then
				local th = fr(holder, 0, 0, 720, 430, RGB(0, 0, 0), 1, 1)
				new("UIScale", { Scale = holder.AbsoluteSize.X > 0 and holder.AbsoluteSize.X / 720 or 250 / 720 }, th)
				ART[id](th)
			end
		end

		-- ---------- driver ----------
		local startedAt, wasOn, lastCycle, cycleIx = 0, false, 0, 0
		local MM_TIPS = {
			"Headshots deal bonus damage. Aim for the head.",
			"Sprint, then tap Slide to break line of sight.",
			"Two kills earns your scorestreak: AIRSTRIKE or SUICIDE BOMBER. Swap it on the loadout screen.",
			"Bring friends: invited players join this server and queue with you.",
			"Throw a stim to heal and move faster.",
			"3 kills calls a UAV. 5 kills calls a SUPER UAV that sees through walls.",
			"Hold your aim button for pinpoint accuracy, but you move slower.",
			"Gun Game: every kill upgrades your weapon. Melee kills downgrade your victim.",
		}
		local function stepNow()
			local ph = phase()
			if ph == "COUNTDOWN" or ph == "MATCH" then return 3 end
			if XT.mmResult ~= nil then return 2 end
			return 1
		end
		task.spawn(function()
			while true do
				local on = player:GetAttribute("Queued") == true and not inMatch()
				if on and not wasOn then
					startedAt, lastCycle = os.clock(), 0
					shownMap = nil
					XT.mmResult = nil
					XT.mmHasPick = false
					pcall(buildHero)
				end
				wasOn = on
				mm.Visible = on
				if on then
					local now = os.clock()
					local el = math.floor(now - startedAt)
					local tl = (XT.mmVoteOpen and XT.mmVoteLeft) or nil
					local shown = tl or el
					mmTimer.Text = string.format("%02d:%02d", math.floor(shown / 60), shown % 60)
					timerCap.Text = tl and "MATCH STARTS IN" or "TIME IN LOBBY"
					local ph = phase()
					if ph ~= "VOTE" and ph ~= "COUNTDOWN" then XT.mmResult = nil end
					local cur = stepNow()
					for i, s in ipairs(steps) do
						local done, act = i < cur, i == cur
						s.f.BackgroundColor3 = act and COL.accent2 or (done and COL.accent or COL.panel2)
						s.f.BackgroundTransparency = act and 0 or (done and 0.1 or 0.35)
						s.t.TextColor3 = act and INKC or COL.text
						s.num.BackgroundColor3 = act and INKC or (done and WHITE or COL.panel)
						s.num.TextColor3 = act and COL.accent2 or (done and COL.accent or COL.dim)
						s.num.Text = done and "OK" or tostring(i)
						s.num.TextSize = done and 14 or 22
					end
					local dots = string.rep(".", math.floor(now * 2) % 4)
					local status = string.upper(tostring(State:GetAttribute("Status") or ""))
					local nq = 0
					for _, pl2 in ipairs(Players:GetPlayers()) do if pl2:GetAttribute("Queued") == true then nq += 1 end end
					if cur == 1 and XT.mmVoteOpen then
						mmStatus.Text = "SEARCHING FOR PLAYERS  -  " .. nq .. " IN LOBBY" .. dots
					elseif cur == 3 then
						mmStatus.Text = "MAP LOCKED IN - GET READY" .. dots
					else
						mmStatus.Text = (status ~= "" and status or "SEARCHING") .. dots
					end
					voteCount.Text = (cur == 1 and XT.mmVoteLeft) and (XT.mmVoteLeft .. "s") or (cur >= 2 and "LOCKED IN" or "VOTING OPENS SOON")
					voteCount.TextColor3 = cur == 1 and COL.accent2 or COL.dim
					local chosen = State:GetAttribute("MapId")
					if ph == "COUNTDOWN" and type(chosen) == "string" and MAPS[chosen] then
						showMap(chosen, true)
					elseif not XT.mmHasPick and XT.mmResult == nil and now - lastCycle > 4.5 then
						lastCycle = now
						cycleIx = cycleIx % #MAP_ORDER + 1
						shownMap = nil
						showMap(MAP_ORDER[cycleIx], false)
					end
					tipLbl.Text = MM_TIPS[(math.floor(el / 7) % #MM_TIPS) + 1]
					refreshSlots()
				end
				task.wait(on and 0.25 or 0.25)
			end
		end)
	end

	-- =================================================================
	-- SETTINGS (gameplay sliders, key + controller rebinding, mobile HUD)
	-- =================================================================
	local GuiService = game:GetService("GuiService")
	local sp = makePage("SETTINGS", "SETTINGS", "Changes save automatically to your account.")
	local tabBar = new("Frame", { Size = UDim2.fromOffset(780, 40), Position = UDim2.fromOffset(25, 90), BackgroundTransparency = 1 }, sp)
	new("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 8) }, tabBar)
	local tabFrames, tabBtns = {}, {}
	XT.tabNames = { "VIDEO", "GAME", "AUDIO", "MOUSE AND KEYBOARD", "CONTROLLER", "ACCESSIBILITY", "TOUCH" }
	local function showTab(name)
		for n, f in pairs(tabFrames) do f.Visible = (n == name) end
		for n, b in pairs(tabBtns) do
			b.BackgroundColor3 = (n == name) and COL.accent2 or COL.panel2
			b.TextColor3 = (n == name) and RGB(10, 10, 10) or COL.text
		end
	end
	for _, n in ipairs(XT.tabNames) do
		tabBtns[n] = button(tabBar, n, UDim2.fromOffset(math.floor(#n * 9 + 24), 38), UDim2.new(), COL.panel2, function() showTab(n) end, 13)
		local f = new("ScrollingFrame", { Size = UDim2.fromOffset(780, 470), Position = UDim2.fromOffset(25, 140), BackgroundTransparency = 1,
			BorderSizePixel = 0, ScrollBarThickness = 5, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, Visible = false }, sp)
		new("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }, f)
		new("UIPadding", { PaddingRight = UDim.new(0, 8) }, f)
		tabFrames[n] = f
	end

	-- global slider drag
	local dragFn
	UIS.InputChanged:Connect(function(input)
		if dragFn and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			local inset = GuiService:GetGuiInset()
			dragFn(Vector2.new(input.Position.X, input.Position.Y + inset.Y))
		end
	end)
	UIS.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			if dragFn then
				dragFn = nil
				saveSettings()
			end
		end
	end)

	local rowOrder = {}
	local function nextOrder(tab)
		rowOrder[tab] = (rowOrder[tab] or 0) + 1
		return rowOrder[tab]
	end
	local function rowFrame(tab, h)
		local r = new("Frame", { Size = UDim2.new(1, 0, 0, h or 44), BackgroundColor3 = COL.panel2, BorderSizePixel = 0, LayoutOrder = nextOrder(tab) },
			tabFrames[tab])
		corner(r, 8)
		return r
	end
	-- plain-English names and one-line explanations for every option (keyed by setting key)
	local SET_INFO = {
		fov = { "Field of View", "How wide your view is. Higher shows more at the sides but makes things look smaller." },
		bright = { "Brightness", "Makes the whole game lighter or darker." },
		quality = { "Graphics Quality", "0 is Auto, 1 is Low, 3 is High. Lower it if the game runs slowly." },
		showFps = { "Show Performance Stats", "Shows your frame rate and connection delay in the corner." },
		shake = { "Screen Shake", "How much the screen jolts when you fire a gun or get hit." },
		tilt = { "Camera Lean", "Tilts your view a little when you strafe or slide." },
		bob = { "Weapon Movement", "How much your gun swings and bobs while you move." },
		autoSprint = { "Auto Sprint", "Hold forward to run fast without holding a sprint key." },
		toggleAds = { "Tap to Aim", "On: tap once to aim and again to stop. Off: hold the button to aim." },
		hud = { "HUD Size", "Makes the on-screen display (health, ammo, score) bigger or smaller." },
		hitMark = { "Hit Markers", "Flashes an X on your crosshair when your shot lands." },
		feed = { "Kill Feed", "Shows who eliminated who in the top corner." },
		crossScale = { "Crosshair Size", "How big the aiming crosshair is." },
		crossAlpha = { "Crosshair Opacity", "Lower makes the crosshair see-through." },
		crossDot = { "Crosshair Centre Dot", "Adds a small dot in the middle of the crosshair." },
		master = { "Overall Volume", "Turns everything up or down at once." },
		musicVol = { "Music Volume", "How loud the background music is." },
		sfx = { "Game Sounds Volume", "How loud guns, footsteps, explosions and other game sounds are." },
		voiceVol = { "Announcer Voice Volume", "How loud the voice is that calls out kills and match events." },
		musicOn = { "Background Music", "Turn the music on or off." },
		voice = { "Announcer Voice", "The spoken voice for kills, streaks and match events. Off silences it." },
		hitSound = { "Hit Sounds", "Plays a short tick when you hit an enemy." },
		sens = { "X-Axis Sensitivity", "How fast the camera turns left and right when you move the mouse." },
		ySens = { "Y-Axis Sensitivity", "How fast the camera looks up and down when you move the mouse." },
		adsSens = { "Targeting Sensitivity", "Mouse speed while you are aiming down your sights." },
		scopeSens = { "Scope Sensitivity", "Mouse speed while you look through a sniper scope." },
		invertY = { "Invert Y-Axis", "Moving the mouse up looks down, like a flight stick." },
		padX = { "Look Sensitivity X-Axis", "How fast the right stick turns you left and right." },
		padY = { "Look Sensitivity Y-Axis", "How fast the right stick looks up and down." },
		padAds = { "Targeting Sensitivity", "Right stick speed while you are aiming down your sights." },
		padDead = { "Right Stick Dead Zone", "Ignores tiny stick movements. Raise it if the camera drifts by itself." },
		padInvert = { "Invert Y-Axis", "Pushing the stick up looks down." },
		colorblind = { "Colorblind Mode", "Changes the team colours so they are easier to tell apart." },
		mobScale = { "Button Size", "How big the on-screen touch buttons are." },
		mobOpacity = { "Button Opacity", "Lower makes the touch buttons see-through." },
		touchSens = { "Touch Look Sensitivity", "How fast swiping the screen turns the camera." },
	}
	local function addSlider(tab, text, key, lo, hi, fmt)
		local info = SET_INFO[key]
		if info then text = info[1] end
		local H = info and 66 or 50
		local r = rowFrame(tab, H)
		label(r, { Size = UDim2.fromOffset(260, info and 24 or 50), Position = UDim2.fromOffset(14, info and 6 or 0), Text = text, TextSize = 17, TextXAlignment = Enum.TextXAlignment.Left })
		if info then
			label(r, { Size = UDim2.fromOffset(262, 32), Position = UDim2.fromOffset(14, 30), Text = info[2], TextSize = 12, TextColor3 = COL.dim, Font = Enum.Font.Gotham,
				TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true })
		end
		local val = label(r, { Size = UDim2.fromOffset(70, H), Position = UDim2.new(1, -84, 0, 0), TextSize = 17, TextColor3 = COL.accent2,
			TextXAlignment = Enum.TextXAlignment.Right })
		local bg = new("Frame", { Size = UDim2.fromOffset(300, 8), Position = UDim2.fromOffset(290, H / 2 - 4), BackgroundColor3 = COL.bg, BorderSizePixel = 0 }, r)
		corner(bg, 4)
		local fill = new("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = COL.accent2, BorderSizePixel = 0 }, bg)
		corner(fill, 4)
		local knob = new("Frame", { Size = UDim2.fromOffset(18, 18), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0, 0.5),
			BackgroundColor3 = COL.text, BorderSizePixel = 0 }, bg)
		corner(knob, 9)
		local function refresh()
			local a = math.clamp((S.g[key] - lo) / (hi - lo), 0, 1)
			fill.Size = UDim2.fromScale(a, 1)
			knob.Position = UDim2.fromScale(a, 0.5)
			val.Text = string.format(fmt, S.g[key])
		end
		table.insert(settingsHooks, refresh)
		local hit = new("TextButton", { Size = UDim2.new(1, 20, 0, 40), Position = UDim2.fromOffset(-10, -16), BackgroundTransparency = 1, Text = "" }, bg)
		hit.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
				local function setFrom(pos)
					local a = math.clamp((pos.X - bg.AbsolutePosition.X) / math.max(1, bg.AbsoluteSize.X), 0, 1)
					S.g[key] = lo + (hi - lo) * a
					if key == "sens" then S.g.sensSet = true end
					refresh()
					settingsChanged()
				end
				dragFn = setFrom
				local inset = GuiService:GetGuiInset()
				setFrom(Vector2.new(input.Position.X, input.Position.Y + inset.Y))
			end
		end)
		refresh()
	end
	local function addToggle(tab, text, key)
		local info = SET_INFO[key]
		if info then text = info[1] end
		local H = info and 58 or 44
		local r = rowFrame(tab, H)
		label(r, { Size = UDim2.fromOffset(450, info and 24 or 44), Position = UDim2.fromOffset(14, info and 6 or 0), Text = text, TextSize = 17, TextXAlignment = Enum.TextXAlignment.Left })
		if info then
			label(r, { Size = UDim2.fromOffset(540, 20), Position = UDim2.fromOffset(14, 31), Text = info[2], TextSize = 12, TextColor3 = COL.dim, Font = Enum.Font.Gotham,
				TextXAlignment = Enum.TextXAlignment.Left, TextWrapped = true })
		end
		local b = button(r, "", UDim2.fromOffset(90, 30), UDim2.new(1, -104, 0.5, -15), COL.line, function()
			S.g[key] = not S.g[key]
			settingsChanged()
			saveSettings()
		end, 14)
		table.insert(settingsHooks, function()
			b.Text = S.g[key] and "ON" or "OFF"
			b.BackgroundColor3 = S.g[key] and COL.green or COL.line
			b.TextColor3 = S.g[key] and RGB(10, 10, 10) or COL.text
		end)
	end
	local function addCrosshairPicker(tab)
		local r = rowFrame(tab, 50)
		label(r, { Size = UDim2.fromOffset(250, 50), Position = UDim2.fromOffset(14, 0), Text = "Crosshair Colour", TextSize = 15, TextXAlignment = Enum.TextXAlignment.Left })
		local swatches = {}
		for i, c in ipairs(CROSS_COLORS) do
			local s = button(r, "", UDim2.fromOffset(34, 34), UDim2.fromOffset(270 + (i - 1) * 44, 8), c, function()
				S.g.crossColor = i
				settingsChanged()
				saveSettings()
			end)
			swatches[i] = stroke(s, COL.text, 3, 1)
		end
		table.insert(settingsHooks, function()
			for i, st in ipairs(swatches) do st.Transparency = (S.g.crossColor == i) and 0 or 1 end
		end)
	end
	local function addHeader(tab, text)
		local h = label(tabFrames[tab], { Size = UDim2.new(1, 0, 0, 26), Text = text, TextSize = 13, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left,
			LayoutOrder = nextOrder(tab) })
		return h
	end

	-- VIDEO
	addHeader("VIDEO", "DISPLAY")
	addSlider("VIDEO", "Field of View", "fov", 70, 110, "%d")
	addSlider("VIDEO", "Brightness", "bright", -0.1, 0.2, "%.2f")
	addSlider("VIDEO", "Graphics Quality  (0 = Auto, 1 = Low, 3 = High)", "quality", 0, 3, "%.0f")
	addToggle("VIDEO", "Show FPS + Ping", "showFps")
	addHeader("VIDEO", "CAMERA + MOTION")
	addSlider("VIDEO", "Screen Shake / Recoil Kick", "shake", 0, 1, "%.2f")
	addSlider("VIDEO", "Camera Tilt (strafe + slide)", "tilt", 0, 1, "%.2f")
	addSlider("VIDEO", "Weapon Sway / Bob", "bob", 0, 1.5, "%.2f")
	-- GAME
	addHeader("GAME", "GAMEPLAY")
	addToggle("GAME", "Sprint by Default (hold forward to sprint)", "autoSprint")
	addToggle("GAME", "Toggle Targeting (tap to aim instead of hold)", "toggleAds")
	addHeader("GAME", "HUD")
	addSlider("GAME", "HUD Scale", "hud", 0.7, 1.4, "%.2f")
	addToggle("GAME", "Hit Markers", "hitMark")
	addToggle("GAME", "Kill Feed", "feed")
	addHeader("GAME", "CROSSHAIR")
	addCrosshairPicker("GAME")
	addSlider("GAME", "Crosshair Size", "crossScale", 0.6, 2, "%.2f")
	addSlider("GAME", "Crosshair Opacity", "crossAlpha", 0.2, 1, "%.2f")
	addToggle("GAME", "Crosshair Centre Dot", "crossDot")
	addHeader("GAME", "RESET")
	local resetAll = button(tabFrames["GAME"], "RESET ALL SETTINGS TO DEFAULT", UDim2.fromOffset(300, 38), UDim2.new(), COL.red, function()
		S.g = copy(DEFAULT_G)
		S.keys = copy(DEFAULT_KEYS)
		S.pad = copy(DEFAULT_PAD)
		settingsChanged()
		saveSettings()
	end, 14)
	resetAll.LayoutOrder = nextOrder("GAME")
	-- AUDIO
	addHeader("AUDIO", "VOLUME")
	addSlider("AUDIO", "Master Volume", "master", 0, 1, "%.2f")
	addSlider("AUDIO", "Music Volume", "musicVol", 0, 1, "%.2f")
	addSlider("AUDIO", "Sound Effects Volume", "sfx", 0, 1, "%.2f")
	addSlider("AUDIO", "Announcer Volume", "voiceVol", 0, 1, "%.2f")
	addHeader("AUDIO", "OPTIONS")
	addToggle("AUDIO", "Music", "musicOn")
	addToggle("AUDIO", "Announcer Voice", "voice")
	addToggle("AUDIO", "Hit Sounds", "hitSound")
	-- MOUSE AND KEYBOARD
	addHeader("MOUSE AND KEYBOARD", "MOUSE SENSITIVITY")
	addSlider("MOUSE AND KEYBOARD", "X-Axis Sensitivity", "sens", 0.1, 3, "%.2f")
	addSlider("MOUSE AND KEYBOARD", "Y-Axis Sensitivity", "ySens", 0.1, 3, "%.2f")
	addSlider("MOUSE AND KEYBOARD", "Targeting Sensitivity", "adsSens", 0.1, 2, "%.2f")
	addSlider("MOUSE AND KEYBOARD", "Scope Sensitivity", "scopeSens", 0.1, 2, "%.2f")
	addToggle("MOUSE AND KEYBOARD", "Invert Y-Axis", "invertY")
	-- CONTROLLER
	addHeader("CONTROLLER", "CONTROLLER SENSITIVITY")
	addSlider("CONTROLLER", "Look Sensitivity X-Axis", "padX", 0.2, 3, "%.2f")
	addSlider("CONTROLLER", "Look Sensitivity Y-Axis", "padY", 0.2, 3, "%.2f")
	addSlider("CONTROLLER", "Targeting Sensitivity", "padAds", 0.2, 2, "%.2f")
	addSlider("CONTROLLER", "Right Stick Dead Zone", "padDead", 0, 0.5, "%.2f")
	addToggle("CONTROLLER", "Invert Y-Axis", "padInvert")
	-- ACCESSIBILITY
	addHeader("ACCESSIBILITY", "VISION")
	addToggle("ACCESSIBILITY", "Colorblind Mode (team colours)", "colorblind")
	addSlider("ACCESSIBILITY", "HUD Scale", "hud", 0.7, 1.4, "%.2f")
	addSlider("ACCESSIBILITY", "Crosshair Size", "crossScale", 0.6, 2, "%.2f")
	addSlider("ACCESSIBILITY", "Brightness", "bright", -0.1, 0.2, "%.2f")
	addHeader("ACCESSIBILITY", "MOTION")
	addSlider("ACCESSIBILITY", "Screen Shake / Recoil Kick", "shake", 0, 1, "%.2f")
	addSlider("ACCESSIBILITY", "Camera Tilt (strafe + slide)", "tilt", 0, 1, "%.2f")
	addSlider("ACCESSIBILITY", "Weapon Sway / Bob", "bob", 0, 1.5, "%.2f")
	addHeader("ACCESSIBILITY", "AUDIO")
	addToggle("ACCESSIBILITY", "Hit Sounds", "hitSound")
	addToggle("ACCESSIBILITY", "Announcer Voice", "voice")
	-- rebinding
	local capture -- { kind, id, btn }
	local function setCaptureText()
		for _, b in ipairs(UI.bindButtons or {}) do
			if capture and capture.btn == b.btn then
				b.btn.Text = "PRESS A KEY..."
				b.btn.BackgroundColor3 = COL.gold
				b.btn.TextColor3 = RGB(10, 10, 10)
			else
				b.btn.Text = prettyKey(S[b.kind][b.id] or "?")
				b.btn.BackgroundColor3 = COL.line
				b.btn.TextColor3 = COL.text
			end
		end
	end
	UI.bindButtons = {}
	local function addBindRows(tab, kind, actions, hint, groups)
		addHeader(tab, hint)
		for _, a in ipairs(actions) do
			if groups and groups[a.id] then addHeader(tab, groups[a.id]) end
			local r = rowFrame(tab, 40)
			label(r, { Size = UDim2.fromOffset(400, 40), Position = UDim2.fromOffset(14, 0), Text = a.label, TextSize = 15, TextXAlignment = Enum.TextXAlignment.Left })
			local b = button(r, "", UDim2.fromOffset(160, 30), UDim2.new(1, -174, 0.5, -15), COL.line, function(btn)
				capture = { kind = kind, id = a.id, btn = btn }
				setCaptureText()
			end, 14)
			table.insert(UI.bindButtons, { btn = b, kind = kind, id = a.id })
		end
		local rb = button(tabFrames[tab], "RESET TO DEFAULTS", UDim2.fromOffset(220, 38), UDim2.new(), COL.red, function()
			S[kind] = copy(kind == "keys" and DEFAULT_KEYS or DEFAULT_PAD)
			capture = nil
			setCaptureText()
			settingsChanged()
			saveSettings()
		end, 14)
		rb.LayoutOrder = nextOrder(tab)
	end
	addBindRows("MOUSE AND KEYBOARD", "keys", KEY_ACTIONS, "KEYBOARD BINDINGS  -  CLICK A BUTTON, THEN PRESS THE KEY OR MOUSE BUTTON YOU WANT  (ESC cancels)",
		{ MoveF = "MOVEMENT", Fire = "COMBAT", Scoreboard = "INTERFACE" })
	addBindRows("CONTROLLER", "pad", PAD_ACTIONS, "CONTROLLER BINDINGS  -  CLICK A BUTTON, THEN PRESS THE CONTROLLER BUTTON YOU WANT  (ESC / B cancels)",
		{ Jump = "MOVEMENT", Fire = "COMBAT", Scoreboard = "INTERFACE" })
	table.insert(settingsHooks, setCaptureText)

	UIS.InputBegan:Connect(function(input, processed)
		if not capture then return end
		local kind, id = capture.kind, capture.id
		local name
		if input.UserInputType == Enum.UserInputType.Keyboard then
			if input.KeyCode == Enum.KeyCode.Escape then
				capture = nil
				setCaptureText()
				return
			end
			if kind == "keys" and input.KeyCode ~= Enum.KeyCode.Unknown then name = input.KeyCode.Name end
		elseif input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.MouseButton2
			or input.UserInputType == Enum.UserInputType.MouseButton3 then
			if kind == "keys" then name = input.UserInputType.Name end
		elseif input.UserInputType == Enum.UserInputType.Gamepad1 then
			if kind == "pad" then
				if input.KeyCode == Enum.KeyCode.ButtonB and id ~= "Slide" then
					capture = nil
					setCaptureText()
					return
				end
				local n = input.KeyCode.Name
				if n:sub(1, 6) == "Button" then name = n end
			end
		end
		if not name then return end
		-- if another action uses this input, swap them so nothing is left unbound
		local old = S[kind][id]
		for otherId, v in pairs(S[kind]) do
			if v == name and otherId ~= id then S[kind][otherId] = old end
		end
		S[kind][id] = name
		capture = nil
		settingsChanged()
		saveSettings()
	end)

	-- ---------- mobile HUD tab + layout editor ----------
	addHeader("TOUCH", "TOUCH CONTROLS")
	addSlider("TOUCH", "Button Size", "mobScale", 0.6, 1.6, "%.2f")
	addSlider("TOUCH", "Button Opacity", "mobOpacity", 0.2, 1, "%.2f")
	addSlider("TOUCH", "Touch Look Sensitivity", "touchSens", 0.2, 3, "%.2f")
	local editorFrame = new("Frame", { Name = "LayoutEditor", Size = UDim2.fromScale(1, 1), BackgroundColor3 = COL.bg, BackgroundTransparency = 0.35,
		Visible = false, ZIndex = 50 }, gui)
	do
	label(editorFrame, { Size = UDim2.fromScale(1, 0), AutomaticSize = Enum.AutomaticSize.Y, Position = UDim2.fromOffset(0, 14),
		Text = "HUD LAYOUT EDITOR  -  drag any button to move it", TextSize = 22, ZIndex = 51 })
	local edButtons = {}
	local function layoutEditorButtons()
		local vp = camera.ViewportSize
		for _, def in ipairs(MOBILE_BUTTONS) do
			local b = edButtons[def.id]
			local p = S.mobile[def.id]
			local sz = def.size * S.g.mobScale
			b.Size = UDim2.fromOffset(sz, sz)
			b.Position = UDim2.fromOffset(p[1] * vp.X, p[2] * vp.Y)
			b.BackgroundTransparency = 1 - S.g.mobOpacity
		end
	end
	for _, def in ipairs(MOBILE_BUTTONS) do
		local b = new("TextButton", { AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = COL.panel2, Text = def.label, Font = Enum.Font.GothamBold,
			TextColor3 = COL.text, TextSize = 14, AutoButtonColor = false, ZIndex = 52 }, editorFrame)
		corner(b, 100)
		stroke(b, COL.accent, 2)
		edButtons[def.id] = b
		b.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
				dragFn = function(pos)
					local vp = camera.ViewportSize
					S.mobile[def.id] = { math.clamp(pos.X / vp.X, 0.03, 0.97), math.clamp(pos.Y / vp.Y, 0.05, 0.95) }
					layoutEditorButtons()
					settingsChanged()
				end
			end
		end)
	end
	local function addEditorSlider(text, key, lo, hi, yOff)
		local f = new("Frame", { Size = UDim2.fromOffset(330, 40), AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, yOff),
			BackgroundColor3 = COL.panel, ZIndex = 52 }, editorFrame)
		corner(f, 8)
		label(f, { Size = UDim2.fromOffset(110, 40), Position = UDim2.fromOffset(10, 0), Text = text, TextSize = 13, ZIndex = 53, TextXAlignment = Enum.TextXAlignment.Left })
		local bg = new("Frame", { Size = UDim2.fromOffset(190, 8), Position = UDim2.fromOffset(125, 16), BackgroundColor3 = COL.bg, BorderSizePixel = 0, ZIndex = 53 }, f)
		corner(bg, 4)
		local knob = new("Frame", { Size = UDim2.fromOffset(18, 18), AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = COL.accent2, BorderSizePixel = 0, ZIndex = 54 }, bg)
		corner(knob, 9)
		local function refresh() knob.Position = UDim2.fromScale(math.clamp((S.g[key] - lo) / (hi - lo), 0, 1), 0.5) end
		table.insert(settingsHooks, refresh)
		refresh()
		local hit = new("TextButton", { Size = UDim2.new(1, 20, 0, 40), Position = UDim2.fromOffset(-10, -16), BackgroundTransparency = 1, Text = "", ZIndex = 55 }, bg)
		hit.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
				dragFn = function(pos)
					local a = math.clamp((pos.X - bg.AbsolutePosition.X) / math.max(1, bg.AbsoluteSize.X), 0, 1)
					S.g[key] = lo + (hi - lo) * a
					refresh()
					layoutEditorButtons()
					settingsChanged()
				end
			end
		end)
	end
	addEditorSlider("Size", "mobScale", 0.6, 1.6, 56)
	addEditorSlider("Opacity", "mobOpacity", 0.2, 1, 104)
	local edDone = button(editorFrame, "SAVE + CLOSE", UDim2.fromOffset(180, 46), UDim2.new(0.5, -190, 1, -70), COL.green, function()
		editorFrame.Visible = false
		settingsChanged()
		saveSettings()
	end, 16)
	edDone.ZIndex = 52
	edDone.TextColor3 = RGB(10, 10, 10)
	XT.edReset = button(editorFrame, "RESET LAYOUT", UDim2.fromOffset(180, 46), UDim2.new(0.5, 10, 1, -70), COL.red, function()
		S.mobile = copy(DEFAULT_MOBILE)
		S.g.mobScale, S.g.mobOpacity = DEFAULT_G.mobScale, DEFAULT_G.mobOpacity
		layoutEditorButtons()
		settingsChanged()
	end, 16)
	XT.edReset.ZIndex = 52
	table.insert(settingsHooks, layoutEditorButtons)
	local openEdBtn = button(tabFrames["TOUCH"], "OPEN HUD LAYOUT EDITOR  (drag buttons where you want them)", UDim2.new(1, 0, 0, 56), UDim2.new(), COL.accent, function()
		layoutEditorButtons()
		editorFrame.Visible = true
	end, 16)
	openEdBtn.LayoutOrder = nextOrder("TOUCH")
	openEdBtn.TextColor3 = RGB(10, 10, 10)
	label(tabFrames["TOUCH"], { Size = UDim2.new(1, 0, 0, 60), Text = "These buttons only show on touch devices. You can open the editor on any device to set up your layout, and it follows your account.",
		TextSize = 13, TextColor3 = COL.dim, Font = Enum.Font.Gotham, TextXAlignment = Enum.TextXAlignment.Left, TextWrapped = true, LayoutOrder = nextOrder("TOUCH") })
	end

	showTab("VIDEO")
	settingsChanged()

	XT.purple = false
	-- =================================================================
	-- MENU POLISH: top + bottom bars, entrance animation, glow, sweep, pulsing deploy button
	-- =================================================================
	do
		local LEFT, RIGHT = Enum.TextXAlignment.Left, Enum.TextXAlignment.Right
		local top = menuHolder:WaitForChild("Nav")
		local bot = new("Frame", { Size = UDim2.new(1, 0, 0, 34), Position = UDim2.new(0, 0, 1, -34), BackgroundColor3 = RGB(5, 7, 11), BackgroundTransparency = 0.2,
			BorderSizePixel = 0, ZIndex = 3, Visible = false }, menuHolder)
		local hint = label(bot, { Size = UDim2.new(1, -40, 1, 0), Position = UDim2.fromOffset(20, 0), Text = "", TextSize = 12, TextColor3 = COL.dim, TextXAlignment = LEFT,
			Font = Enum.Font.GothamMedium, ZIndex = 4 })
		table.insert(settingsHooks, function()
			hint.Text = string.format("[%s] MENU      [%s] SCOREBOARD      [%s] FIRE      [%s] AIM      [%s] RELOAD      [%s] SLIDE      -  REBIND ANYTHING IN SETTINGS",
				prettyKey(S.keys.Menu), prettyKey(S.keys.Scoreboard), prettyKey(S.keys.Fire), prettyKey(S.keys.ADS), prettyKey(S.keys.Reload), prettyKey(S.keys.Slide))
		end)
		-- glowing title
		for _, c in ipairs(side:GetChildren()) do
			if c:IsA("TextLabel") and c.Text == "BLACKOUT" then
				c.Font = Enum.Font.GothamBlack
				new("UIStroke", { Color = COL.accent, Thickness = 1.5, Transparency = 0.6 }, c)
			end
		end
		-- corner brackets on the content panel
		for _, cn in ipairs({ { 0, 0 }, { 1, 0 }, { 0, 1 }, { 1, 1 } }) do
			new("Frame", { Size = UDim2.fromOffset(26, 3), AnchorPoint = Vector2.new(cn[1], cn[2]), Position = UDim2.fromScale(cn[1], cn[2]), BackgroundColor3 = COL.accent, BorderSizePixel = 0, ZIndex = 5 }, content)
			new("Frame", { Size = UDim2.fromOffset(3, 26), AnchorPoint = Vector2.new(cn[1], cn[2]), Position = UDim2.fromScale(cn[1], cn[2]), BackgroundColor3 = COL.accent, BorderSizePixel = 0, ZIndex = 5 }, content)
		end
		-- subtle vertical shading on the mode / map cards
		for _, pg in ipairs({ dep }) do
			for _, c in ipairs(pg:GetChildren()) do
				if c:IsA("Frame") and c.Size.X.Offset == 250 then
					new("UIGradient", { Rotation = 90, Color = ColorSequence.new(RGB(255, 255, 255), RGB(140, 146, 160)) }, c)
					stroke(c, COL.line, 1, 0.5)
				end
			end
		end
		-- animated light sweep across the content panel
		local sweep = new("Frame", { Size = UDim2.fromOffset(180, 650), Position = UDim2.fromOffset(-200, 0), BackgroundColor3 = COL.accent, BorderSizePixel = 0, ZIndex = 1 }, content)
		new("UIGradient", { Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.5, 0.93), NumberSequenceKeypoint.new(1, 1) }) }, sweep)
		TweenService:Create(sweep, TweenInfo.new(8, Enum.EasingStyle.Linear, Enum.EasingDirection.Out, -1, false, 1.5), { Position = UDim2.fromOffset(860, 0) }):Play()
		-- pulsing deploy button glow
		local qs = stroke(queueBtn, COL.accent, 3, 0.3)
		TweenService:Create(qs, TweenInfo.new(1.1, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), { Transparency = 0.9 }):Play()
		-- entrance animation whenever the menu opens
		local function intro()
			content.Position = UDim2.fromOffset(40, 0)
			top.Position = UDim2.fromOffset(0, -64)
			bot.Position = UDim2.new(0, 0, 1, 0)
			tween(content, 0.5, { Position = UDim2.fromOffset(0, 0) }, Enum.EasingStyle.Quint)
			tween(top, 0.5, { Position = UDim2.fromOffset(0, 0) }, Enum.EasingStyle.Quint)
			tween(bot, 0.5, { Position = UDim2.new(0, 0, 1, -34) }, Enum.EasingStyle.Quint)
		end
		menuHolder:GetPropertyChangedSignal("Visible"):Connect(function()
			if menuHolder.Visible then intro() end
		end)
		-- page slide-in
		for _, pg in pairs(UI.pages) do
			pg:GetPropertyChangedSignal("Visible"):Connect(function()
				if pg.Visible then
					pg.Position = UDim2.fromOffset(26, 0)
					tween(pg, 0.35, { Position = UDim2.new() }, Enum.EasingStyle.Quint)
				end
			end)
		end
		intro()
	end

	-- =================================================================
	-- IN-MATCH HUD  (layout zones never overlap:
	--   top-centre = score/timer, top-right = kill feed, below score = banner,
	--   bottom-left = health, bottom-right = ammo, centre = crosshair)
	-- =================================================================
	UI.touch = UIS.TouchEnabled and not UIS.KeyboardEnabled
	local HUD = { ammoText = "", scoreA = 0, scoreB = 0, limit = 0, rows = {}, feed = {}, health = 100, spread = 20, deadUntil = 0, results = nil }
	local hud = new("Frame", { Name = "HUD", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, ZIndex = 1 }, gui)
	MS.hudScale = new("UIScale", { Scale = 0.84 }, hud)
	function MS.fitHud()
		local sc = math.max(0.1, MS.hudScale.Scale)
		hud.Size = UDim2.fromScale(1 / sc, 1 / sc) -- UIScale shrinks the frame, so grow it back to fill the real screen
	end
	MS.hudScale.Scale = ((S and S.g and S.g.hud) or 1) * 0.84
	MS.fitHud()

	-- score bar (top centre)
	local scoreBar = new("Frame", { Size = UDim2.fromOffset(380, 62), Position = UDim2.new(0.5, -190, 0, 22), BackgroundColor3 = RGB(8, 10, 10), BackgroundTransparency = 0.2,
		BorderSizePixel = 0 }, hud)
	corner(scoreBar, 4)
	stroke(scoreBar, COL.line, 1, 0.4)
	new("Frame", { Size = UDim2.new(0, 2, 0.6, 0), Position = UDim2.new(0, 126, 0.2, 0), BackgroundColor3 = COL.line, BorderSizePixel = 0 }, scoreBar)
	new("Frame", { Size = UDim2.new(0, 2, 0.6, 0), Position = UDim2.new(0, 252, 0.2, 0), BackgroundColor3 = COL.line, BorderSizePixel = 0 }, scoreBar)
	HUD.scoreA = label(scoreBar, { Size = UDim2.fromOffset(126, 38), Position = UDim2.fromOffset(0, 2), Text = "0", TextSize = 32, Font = Enum.Font.GothamBlack, TextColor3 = COL.text })
	HUD.scoreB = label(scoreBar, { Size = UDim2.fromOffset(126, 38), Position = UDim2.fromOffset(254, 2), Text = "0", TextSize = 32, Font = Enum.Font.GothamBlack, TextColor3 = COL.red })
	HUD.killsA = label(scoreBar, { Size = UDim2.fromOffset(126, 16), Position = UDim2.fromOffset(0, 40), Text = "", TextSize = 11, TextColor3 = COL.dim, Font = Enum.Font.GothamBold })
	HUD.killsB = label(scoreBar, { Size = UDim2.fromOffset(126, 16), Position = UDim2.fromOffset(254, 40), Text = "", TextSize = 11, TextColor3 = COL.dim, Font = Enum.Font.GothamBold })
	HUD.timer = label(scoreBar, { Size = UDim2.fromOffset(126, 32), Position = UDim2.fromOffset(127, 6), Text = "0:00", TextSize = 24, Font = Enum.Font.GothamBlack })
	HUD.modeTxt = label(scoreBar, { Size = UDim2.fromOffset(126, 18), Position = UDim2.fromOffset(127, 38), Text = "", TextSize = 10, TextColor3 = COL.dim })
	-- both squads, shown beside the score: your team on the left, the enemy (red) on the right
	HUD.rosterL = new("Frame", { Size = UDim2.fromOffset(330, 62), AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(0.5, -202, 0, 22), BackgroundTransparency = 1 }, hud)
	HUD.rosterR = new("Frame", { Size = UDim2.fromOffset(330, 62), Position = UDim2.new(0.5, 202, 0, 22), BackgroundTransparency = 1 }, hud)
	new("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 8), HorizontalAlignment = Enum.HorizontalAlignment.Right,
		VerticalAlignment = Enum.VerticalAlignment.Top, SortOrder = Enum.SortOrder.LayoutOrder }, HUD.rosterL)
	new("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 8), HorizontalAlignment = Enum.HorizontalAlignment.Left,
		VerticalAlignment = Enum.VerticalAlignment.Top, SortOrder = Enum.SortOrder.LayoutOrder }, HUD.rosterR)
	HUD.rosterSig = ""
	function HUD.updateRoster(rows)
		local mode = MODES[State:GetAttribute("ModeId") or ""]
		local myTeam = player:GetAttribute("Team")
		if not (mode and mode.teams) or not myTeam or myTeam == "" then
			HUD.rosterL.Visible, HUD.rosterR.Visible = false, false
			HUD.killsA.Text, HUD.killsB.Text = "", ""
			HUD.rosterSig = ""
			return
		end
		HUD.rosterL.Visible, HUD.rosterR.Visible = true, true
		local mates, foes = {}, {}
		for _, r in ipairs(rows) do
			if r.t == myTeam then table.insert(mates, r) elseif r.t and r.t ~= "" then table.insert(foes, r) end
		end
		local sig = myTeam
		for _, r in ipairs(mates) do sig ..= "|" .. r.n .. (r.o or "") .. tostring(r.u or "") end
		sig ..= "#"
		for _, r in ipairs(foes) do sig ..= "|" .. r.n .. (r.o or "") .. tostring(r.u or "") end
		local kA, kB = 0, 0
		for _, r in ipairs(mates) do kA += r.k or 0 end
		for _, r in ipairs(foes) do kB += r.k or 0 end
		HUD.killsA.Text = kA .. " KILLS"
		HUD.killsB.Text = kB .. " KILLS"
		if sig ~= HUD.rosterSig then
			HUD.rosterSig = sig
			HUD.rosterCards = {}
			for _, fr in ipairs({ HUD.rosterL, HUD.rosterR }) do
				for _, c in ipairs(fr:GetChildren()) do if c:IsA("Frame") then c:Destroy() end end
			end
			local function build(list, frame, enemy, flip)
				for i, r in ipairs(list) do
					if i > 6 then break end
					local card = new("Frame", { Size = UDim2.fromOffset(46, 62), BackgroundTransparency = 1, LayoutOrder = flip and (100 - i) or i }, frame)
					local port = new("Frame", { Size = UDim2.fromOffset(46, 46), BackgroundColor3 = RGB(14, 16, 16), BorderSizePixel = 0, ClipsDescendants = true }, card)
					corner(port, 4)
					local st = stroke(port, enemy and COL.red or RGB(230, 236, 245), 2, enemy and 0.05 or 0.35)
					local face
					if r.u then
						face = new("ImageLabel", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(30, 34, 36), BorderSizePixel = 0, ScaleType = Enum.ScaleType.Crop }, port)
						task.spawn(function()
							local ok, img = pcall(function() return Players:GetUserThumbnailAsync(r.u, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size100x100) end)
							if ok and img and face.Parent then face.Image = img end
						end)
					else
						local bust = new("ViewportFrame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(44, 48, 50), BackgroundTransparency = 0, Ambient = RGB(255, 255, 255), LightColor = RGB(255, 255, 255),
							LightDirection = Vector3.new(0.3, -0.4, -1) }, port)
						local bcam = new("Camera", { FieldOfView = 20 }, bust)
						bust.CurrentCamera = bcam
						bcam.CFrame = CFrame.lookAt(Vector3.new(0, 4.85, 5.4), Vector3.new(0, 4.6, 0))
						pcall(function() XT.soldier(r.o or "OP_STANDARD").Parent = bust end)
					end
					local nm = label(card, { Size = UDim2.fromOffset(52, 12), Position = UDim2.fromOffset(-3, 47), Text = string.upper(string.sub(tostring(r.n), 1, 8)), TextSize = 8,
						TextColor3 = enemy and COL.red or COL.text, Font = Enum.Font.GothamBold })
					local kl = label(port, { Size = UDim2.fromOffset(18, 14), AnchorPoint = Vector2.new(1, 1), Position = UDim2.fromScale(1, 1), Text = "0", TextSize = 11,
						Font = Enum.Font.GothamBlack, BackgroundColor3 = RGB(0, 0, 0), BackgroundTransparency = 0.35, ZIndex = 7 })
					HUD.rosterCards[r.n] = { kl = kl, st = st }
				end
			end
			build(mates, HUD.rosterL, false, true)
			build(foes, HUD.rosterR, true, false)
		end
		for _, r in ipairs(rows) do
			local c = HUD.rosterCards and HUD.rosterCards[r.n]
			if c then
				c.kl.Text = tostring(r.k or 0)
			end
		end
	end

	-- banner (below score bar)
	HUD.banner = label(hud, { Size = UDim2.fromOffset(600, 44), Position = UDim2.new(0.5, -300, 0, 84), Text = "", TextSize = 30, TextTransparency = 1,
		TextStrokeTransparency = 1, TextStrokeColor3 = RGB(0, 0, 0) })
	local bannerSeq = 0
	local function showBanner(text, kind)
		bannerSeq += 1
		local my = bannerSeq
		HUD.banner.Text = text
		HUD.banner.TextColor3 = (kind == "kill" and COL.red) or (kind == "streak" and COL.gold) or (kind == "warn" and COL.orange) or (kind == "good" and COL.green) or COL.text
		HUD.banner.TextTransparency = 0
		HUD.banner.TextStrokeTransparency = 0.4
		task.delay(2.2, function()
			if bannerSeq == my then
				tween(HUD.banner, 0.6, { TextTransparency = 1, TextStrokeTransparency = 1 })
			end
		end)
	end
	XT.banner = showBanner
	Net.Banner.OnClientEvent:Connect(function(text, kind)
		if type(text) == "string" then showBanner(text, kind) end
	end)

	-- ---------- announcer: kill callouts, multi-kills, match events ----------
	do
		local callout = label(hud, { Size = UDim2.fromOffset(700, 44), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.64), Text = "", TextSize = 30,
			TextTransparency = 1, TextStrokeTransparency = 1, TextStrokeColor3 = RGB(0, 0, 0), Font = Enum.Font.GothamBlack })
		local seq = 0
		local KILL_LINES = { "MAN DOWN", "TARGET ELIMINATED", "HOSTILE DOWN", "ENEMY NEUTRALISED", "CONFIRMED KILL", "TANGO DOWN", "ONE LESS", "CLEAN SHOT" }
		local function stinger(kind)
			local v = S.g.sfx
			if kind == "KILL" then
				playSound(SND.ping, 0.7 + math.random() * 0.4, 0.45 * v)
				playSound(SND.bass, 1.3, 0.35 * v)
			elseif kind == "DOUBLE" or kind == "TRIPLE" or kind == "MULTI" then
				local n = (kind == "DOUBLE") and 2 or (kind == "TRIPLE" and 3 or 4)
				for i = 1, n do task.delay((i - 1) * 0.09, function() playSound(SND.ping, 0.9 + i * 0.22, 0.45 * v) end) end
				playSound(SND.bass, 0.8, 0.5 * v)
			elseif kind == "HEADSHOT" then
				playSound(SND.snap, 1.6, 0.5 * v)
				playSound(SND.ping, 1.8, 0.35 * v)
			elseif kind == "UAV" then
				for i = 1, 3 do task.delay((i - 1) * 0.16, function() playSound(SND.click2, 1.8 + i * 0.2, 0.4 * v) end) end
			elseif kind == "LOWHP" or kind == "DOWN" then
				playSound(SND.bass, 0.6, 0.55 * v)
			else
				playSound(SND.ping, 1.1, 0.4 * v)
				playSound(SND.flash, 1.2, 0.25 * v)
			end
		end
		-- spoken announcer via Roblox text-to-speech (AudioTextToSpeech); falls back to stingers if unavailable
		local ttsBroken = false
		local function speak(text, pitch, voiceId, emitFrom)
			if ttsBroken then return false end
			local ok = pcall(function()
				local tts = Instance.new("AudioTextToSpeech")
				pcall(function() tts.Text = string.lower(text) end)
				pcall(function() tts.Pitch = pitch or -3 end)
				if voiceId then pcall(function() tts.VoiceId = voiceId end) end
				pcall(function() tts.Speed = 1.12 end)
				local fader = Instance.new("AudioFader")
				fader.Volume = (voiceId and 3 or 1.6) * S.g.sfx * S.g.voiceVol * S.g.master
				local eq = Instance.new("AudioEqualizer") -- slight radio/comms colouring
				pcall(function() eq.LowGain = -6; eq.MidGain = 4; eq.HighGain = -2 end)
				local out = Instance.new("AudioDeviceOutput")
				local emitter
				if emitFrom and emitFrom.Parent then
					-- 3D voice: comes out of the character itself (needs a listener riding on the camera)
					emitter = Instance.new("AudioEmitter")
					emitter.Parent = emitFrom
					if not (camera:FindFirstChild("BO_VoiceListener")) then
						local lis = Instance.new("AudioListener")
						lis.Name = "BO_VoiceListener"
						lis.Parent = camera
						local lo = Instance.new("AudioDeviceOutput")
						lo.Name = "BO_VoiceListenerOut"
						lo.Parent = lis
						local lw = Instance.new("Wire")
						lw.SourceInstance, lw.TargetInstance = lis, lo
						lw.Parent = lo
					end
					Debris:AddItem(emitter, 7)
				end
				tts.Parent = SoundService
				fader.Parent = SoundService
				eq.Parent = SoundService
				out.Parent = SoundService
				local function wire(a, b)
					local w = Instance.new("Wire")
					w.SourceInstance, w.TargetInstance = a, b
					w.Parent = a
				end
				wire(tts, eq)
				wire(eq, fader)
				if emitter then wire(fader, emitter) else wire(fader, out) end
				tts:Play()
				for _, o in ipairs({ tts, fader, eq, out }) do Debris:AddItem(o, 7) end
			end)
			if not ok then ttsBroken = true end
			return ok
		end
		HUD.speak = speak
		local function announce(kind, text, color, say)
			seq += 1
			local my = seq
			local ids = ANNOUNCER_IDS[kind]
			if ids and #ids > 0 then
				playSound(ids[math.random(1, #ids)], 1, 0.9 * S.g.sfx)
			else
				stinger(kind)
				if S.g.voice then if kind == "STRIKE" then
					local ch = player.Character
					speak(say or text, -1, "1", ch and ch:FindFirstChild("HumanoidRootPart")) -- VoiceId 1 = British male, spoken from the player
				else
					speak(say or text)
				end end
			end
			-- voice + sound only: no on-screen kill text
		end
		local lastKill, chain, lastHead = 0, 0, 0
		Net.Hit.OnClientEvent:Connect(function(kind)
			local now = os.clock()
			if kind == "head" then lastHead = now end
			if kind ~= "kill" then return end
			chain = (now - lastKill < 4) and (chain + 1) or 1
			lastKill = now
			if chain == 2 then announce("DOUBLE", "DOUBLE KILL", COL.gold)
			elseif chain == 3 then announce("TRIPLE", "TRIPLE KILL", COL.orange)
			elseif chain >= 4 then announce("MULTI", "MULTI KILL x" .. chain, COL.red, "multi kill")
			elseif now - lastHead < 0.4 then announce("HEADSHOT", "HEADSHOT - MAN DOWN", COL.gold, "headshot. man down")
			else announce("KILL", KILL_LINES[math.random(1, #KILL_LINES)], COL.red) end
		end)
		Net.Banner.OnClientEvent:Connect(function(text)
			if type(text) == "string" and string.find(text, "UAV") then announce("UAV", text, COL.green)
			end
		end)
		Net.Died.OnClientEvent:Connect(function() announce("DOWN", "YOU WERE ELIMINATED", COL.red) end)
		State:GetAttributeChangedSignal("Phase"):Connect(function()
			local ph = phase()
			if ph == "MATCH" and inMatch() then announce("START", "ENGAGE", COL.orange) end
		end)
		local warned1, warned2, lowWarned = false, false, false
		task.spawn(function()
			while true do
				task.wait(0.5)
				if inMatch() and phase() == "MATCH" then
					local left = (State:GetAttribute("EndsAt") or 0) - serverNow()
					if left > 0 and left < 61 and not warned1 then warned1 = true; announce("MIN1", "ONE MINUTE REMAINING", COL.gold) end
					if left > 0 and left < 31 and not warned2 then warned2 = true; announce("SEC30", "30 SECONDS", COL.orange) end
					local h = myHum()
					if h and h.Health > 0 then
						if h.Health < 30 and not lowWarned then lowWarned = true; announce("LOWHP", "TAKING HEAVY DAMAGE", COL.red)
						elseif h.Health > 60 then lowWarned = false end
					end
				else
					warned1, warned2, lowWarned = false, false, false
				end
			end
		end)
	end

	-- kill feed (top right)
	HUD.feedBox = new("Frame", { Size = UDim2.fromOffset(460, 230), Position = UDim2.fromOffset(32, 96), BackgroundTransparency = 1 }, hud)
	new("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder, VerticalAlignment = Enum.VerticalAlignment.Top,
		HorizontalAlignment = Enum.HorizontalAlignment.Left }, HUD.feedBox)
	local feedN = 0
	-- settings that touch the HUD (feed, FPS readout, brightness)
	XT.applyHudSettings = function()
		ensureFx()
		lightObjs.cc.Brightness = S.g.bright
		if XT.sfxGroup then XT.sfxGroup.Volume = S.g.master end
		HUD.feedBox.Visible = S.g.feed
		if HUD.fps then HUD.fps.Visible = S.g.showFps end
	end
	table.insert(settingsHooks, XT.applyHudSettings)
	-- FPS + ping readout
	HUD.fps = label(hud, { Size = UDim2.fromOffset(80, 18), AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -36, 1, -6), Text = "", TextSize = 12,
		TextColor3 = COL.green, TextXAlignment = Enum.TextXAlignment.Right, BackgroundTransparency = 1, Visible = true })
	do
		local n = 0
		RunService.Heartbeat:Connect(function() n += 1 end)
		task.spawn(function()
			while true do
				task.wait(0.5)
				local fps = n * 2
				n = 0
				if HUD.fps and HUD.fps.Visible then
					local ping = 0
					pcall(function() ping = math.floor(player:GetNetworkPing() * 2000) end)
					HUD.fps.Text = string.format("%d ms", ping)
					HUD.fps.TextColor3 = ping < 80 and COL.green or (ping < 160 and COL.accent2 or COL.red)
				end
			end
		end)
	end

	XT.applyHudSettings()
	-- centre-screen kill confirm + multikill medals
	local kcBox = new("Frame", { Size = UDim2.fromOffset(420, 90), AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0.6, 0), BackgroundTransparency = 1, ZIndex = 20 }, hud)
	local kcMain = label(kcBox, { Size = UDim2.fromOffset(420, 26), Text = "", TextSize = 19, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent2, TextTransparency = 1, TextStrokeTransparency = 0.5, ZIndex = 21 })
	local kcSub = label(kcBox, { Size = UDim2.fromOffset(420, 18), Position = UDim2.fromOffset(0, 26), Text = "", TextSize = 13, Font = Enum.Font.GothamBold, TextColor3 = COL.text, TextTransparency = 1, ZIndex = 21 })
	local kcMedal = label(kcBox, { Size = UDim2.fromOffset(420, 34), Position = UDim2.fromOffset(0, 48), Text = "", TextSize = 28, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent, TextTransparency = 1, ZIndex = 21 })
	local kcScale = new("UIScale", { Scale = 1 }, kcBox)
	local kcStamp, kcChain, kcStreak, kcToken = 0, 0, 0, 0
	local kcStreakRef
	kcStreakRef = function() return kcStreak end
	local MEDALS = { [2] = "DOUBLE KILL", [3] = "TRIPLE KILL", [4] = "QUAD FEED", [5] = "OVERKILL" }
	local STREAKS = { [3] = "3 KILL STREAK", [5] = "5 KILL STREAK", [7] = "7 KILL STREAK", [10] = "UNSTOPPABLE" }
	local function killConfirm(victimName, headshot, wname)
		local now = os.clock()
		kcChain = (now - kcStamp < 4.5) and (kcChain + 1) or 1
		kcStamp = now
		kcStreak += 1
		kcToken += 1
		local tk = kcToken
		kcMain.Text = "ELIMINATED  " .. string.upper(tostring(victimName))
		kcSub.Text = (headshot and "HEADSHOT   " or "") .. string.upper(tostring(wname or "")) .. "   +" .. (headshot and 150 or 100)
		kcMedal.Text = MEDALS[math.min(kcChain, 5)] or STREAKS[kcStreak] or ""
		if kcChain < 2 and STREAKS[kcStreak] then kcMedal.Text = STREAKS[kcStreak] end
		kcMain.TextColor3 = headshot and RGB(255, 90, 70) or COL.accent2
		kcScale.Scale = 1.25
		for _, l in ipairs({ kcMain, kcSub, kcMedal }) do l.TextTransparency = 0 end
		tween(kcScale, 0.15, { Scale = 1 }, Enum.EasingStyle.Back)
		task.delay(2.2, function()
			if tk ~= kcToken then return end
			for _, l in ipairs({ kcMain, kcSub, kcMedal }) do tween(l, 0.4, { TextTransparency = 1 }) end
		end)
	end
	Net.Spawned.OnClientEvent:Connect(function() kcStreak = 0 kcChain = 0 end)
	Net.Died.OnClientEvent:Connect(function() kcStreak = 0 kcChain = 0 end)

	Net.Feed.OnClientEvent:Connect(function(f)
		if type(f) ~= "table" then return end
		feedN += 1
		local w = WEAPONS[f.w]
		local myName = player.Name
		local mineK = (f.k == myName or f.k == player.DisplayName)
		local mineV = (f.v == myName or f.v == player.DisplayName)
		if mineK and f.k ~= "" then pcall(killConfirm, f.v, f.h, w and w.name or f.w) end
		local function tcol(t, nm)
			if nm == myName or nm == player.DisplayName then return COL.accent2 end
			local mt = player:GetAttribute("Team")
			if t == nil or t == "" or t == "FFA" or t ~= mt then return RGB(255, 80, 80) end
			return RGB(225, 235, 250)
		end
		local row = new("Frame", { Size = UDim2.fromOffset(0, 36), AutomaticSize = Enum.AutomaticSize.X, BackgroundColor3 = (mineK and RGB(60, 44, 8)) or (mineV and RGB(66, 12, 12)) or RGB(8, 10, 10),
			BackgroundTransparency = 0, BorderSizePixel = 0, LayoutOrder = feedN }, HUD.feedBox)
		-- fades in from the left like a broadcast strip; your own kills glow gold, your deaths red
		new("UIGradient", { Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.15), NumberSequenceKeypoint.new(0.7, 0.45), NumberSequenceKeypoint.new(1, 1) }), Rotation = 0 }, row)
		local edgeBar = new("Frame", { Size = UDim2.new(0, 3, 0, 36), BorderSizePixel = 0, BackgroundColor3 = mineK and COL.accent2 or (mineV and COL.red or COL.line), ZIndex = 3 }, row)
		edgeBar.LayoutOrder = 0
		new("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 8), VerticalAlignment = Enum.VerticalAlignment.Center, SortOrder = Enum.SortOrder.LayoutOrder }, row)
		new("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 40) }, row)
		-- team-colour spine on the killer side
		local function part(txt, col, order, font, size)
			return new("TextLabel", { Size = UDim2.fromOffset(0, 36), AutomaticSize = Enum.AutomaticSize.X, BackgroundTransparency = 1, Text = txt, TextColor3 = col, Font = font or Enum.Font.GothamBlack,
				TextSize = size or 17, LayoutOrder = order, TextStrokeTransparency = 0.6 }, row)
		end
		if f.k ~= "" then
			part(string.upper(tostring(f.k)), tcol(f.kt, f.k), 1)
			-- real weapon silhouette
			local gv = new("ViewportFrame", { Size = UDim2.fromOffset(86, 30), BackgroundTransparency = 1, LayoutOrder = 2, Ambient = RGB(255, 255, 255), LightColor = RGB(255, 255, 255),
				LightDirection = Vector3.new(0.2, -0.6, -1) }, row)
			local okG = pcall(function()
				if not w then error("no weapon") end
				local gcam = new("Camera", { FieldOfView = 24 }, gv)
				gv.CurrentCamera = gcam
				local gm = XT.gun(w)
				for _, d in ipairs(gm:GetDescendants()) do
					if d:IsA("BasePart") then d.Color = RGB(236, 238, 242):Lerp(d.Color, 0.5) d.Material = Enum.Material.SmoothPlastic end
				end
				gm.Parent = gv
				local bcf, size = gm:GetBoundingBox()
				local th = math.tan(math.rad(12))
				local d = math.max((size.Z / 2 * 1.1) / (th * (86 / 30)), (size.Y / 2 * 1.3) / th) + size.X / 2
				gcam.CFrame = CFrame.lookAt(bcf.Position + Vector3.new(-d, 0.05, 0), bcf.Position)
			end)
			if not okG then
				gv:Destroy()
				new("TextLabel", { Size = UDim2.fromOffset(0, 20), AutomaticSize = Enum.AutomaticSize.X, BackgroundColor3 = COL.panel2, BorderSizePixel = 0,
					Text = " " .. string.upper(w and w.name or tostring(f.w or "?")) .. " ", TextColor3 = COL.accent2, Font = Enum.Font.GothamBlack, TextSize = 12, LayoutOrder = 2 }, row)
			end
			if f.h then
				local hs = new("TextLabel", { Size = UDim2.fromOffset(0, 18), AutomaticSize = Enum.AutomaticSize.X, BackgroundColor3 = COL.red, BorderSizePixel = 0, Text = " HS ",
					TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBlack, TextSize = 12, LayoutOrder = 3 }, row)
				corner(hs, 3)
			end
		else
			part("ELIMINATED", COL.dim, 1, Enum.Font.GothamBlack, 12)
		end
		if f.k ~= "" then part("X", COL.red, 3.5, Enum.Font.GothamBlack, 14) end
		part(string.upper(tostring(f.v)), tcol(f.vt, f.v), 4)
		local sc = new("UIScale", { Scale = 0.7 }, row)
		tween(sc, 0.2, { Scale = 1 }, Enum.EasingStyle.Back)
		task.delay(7, function()
			if row.Parent then
				tween(row, 0.4, { BackgroundTransparency = 1 })
				for _, c in ipairs(row:GetDescendants()) do
					if c:IsA("TextLabel") then tween(c, 0.4, { TextTransparency = 1, BackgroundTransparency = 1 }) end
				end
				task.delay(0.45, function() row:Destroy() end)
			end
		end)
		local kids = {}
		for _, c in ipairs(HUD.feedBox:GetChildren()) do
			if c:IsA("Frame") then table.insert(kids, c) end
		end
		table.sort(kids, function(a, b) return a.LayoutOrder < b.LayoutOrder end)
		while #kids > 6 do
			table.remove(kids, 1):Destroy()
		end
	end)

	-- health (bottom left; top-left on touch so the thumbstick stays clear)
	-- health: big number + segmented bar with a lagging damage trail
	HUD.hpBox = new("Frame", { Size = UDim2.fromOffset(310, 46), BackgroundTransparency = 1 }, hud)
	HUD.hpBox.Position = UI.touch and UDim2.fromOffset(28, 22) or UDim2.new(0, 32, 1, -78)
	HUD.hpText = label(HUD.hpBox, { Size = UDim2.fromOffset(74, 40), Position = UDim2.fromOffset(0, 0), Text = "100", TextSize = 38, Font = Enum.Font.GothamBlack,
		TextXAlignment = Enum.TextXAlignment.Left, TextStrokeTransparency = 0.5, ZIndex = 4 })
	label(HUD.hpBox, { Size = UDim2.fromOffset(40, 12), Position = UDim2.fromOffset(2, 34), Text = "HEALTH", TextSize = 9, TextColor3 = COL.dim, Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Left })
	local hpTrack = new("Frame", { Size = UDim2.fromOffset(222, 18), Position = UDim2.fromOffset(86, 14), BackgroundColor3 = RGB(8, 10, 10), BackgroundTransparency = 0.1, BorderSizePixel = 0, ClipsDescendants = true }, HUD.hpBox)
	corner(hpTrack, 2)
	HUD.hpStroke = stroke(hpTrack, COL.line, 1.5, 0.2)
	HUD.hpTrail = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(255, 255, 255), BackgroundTransparency = 0.35, BorderSizePixel = 0, ZIndex = 2 }, hpTrack)
	HUD.hpFill = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = COL.green, BorderSizePixel = 0, ZIndex = 3 }, hpTrack)
	new("UIGradient", { Rotation = 90, Color = ColorSequence.new(RGB(255, 255, 255), RGB(170, 170, 170)) }, HUD.hpFill)
	new("Frame", { Size = UDim2.new(1, 0, 0.4, 0), BackgroundColor3 = RGB(255, 255, 255), BackgroundTransparency = 0.82, BorderSizePixel = 0, ZIndex = 5 }, hpTrack)
	for i = 1, 9 do
		new("Frame", { Size = UDim2.fromOffset(2, 18), Position = UDim2.new(i / 10, -1, 0, 0), BackgroundColor3 = RGB(8, 10, 10), BorderSizePixel = 0, ZIndex = 6 }, hpTrack)
	end
	HUD.hpLast, HUD.hpTok = 1, 0
	function HUD.hpSet(ratio, hp)
		ratio = math.clamp(ratio, 0, 1)
		HUD.hpFill.Size = UDim2.fromScale(ratio, 1)
		local col = (ratio > 0.5) and COL.green or ((ratio > 0.25) and COL.gold or COL.red)
		HUD.hpFill.BackgroundColor3 = col
		if ratio < HUD.hpLast - 0.001 then
			HUD.hpFill.BackgroundColor3 = RGB(255, 255, 255)
			tween(HUD.hpFill, 0.25, { BackgroundColor3 = col })
			HUD.hpTok += 1
			local tk = HUD.hpTok
			task.delay(0.45, function()
				if tk == HUD.hpTok then tween(HUD.hpTrail, 0.5, { Size = UDim2.fromScale(ratio, 1) }) end
			end)
		elseif ratio > HUD.hpLast + 0.001 then
			HUD.hpTrail.Size = UDim2.fromScale(ratio, 1)
		end
		HUD.hpLast = ratio
		local low = ratio < 0.3
		HUD.hpText.TextColor3 = low and COL.red or COL.text
		HUD.hpStroke.Color = low and COL.red or COL.line
		HUD.hpStroke.Transparency = low and (0.1 + 0.4 * (0.5 + 0.5 * math.sin(os.clock() * 7))) or 0.2
		HUD.hpText.Text = tostring(math.ceil(hp))
	end
	HUD.lvl = label(hud, { Size = UDim2.fromOffset(10, 10), BackgroundTransparency = 1, TextSize = 1, Text = "", Visible = false })
	-- low health: screen goes red (redder the lower you are), heartbeat + heavy breathing
	do
		local lowFrames = {}
		local function edgeF(rot, size, pos, anchor)
			local f = new("Frame", { Size = size, Position = pos, AnchorPoint = anchor, BackgroundColor3 = RGB(200, 0, 0), BorderSizePixel = 0, ZIndex = 0, Visible = false }, hud)
			local g = new("UIGradient", { Rotation = rot, Transparency = NumberSequence.new(0, 1) }, f)
			table.insert(lowFrames, { f = f, g = g })
		end
		edgeF(90, UDim2.new(1, 0, 0.45, 0), UDim2.fromScale(0, 0), Vector2.new(0, 0))
		edgeF(270, UDim2.new(1, 0, 0.45, 0), UDim2.fromScale(0, 1), Vector2.new(0, 1))
		edgeF(0, UDim2.new(0.32, 0, 1, 0), UDim2.fromScale(0, 0), Vector2.new(0, 0))
		edgeF(180, UDim2.new(0.32, 0, 1, 0), UDim2.fromScale(1, 0), Vector2.new(1, 0))
		local wash = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(150, 0, 0), BorderSizePixel = 0, BackgroundTransparency = 1, ZIndex = 0, Visible = false }, hud)
		local nextBeat, nextBreath, beatPhase = 0, 0, 0
		RunService.Heartbeat:Connect(function(dt)
			local ch = player.Character
			local h = ch and ch:FindFirstChildOfClass("Humanoid")
			local ratio = (h and h.Health > 0) and (h.Health / math.max(1, h.MaxHealth)) or 1
			local on = hud.Visible and h and h.Health > 0 and ratio < 0.5
			if not on then
				for _, e in ipairs(lowFrames) do e.f.Visible = false end
				wash.Visible = false
				return
			end
			local k = math.clamp((0.5 - ratio) / 0.42, 0, 1) -- 0 at half health, 1 at ~8%
			local now = os.clock()
			local bpm = 78 + 80 * k
			local period = 60 / bpm
			if now >= nextBeat then
				nextBeat = now + period
				beatPhase = now
				playSound(SND.bass, 0.5, (0.12 + 0.38 * k) * S.g.sfx, nil, 0.22)
				task.delay(0.17, function() playSound(SND.bass, 0.42, (0.07 + 0.26 * k) * S.g.sfx, nil, 0.2) end)
			end
			if now >= nextBreath then
				nextBreath = now + (2.7 - 1.5 * k)
				playSound(SND.lunge, 0.42 + 0.12 * k, (0.14 + 0.3 * k) * S.g.sfx, nil, 0.9)
				task.delay(0.95 - 0.3 * k, function() playSound(SND.slash, 0.34 + 0.08 * k, (0.1 + 0.26 * k) * S.g.sfx, nil, 0.9) end)
			end
			for _, e in ipairs(lowFrames) do e.f.Visible = false end
			wash.Visible = false
			do return end
			local pulse = math.exp(-(now - beatPhase) * 7) -- sharp thump right after each beat, then fades
			local base = 0.18 + 0.5 * k
			local a = math.clamp(base + pulse * (0.15 + 0.3 * k), 0, 0.97)
			for _, e in ipairs(lowFrames) do
				e.f.Visible = true
				e.g.Transparency = NumberSequence.new(1 - a, 1)
			end
			wash.Visible = true
			wash.BackgroundTransparency = 1 - (0.05 + 0.22 * k + pulse * 0.1 * k)
		end)
	end

	local switchSlotRef
	-- operator card (bottom left): your operator, kills / deaths / score / streak
	do
		local oc = new("Frame", { Size = UDim2.fromOffset(300, 92), BackgroundColor3 = RGB(8, 10, 10), BackgroundTransparency = 0.25, BorderSizePixel = 0 }, hud)
		oc.Position = UI.touch and UDim2.fromOffset(28, 80) or UDim2.new(0, 32, 1, -190)
		corner(oc, 4)
		stroke(oc, COL.line, 1, 0.45)
		new("Frame", { Size = UDim2.fromOffset(4, 92), BackgroundColor3 = COL.accent, BorderSizePixel = 0 }, oc)
		local bust = new("ViewportFrame", { Size = UDim2.fromOffset(78, 88), Position = UDim2.fromOffset(8, 2), BackgroundTransparency = 1, ZIndex = 3, Ambient = RGB(255, 255, 255),
			LightColor = RGB(255, 255, 255), LightDirection = Vector3.new(0.3, -0.4, -1) }, oc)
		local bcam = new("Camera", { FieldOfView = 24 }, bust)
		bust.CurrentCamera = bcam
		bcam.CFrame = CFrame.lookAt(Vector3.new(0, 4.6, 7.6), Vector3.new(0, 4.1, 0))
		local curOp
		local opName = { Text = "" }
		local stat = label(oc, { Size = UDim2.fromOffset(200, 34), Position = UDim2.fromOffset(94, 12), Text = "", TextSize = 24, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left })
		local sub = label(oc, { Size = UDim2.fromOffset(200, 16), Position = UDim2.fromOffset(94, 50), Text = "", TextSize = 12, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left })
		local streakL = label(oc, { Size = UDim2.fromOffset(200, 14), Position = UDim2.fromOffset(94, 68), Text = "", TextSize = 11, Font = Enum.Font.GothamBlack, TextColor3 = COL.orange, TextXAlignment = Enum.TextXAlignment.Left })
		HUD.opCard = oc
		function HUD.updateOpCard()
			local id = (P.equip and P.equip.operator) or "OP_STANDARD"
			if id ~= curOp then
				curOp = id
				for _, c in ipairs(bust:GetChildren()) do if c:IsA("Model") then c:Destroy() end end
				pcall(function()
					local m = XT.soldier(id)
					m.Parent = bust
				end)
				opName.Text = OPERATORS[id] and OPERATORS[id].name or "OPERATOR"
			end
			local r
			for _, rr in ipairs(HUD.rows) do if rr.n == player.Name or rr.n == player.DisplayName then r = rr break end end
			stat.Text = string.format("%d  K   %d  D", r and r.k or 0, r and r.d or 0)
			sub.Text = string.format("SCORE  %d     AST  %d", r and r.s or 0, r and r.a or 0)
			streakL.Text = (kcStreakRef and kcStreakRef() or 0) >= 2 and ("STREAK  " .. kcStreakRef()) or ""
		end
	end

	-- weapon panel (bottom right): current gun picture, ammo, both weapon slots and grenade / stim
	local function hudVp(parent, pos, size, z)
		local vpf = new("ViewportFrame", { Size = size, Position = pos, BackgroundTransparency = 1, ZIndex = z or 3, Ambient = RGB(255, 255, 255), LightColor = RGB(255, 255, 255),
			LightDirection = Vector3.new(0.2, -0.6, -1) }, parent)
		local cam = new("Camera", { FieldOfView = 22 }, vpf)
		vpf.CurrentCamera = cam
		return { vp = vpf, cam = cam, aspect = size.X.Offset / math.max(1, size.Y.Offset) }
	end
	local function hudSetGun(h, id)
		if h.id == id then return end
		h.id = id
		if h.model then h.model:Destroy() h.model = nil end
		local def = WEAPONS[id]
		if not def then return end
		local ok = pcall(function()
			local m = XT.gun(def)
			for _, d in ipairs(m:GetDescendants()) do
				if d:IsA("BasePart") then d.Color = RGB(236, 238, 244):Lerp(d.Color, 0.5) d.Material = Enum.Material.SmoothPlastic d.Transparency = math.min(d.Transparency, 0.6) end
			end
			m.Parent = h.vp
			h.model = m
			local bcf, size = m:GetBoundingBox()
			local th = math.tan(math.rad(h.cam.FieldOfView / 2))
			local d = math.max((size.Z / 2 * 1.08) / (th * h.aspect), (size.Y / 2 * 1.35) / th) + size.X / 2
			h.cam.CFrame = CFrame.lookAt(bcf.Position + Vector3.new(-d, 0.05, 0), bcf.Position)
		end)
	end
	HUD.ammoBox = new("Frame", { Size = UDim2.fromOffset(360, 150), AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -36, 1, -30), BackgroundColor3 = RGB(8, 10, 10), BackgroundTransparency = 1,
		BorderSizePixel = 0 }, hud)
	if UI.touch then HUD.ammoBox.Position = UDim2.new(1, -368, 0, 10 + 170) end
	corner(HUD.ammoBox, 4)
	-- (no box, no gun pictures: just ammo numbers, the gun name and grenade / stim counts)
	HUD.slotTxt = label(HUD.ammoBox, { Size = UDim2.fromOffset(112, 14), Position = UDim2.fromOffset(0, 30), Text = "", TextSize = 11, TextColor3 = COL.accent2,
		TextXAlignment = Enum.TextXAlignment.Right, Font = Enum.Font.GothamBold })
	HUD.mag = label(HUD.ammoBox, { Size = UDim2.fromOffset(112, 60), Position = UDim2.fromOffset(0, 44), Text = "30", TextSize = 58, TextXAlignment = Enum.TextXAlignment.Right, TextStrokeTransparency = 0.6, Font = Enum.Font.GothamBlack })
	HUD.res = label(HUD.ammoBox, { Size = UDim2.fromOffset(112, 24), Position = UDim2.fromOffset(0, 104), Text = "/ 120", TextSize = 20, TextColor3 = COL.dim,
		TextXAlignment = Enum.TextXAlignment.Right })
	HUD.gunMain = hudVp(HUD.ammoBox, UDim2.fromOffset(126, 72), UDim2.fromOffset(230, 66), 3)
	HUD.wName = label(HUD.ammoBox, { Size = UDim2.fromOffset(230, 18), Position = UDim2.fromOffset(126, 54), Text = "", TextSize = 14, TextColor3 = COL.text,
		TextXAlignment = Enum.TextXAlignment.Left, Font = Enum.Font.GothamBlack })
	HUD.reloadBar = new("Frame", { Size = UDim2.fromOffset(0, 4), Position = UDim2.fromOffset(0, 138), BackgroundColor3 = COL.orange, BorderSizePixel = 0, Visible = false }, HUD.ammoBox)
	HUD.slotBox = {}
	for i = 1, 2 do
		local f = new("TextButton", { Size = UDim2.fromOffset(116, 44), Position = UDim2.fromOffset(12 + (i - 1) * 122, 114), BackgroundColor3 = RGB(24, 26, 26), AutoButtonColor = false, Text = "",
			BorderSizePixel = 0, Visible = false }, HUD.ammoBox)
		corner(f, 3)
		local st = stroke(f, COL.line, 1, 0.3)
		label(f, { Size = UDim2.fromOffset(16, 14), Position = UDim2.fromOffset(4, 2), Text = tostring(i), TextSize = 11, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent2, ZIndex = 5 })
		label(f, { Size = UDim2.fromOffset(60, 12), Position = UDim2.fromOffset(52, 30), Text = i == 1 and "PRIMARY" or "SIDEARM", TextSize = 8, TextColor3 = COL.dim, ZIndex = 5 })
		HUD.slotBox[i] = { f = f, st = st, icon = hudVp(f, UDim2.fromOffset(18, 2), UDim2.fromOffset(94, 34), 4) }
		f.Activated:Connect(function() if switchSlotRef then switchSlotRef(i) end end)
	end
	-- grenade + stim with real 3D models, count and key
	HUD.gear = {}
	for gi, g in ipairs({ { "Frags", "FRAG", "G", "fragModel", "Grenade" }, { "Stims", "STIM", "H", "stimModel", "Stim" } }) do
		local f = new("Frame", { Size = UDim2.fromOffset(46, 44), Position = UDim2.fromOffset(250 + (gi - 1) * 4 + (gi - 1) * 46 - 0, 114), BackgroundColor3 = RGB(24, 26, 26), BorderSizePixel = 0 }, HUD.ammoBox)
		f.Position = UDim2.fromOffset(226 + (gi - 1) * 66, 2)
		f.Size = UDim2.fromOffset(60, 44)
		f.BackgroundTransparency = 1
		corner(f, 3)
		local st = stroke(f, COL.line, 1, 1)
		local v = hudVp(f, UDim2.fromOffset(2, 2), UDim2.fromOffset(36, 28), 4)
		v.cam.FieldOfView = 30
		local m = XT[g[4]]()
		for _, d in ipairs(m:GetDescendants()) do if d:IsA("BasePart") then d.Anchored = true end end
		m:PivotTo(m:GetPivot() * CFrame.Angles(0, math.rad(25), gi == 2 and math.rad(-30) or 0))
		m.Parent = v.vp
		local bcf, size = m:GetBoundingBox()
		local th = math.tan(math.rad(15))
		v.cam.CFrame = CFrame.lookAt(bcf.Position + Vector3.new(0, 0.1, math.max(size.X, size.Y, size.Z) / 2 / th * 1.15 + size.Z / 2), bcf.Position)
		local cnt = label(f, { Size = UDim2.fromOffset(40, 14), Position = UDim2.fromOffset(0, 29), Text = "0", TextSize = 12, Font = Enum.Font.GothamBlack, ZIndex = 5 })
		label(f, { Size = UDim2.fromOffset(14, 12), Position = UDim2.fromOffset(2, 0), Text = g[3], TextSize = 9, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent2, ZIndex = 6 })
		HUD.gear[gi] = { f = f, st = st, cnt = cnt, attr = g[1] }
	end
	do
		local boostLbl = label(HUD.ammoBox, { Size = UDim2.fromOffset(110, 12), Position = UDim2.fromOffset(126, 40), Text = "", TextSize = 9, TextColor3 = COL.green, TextXAlignment = Enum.TextXAlignment.Left, Font = Enum.Font.GothamBold })
		local function upd()
			local boost = Workspace:GetServerTimeNow() < (player:GetAttribute("StimUntil") or 0)
			for _, g in ipairs(HUD.gear) do
				local n = player:GetAttribute(g.attr) or 0
				g.cnt.Text = "x" .. n
				g.cnt.TextColor3 = n > 0 and COL.text or COL.dim
				g.st.Color = n > 0 and COL.accent2 or COL.line
			end
			HUD.gear[2].st.Color = boost and COL.green or HUD.gear[2].st.Color
			boostLbl.Text = boost and "STIM ACTIVE" or ""
		end
		player:GetAttributeChangedSignal("Frags"):Connect(upd)
		player:GetAttributeChangedSignal("Stims"):Connect(upd)
		player:GetAttributeChangedSignal("StimUntil"):Connect(function() upd(); task.delay(5.1, upd) end)
		upd()
	end

	-- crosshair
	HUD.cross = new("Frame", { Size = UDim2.fromOffset(0, 0), Position = UDim2.fromScale(0.5, 0.5), BackgroundTransparency = 1 }, hud)
	HUD.crossBars = {}
	for i = 1, 4 do
		local horiz = (i <= 2)
		HUD.crossBars[i] = new("Frame", { Size = horiz and UDim2.fromOffset(10, 2) or UDim2.fromOffset(2, 10), AnchorPoint = Vector2.new(0.5, 0.5),
			BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0 }, HUD.cross)
		stroke(HUD.crossBars[i], RGB(0, 0, 0), 1, 0.4)
	end
	HUD.crossDot = new("Frame", { Size = UDim2.fromOffset(3, 3), AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0 }, HUD.cross)
	function HUD.updateCross(spreadPx)
		local cs = S.g.crossScale
		local g = math.clamp(spreadPx, 4, 90) * cs
		local c = CROSS_COLORS[S.g.crossColor] or CROSS_COLORS[1]
		local gl = math.floor(10 * cs + 0.5)
		HUD.crossBars[1].Size, HUD.crossBars[2].Size = UDim2.fromOffset(gl, 2), UDim2.fromOffset(gl, 2)
		HUD.crossBars[3].Size, HUD.crossBars[4].Size = UDim2.fromOffset(2, gl), UDim2.fromOffset(2, gl)
		HUD.crossBars[1].Position = UDim2.fromOffset(-g - 5, 0)
		HUD.crossBars[2].Position = UDim2.fromOffset(g + 5, 0)
		HUD.crossBars[3].Position = UDim2.fromOffset(0, -g - 5)
		HUD.crossBars[4].Position = UDim2.fromOffset(0, g + 5)
		for _, b in ipairs(HUD.crossBars) do
			b.BackgroundColor3 = c
			b.BackgroundTransparency = 1 - S.g.crossAlpha
		end
		HUD.crossDot.BackgroundColor3 = c
		HUD.crossDot.Visible = S.g.crossDot ~= false
		HUD.crossDot.BackgroundTransparency = 1 - S.g.crossAlpha
	end
	HUD.hitMark = new("Frame", { Size = UDim2.fromOffset(0, 0), Position = UDim2.fromScale(0.5, 0.5), BackgroundTransparency = 1 }, hud)
	HUD.hitBars = {}
	for i = 1, 4 do
		local b = new("Frame", { Size = UDim2.fromOffset(14, 3), AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0,
			BackgroundTransparency = 1, Rotation = (i % 2 == 0) and 45 or -45, Position = UDim2.fromOffset((i <= 2 and -1 or 1) * 12, ((i % 2 == 1) and -1 or 1) * 12) }, HUD.hitMark)
		HUD.hitBars[i] = b
	end
	local hitSeq = 0
	Net.Hit.OnClientEvent:Connect(function(kind)
		hitSeq += 1
		local my = hitSeq
		local c = (kind == "kill" and COL.red) or (kind == "head" and COL.gold) or RGB(255, 255, 255)
		if S.g.hitMark then
			for _, b in ipairs(HUD.hitBars) do
				b.BackgroundColor3 = c
				b.BackgroundTransparency = 0
			end
		end
		if S.g.hitSound then
			playSound(SND.hit, kind == "kill" and 0.8 or (kind == "head" and 1.4 or 1.1), 0.5 * S.g.sfx)
			if kind == "kill" then playSound(SND.ping, 1.0, 0.5 * S.g.sfx) end
		end
		task.delay(kind == "kill" and 0.35 or 0.18, function()
			if hitSeq == my then
				for _, b in ipairs(HUD.hitBars) do tween(b, 0.15, { BackgroundTransparency = 1 }) end
			end
		end)
	end)

	-- damage indicators + vignette
	HUD.vignette = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(255, 0, 0), BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 0 }, hud)
	Net.Dmg.OnClientEvent:Connect(function(fromPos)
		HUD.vignette.BackgroundTransparency = 1
		tween(HUD.vignette, 0.6, { BackgroundTransparency = 1 })
		playSound(SND.collide, 1.8, 0.25 * S.g.sfx)
		if typeof(fromPos) == "Vector3" and camera then
			local ring = new("Frame", { Size = UDim2.fromOffset(260, 260), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
				BackgroundTransparency = 1 }, hud)
			local look = camera.CFrame.LookVector
			local rel = fromPos - camera.CFrame.Position
			local a = math.deg(math.atan2(rel.X * look.Z - rel.Z * look.X, rel.X * look.X + rel.Z * look.Z))
			ring.Rotation = a
			local bar = new("Frame", { Size = UDim2.fromOffset(70, 8), AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.fromScale(0.5, 0), BackgroundColor3 = COL.red,
				BorderSizePixel = 0 }, ring)
			corner(bar, 4)
			tween(bar, 1.2, { BackgroundTransparency = 1 })
			Debris:AddItem(ring, 1.3)
		end
	end)

	-- xp popups (left of centre, never over the crosshair)
	HUD.xpBox = new("Frame", { Size = UDim2.fromOffset(240, 120), Position = UDim2.new(0.5, 70, 0.5, 20), BackgroundTransparency = 1 }, hud)
	new("UIListLayout", { Padding = UDim.new(0, 2), SortOrder = Enum.SortOrder.LayoutOrder, VerticalAlignment = Enum.VerticalAlignment.Top }, HUD.xpBox)
	local xpN = 0
	Net.Xp.OnClientEvent:Connect(function(amount, reason)
		if type(amount) ~= "number" then return end
		xpN += 1
		local l = label(HUD.xpBox, { Size = UDim2.fromOffset(240, 20), Text = string.format("+%d  %s", amount, tostring(reason or "")), TextSize = 15,
			TextColor3 = COL.gold, TextXAlignment = Enum.TextXAlignment.Left, LayoutOrder = xpN, TextStrokeTransparency = 0.5 })
		task.delay(2.4, function()
			tween(l, 0.4, { TextTransparency = 1, TextStrokeTransparency = 1 })
			task.delay(0.45, function() l:Destroy() end)
		end)
		local kids = HUD.xpBox:GetChildren()
		if #kids > 7 then
			for _, c in ipairs(kids) do
				if c:IsA("TextLabel") then
					c:Destroy()
					break
				end
			end
		end
	end)
	Net.LevelUp.OnClientEvent:Connect(function(lv)
		showBanner("LEVEL UP!  LEVEL " .. tostring(lv), "good")
		playSound(SND.victory, 1.2, 0.5 * S.g.sfx)
	end)

	-- scoreboard (Tab)
	HUD.sbFrame = new("Frame", { Size = UDim2.fromOffset(880, 470), Position = UDim2.fromScale(0.5, 0.5), AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = RGB(8, 10, 10),
		BackgroundTransparency = 0.12, Visible = false, ZIndex = 20 }, hud)
	corner(HUD.sbFrame, 4)
	stroke(HUD.sbFrame, COL.line, 1, 0.3)
	HUD.sbTitle = label(HUD.sbFrame, { Size = UDim2.fromOffset(500, 30), Position = UDim2.fromOffset(20, 12), Text = "SCOREBOARD", TextSize = 22, Font = Enum.Font.GothamBlack, ZIndex = 21, TextXAlignment = Enum.TextXAlignment.Left })
	HUD.sbList = new("Frame", { Size = UDim2.fromOffset(840, 400), Position = UDim2.fromOffset(20, 52), BackgroundTransparency = 1, ZIndex = 21 }, HUD.sbFrame)
	function HUD.fillBoard(rows)
		for _, c in ipairs(HUD.sbList:GetChildren()) do c:Destroy() end
		local mode = MODES[State:GetAttribute("ModeId") or ""]
		local myTeam = player:GetAttribute("Team")
		local teamMode = mode and mode.teams and myTeam and myTeam ~= ""
		HUD.sbTitle.Text = mode and (string.upper(mode.name) .. "  -  SCOREBOARD") or "SCOREBOARD"
		local COLS = { { "PLAYER", 12, 150, Enum.TextXAlignment.Left }, { "K", 168, 36 }, { "D", 206, 36 }, { "A", 244, 36 }, { "K/D", 282, 46 }, { "ACC", 330, 44 }, { "SCORE", 374, 50 } }
		local function panel(x, w, title, col, list)
			local f = new("Frame", { Size = UDim2.fromOffset(w, 392), Position = UDim2.fromOffset(x, 0), BackgroundColor3 = RGB(16, 18, 18), BackgroundTransparency = 0.2, BorderSizePixel = 0, ZIndex = 21 }, HUD.sbList)
			corner(f, 4)
			new("Frame", { Size = UDim2.new(1, 0, 0, 4), BackgroundColor3 = col, BorderSizePixel = 0, ZIndex = 22 }, f)
			local tk, td, ta, th, ts, thit = 0, 0, 0, 0, 0, 0
			for _, r in ipairs(list) do tk += r.k or 0 td += r.d or 0 ta += r.a or 0 ts += r.s or 0 end
			label(f, { Size = UDim2.fromOffset(220, 26), Position = UDim2.fromOffset(12, 10), Text = title, TextSize = 18, Font = Enum.Font.GothamBlack, TextColor3 = col, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 22 })
			label(f, { Size = UDim2.fromOffset(150, 26), Position = UDim2.new(1, -162, 0, 10), Text = tk .. " KILLS", TextSize = 18, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 22 })
			for _, h in ipairs(COLS) do
				label(f, { Size = UDim2.fromOffset(h[3], 16), Position = UDim2.fromOffset(h[2], 42), Text = h[1], TextSize = 10, TextColor3 = COL.dim, Font = Enum.Font.GothamBold,
					TextXAlignment = h[4] or Enum.TextXAlignment.Center, ZIndex = 22 })
			end
			local y = 62
			local accSum, accN = 0, 0
			for i, r in ipairs(list) do
				if i > 8 then break end
				local mine = (r.n == player.Name or r.n == player.DisplayName)
				local rowF = new("Frame", { Size = UDim2.fromOffset(w - 16, 28), Position = UDim2.fromOffset(8, y), BackgroundColor3 = mine and RGB(70, 54, 12) or RGB(26, 28, 28),
					BackgroundTransparency = mine and 0.1 or 0.35, BorderSizePixel = 0, ZIndex = 22 }, f)
				corner(rowF, 3)
				local vals = { { tostring(r.n), 1 }, { r.k or 0, 2 }, { r.d or 0, 3 }, { r.a or 0, 4 }, { string.format("%.2f", (r.k or 0) / math.max(1, r.d or 0)), 5 }, { (r.c or 0) .. "%", 6 }, { r.s or 0, 7 } }
				for _, v in ipairs(vals) do
					local cdef = COLS[v[2]]
					label(rowF, { Size = UDim2.fromOffset(cdef[3], 28), Position = UDim2.fromOffset(cdef[2] - 8, 0), Text = tostring(v[1]), TextSize = 13, Font = v[2] == 1 and Enum.Font.GothamBold or Enum.Font.Gotham,
						TextColor3 = (r.x == false) and COL.dim or (mine and COL.accent2 or COL.text), TextXAlignment = cdef[4] or Enum.TextXAlignment.Center, TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 23 })
				end
				if r.c then accSum += r.c accN += 1 end
				y += 31
			end
			-- team totals
			local totF = new("Frame", { Size = UDim2.fromOffset(w - 16, 28), Position = UDim2.fromOffset(8, 356), BackgroundColor3 = col, BackgroundTransparency = 0.82, BorderSizePixel = 0, ZIndex = 22 }, f)
			corner(totF, 3)
			local tv = { { "TEAM", 1 }, { tk, 2 }, { td, 3 }, { ta, 4 }, { string.format("%.2f", tk / math.max(1, td)), 5 }, { (accN > 0 and math.floor(accSum / accN + 0.5) or 0) .. "%", 6 }, { ts, 7 } }
			for _, v in ipairs(tv) do
				local cdef = COLS[v[2]]
				label(totF, { Size = UDim2.fromOffset(cdef[3], 28), Position = UDim2.fromOffset(cdef[2] - 8, 0), Text = tostring(v[1]), TextSize = 13, Font = Enum.Font.GothamBlack,
					TextXAlignment = cdef[4] or Enum.TextXAlignment.Center, ZIndex = 23 })
			end
		end
		if teamMode then
			local mates, foes = {}, {}
			for _, r in ipairs(rows) do
				if r.t == myTeam then table.insert(mates, r) else table.insert(foes, r) end
			end
			panel(0, 416, "YOUR TEAM", RGB(120, 200, 255), mates)
			panel(424, 416, "ENEMY TEAM", COL.red, foes)
		else
			panel(0, 840, "ALL PLAYERS", COL.accent, rows)
		end
	end
	local function fillRows(listFrame, rows, myName, maxRows, z)
		for _, c in ipairs(listFrame:GetChildren()) do
			if c:IsA("Frame") then c:Destroy() end
		end
		for i, r in ipairs(rows) do
			if i > maxRows then break end
			local mine = (r.n == myName)
			local f = new("Frame", { Size = UDim2.fromOffset(690, 26), BackgroundColor3 = mine and COL.line or COL.panel, BackgroundTransparency = mine and 0.2 or 0.5,
				BorderSizePixel = 0, LayoutOrder = i, ZIndex = z }, listFrame)
			corner(f, 5)
			new("Frame", { Size = UDim2.fromOffset(4, 26), BackgroundColor3 = (r.t and r.t ~= "") and teamColor(r.t) or COL.dim, BorderSizePixel = 0, ZIndex = z }, f)
			label(f, { Size = UDim2.fromOffset(330, 26), Position = UDim2.fromOffset(12, 0), Text = (r.b and "[BOT] " or "") .. tostring(r.n), TextSize = 14, ZIndex = z,
				TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd })
			local vals = { { r.k, 345 }, { r.d, 415 }, { r.a, 485 }, { r.s, 565 } }
			for _, v in ipairs(vals) do
				label(f, { Size = UDim2.fromOffset(70, 26), Position = UDim2.fromOffset(v[2], 0), Text = tostring(v[1] or 0), TextSize = 14, ZIndex = z,
					TextColor3 = COL.text })
			end
		end
	end
	Net.Scores.OnClientEvent:Connect(function(rows, a, b)
		if type(rows) ~= "table" then return end
		HUD.rows = rows
		HUD.scoreAv, HUD.scoreBv = a or 0, b or 0
		pcall(HUD.updateRoster, rows)
		if HUD.sbFrame.Visible then HUD.fillBoard(rows) end
	end)
	function HUD.showScoreboard(on)
		HUD.sbFrame.Visible = on and inMatch()
		if HUD.sbFrame.Visible then HUD.fillBoard(HUD.rows) end
	end

	-- scope overlay
	HUD.scope = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, ZIndex = 5 }, gui)
	HUD.scopeL = new("Frame", { BackgroundColor3 = RGB(0, 0, 0), BorderSizePixel = 0, ZIndex = 5 }, HUD.scope)
	HUD.scopeR = new("Frame", { BackgroundColor3 = RGB(0, 0, 0), BorderSizePixel = 0, ZIndex = 5, AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromScale(1, 0) }, HUD.scope)
	HUD.scopeRing = new("Frame", { Size = UDim2.fromOffset(100, 100), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), BackgroundTransparency = 1,
		ZIndex = 6 }, HUD.scope)
	new("UICorner", { CornerRadius = UDim.new(1, 0) }, HUD.scopeRing)
	new("UIStroke", { Color = RGB(0, 0, 0), Thickness = 3 }, HUD.scopeRing)
	new("Frame", { Size = UDim2.new(1, 0, 0, 2), Position = UDim2.fromScale(0, 0.5), BackgroundColor3 = RGB(0, 0, 0), BorderSizePixel = 0, ZIndex = 7 }, HUD.scope)
	new("Frame", { Size = UDim2.new(0, 2, 1, 0), Position = UDim2.fromScale(0.5, 0), BackgroundColor3 = RGB(0, 0, 0), BorderSizePixel = 0, ZIndex = 7 }, HUD.scope)
	new("Frame", { Size = UDim2.fromOffset(6, 6), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), BackgroundColor3 = COL.red, BorderSizePixel = 0, ZIndex = 8 }, HUD.scope)
	-- optic reticles (red dot / holo ring / 3x scope reticle) shown while aiming with an optic attachment
	HUD.optic = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, ZIndex = 4 }, hud)
	do
		local function dot(parent, size, color, tr)
			local d = new("Frame", { Size = UDim2.fromOffset(size, size), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), BackgroundColor3 = color,
				BackgroundTransparency = tr or 0, BorderSizePixel = 0, ZIndex = 5 }, parent)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, d)
			return d
		end
		local rd = new("Frame", { Name = "RDOT", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false }, HUD.optic)
		dot(rd, 22, RGB(255, 40, 40), 0.82)
		dot(rd, 7, RGB(255, 50, 50), 0)
		local sc = new("Frame", { Name = "ZOOM3", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false }, HUD.optic)
		new("Frame", { Size = UDim2.new(1, 0, 0, 2), Position = UDim2.fromScale(0, 0.5), BackgroundColor3 = RGB(0, 0, 0), BackgroundTransparency = 0.15, BorderSizePixel = 0, ZIndex = 5 }, sc)
		new("Frame", { Size = UDim2.new(0, 2, 1, 0), Position = UDim2.fromScale(0.5, 0), BackgroundColor3 = RGB(0, 0, 0), BackgroundTransparency = 0.15, BorderSizePixel = 0, ZIndex = 5 }, sc)
		for k = -4, 4 do
			if k ~= 0 then
				new("Frame", { Size = UDim2.fromOffset(6, 6), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, k * 26, 0.5, 0), BackgroundColor3 = RGB(0, 0, 0), BorderSizePixel = 0, ZIndex = 5 }, sc)
				new("Frame", { Size = UDim2.fromOffset(6, 6), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, 0.5, k * 26), BackgroundColor3 = RGB(0, 0, 0), BorderSizePixel = 0, ZIndex = 5 }, sc)
			end
		end
		dot(sc, 5, RGB(255, 40, 40), 0)
		local sring = dot(sc, 520, RGB(255, 255, 255), 1)
		new("UIStroke", { Color = RGB(0, 0, 0), Thickness = 60, Transparency = 0.2 }, sring)
		-- CROSS SIGHT reticle (AK-47 optic): four diagonal red ticks around a small red dot, floating between the rear-sight posts
		local ak = new("Frame", { Name = "XSIGHT", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false }, HUD.optic)
		for _, q in ipairs({ { -1, -1 }, { 1, -1 }, { -1, 1 }, { 1, 1 } }) do
			local tk = new("Frame", { Size = UDim2.fromOffset(15, 3), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, q[1] * 22, 0.5, q[2] * 22),
				Rotation = (q[1] * q[2] > 0) and 45 or -45, BackgroundColor3 = RGB(255, 70, 60), BorderSizePixel = 0, ZIndex = 5 }, ak)
			new("UIStroke", { Color = RGB(0, 0, 0), Thickness = 1, Transparency = 0.45 }, tk)
		end
		dot(ak, 4, RGB(255, 50, 50), 0)
		HUD.opticFrames = { RDOT = rd, ZOOM3 = sc, XSIGHT = ak }
	end
	function HUD.setOptic(def, adsT)
		local which
		if def and def.att and not def.scope then
			for _, id in ipairs(def.att) do
				local a = ATTACH_BY_ID[id]
				if a and a.cat == "OPTIC" then which = id end
			end
		end
		local show = which ~= nil and adsT > 0.6
		HUD.optic.Visible = show
		for id, f in pairs(HUD.opticFrames) do f.Visible = show and id == which end
		return show
	end
	function HUD.layoutScope()
		local vp = camera.ViewportSize
		local sq = vp.Y * 0.94
		local bw = math.max(0, (vp.X - sq) / 2)
		HUD.scopeL.Size = UDim2.new(0, bw, 1, 0)
		HUD.scopeR.Size = UDim2.new(0, bw, 1, 0)
		HUD.scopeRing.Size = UDim2.fromOffset(sq, sq)
	end

	-- death screen (centre-low, away from score bar / ammo / health)
	HUD.death = new("Frame", { Size = UDim2.fromOffset(460, 150), Position = UDim2.new(0.5, -230, 0.62, 0), BackgroundColor3 = COL.bg, BackgroundTransparency = 0.3,
		BorderSizePixel = 0, Visible = false, ZIndex = 15 }, hud)
	corner(HUD.death, 12)
	stroke(HUD.death, COL.red, 2)
	label(HUD.death, { Size = UDim2.fromOffset(440, 36), Position = UDim2.fromOffset(10, 10), Text = "ELIMINATED", TextSize = 30, TextColor3 = COL.red, ZIndex = 16 })
	HUD.deathBy = label(HUD.death, { Size = UDim2.fromOffset(440, 50), Position = UDim2.fromOffset(10, 52), Text = "", TextSize = 16, ZIndex = 16, TextWrapped = true })
	HUD.deathTimer = label(HUD.death, { Size = UDim2.fromOffset(440, 30), Position = UDim2.fromOffset(10, 108), Text = "", TextSize = 18, TextColor3 = COL.gold, ZIndex = 16 })
	-- ---------- MUSIC: built-in dark synth score (menu = slow and tense, battle = driving) ----------
	do
		local mg = Instance.new("SoundGroup")
		mg.Name = "BO_Music"
		mg.Volume = 1
		mg.Parent = SoundService
		new("ReverbSoundEffect", { DecayTime = 3.2, Density = 0.9, Diffusion = 0.9, DryLevel = -2, WetLevel = -6 }, mg)
		new("EqualizerSoundEffect", { LowGain = 2, MidGain = -3, HighGain = -6 }, mg)
		XT.music = { mode = "menu" }
		local function mvol()
			if S.g.musicOn == false then return 0 end
			return (S.g.musicVol or 0.5) * 1.15 * (S.g.master or 1)
		end
		local function mplay(id, speed, v, len)
			local vv = v * mvol()
			if vv <= 0.002 then return end
			local snd = Instance.new("Sound")
			snd.SoundId = id
			snd.PlaybackSpeed = speed
			snd.Volume = vv
			snd.SoundGroup = mg
			snd.Parent = SoundService
			snd:Play()
			Debris:AddItem(snd, len)
		end
		local function pitch(n) return 2 ^ (n / 12) end
		local PROG = { 0, -4, 3, -2 } -- i, VI, III, VII in minor
		local ARP = { 0, 3, 7, 10, 12, 10, 7, 3 }
		local step, nextT = 0, os.clock()
		local custom, customMode
		RunService.Heartbeat:Connect(function()
			local mode = XT.music.mode
			local list = MUSIC_IDS[mode]
			if list and #list > 0 then
				if customMode ~= mode then
					if custom then custom:Destroy() end
					custom = Instance.new("Sound")
					custom.SoundId = list[math.random(1, #list)]
					custom.Looped = true
					custom.SoundGroup = mg
					custom.Parent = SoundService
					custom:Play()
					customMode = mode
				end
				custom.Volume = mvol()
				return
			elseif custom then
				custom:Destroy()
				custom, customMode = nil, nil
			end
			local now = os.clock()
			if now - nextT > 0.5 then nextT = now end
			if mvol() <= 0.002 then return end
			local battle = mode == "battle"
			local dt = 60 / (battle and 126 or 84) / 4
			while now >= nextT do
				nextT += dt
				local i = step % 16
				local bar = math.floor(step / 16) % 4
				local root = PROG[bar + 1]
				if battle then
					if i % 4 == 0 then mplay(SND.bass, 0.75, 1.0, 0.2) end
					if i == 4 or i == 12 then mplay(SND.snap, 0.55, 0.8, 0.22) end
					if i % 2 == 0 then mplay(SND.clickfast, 3.2, i % 4 == 2 and 0.35 or 0.18, 0.08) end
					if i % 2 == 0 and i % 4 ~= 0 then mplay(SND.bass, 0.5 * pitch(root), 0.6, 0.16) end
					mplay(SND.ping, pitch(ARP[step % 8 + 1] + root), 0.1, 0.3)
					if i == 0 and bar % 2 == 0 then mplay(SND.flash, 0.6, 0.35, 0.5) end
				else
					if i == 0 then
						mplay(SND.bass, 0.5 * pitch(root), 0.9, 1.4)
						mplay(SND.bass, 0.5 * pitch(root + 7), 0.35, 1.4)
					end
					if i == 8 then mplay(SND.bass, 0.75, 0.55, 0.25) end
					if i == 4 or i == 12 then mplay(SND.clickfast, 3.0, 0.12, 0.08) end
					if i % 2 == 0 then mplay(SND.ping, pitch(ARP[(i / 2) % 8 + 1] + root), 0.09, 0.45) end
				end
				step += 1
			end
		end)
		Net.Spawned.OnClientEvent:Connect(function() XT.music.mode = "battle" end)
		Net.Results.OnClientEvent:Connect(function() XT.music.mode = "menu" end)
	end
	-- ---------- KILLCAM: replay the killer's last seconds from their eyes, with the weapon card ----------
	do
		local KC = { on = false }
		XT.kc = KC
		local kcGui = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, ZIndex = 18 }, hud)
		local topBar = new("Frame", { Size = UDim2.fromOffset(260, 40), AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 84), BackgroundColor3 = COL.bg, BackgroundTransparency = 0.25, BorderSizePixel = 0, ZIndex = 19 }, kcGui)
		corner(topBar, 4)
		stroke(topBar, COL.red, 2, 0.2)
		local recDot = new("Frame", { Size = UDim2.fromOffset(14, 14), Position = UDim2.fromOffset(14, 13), BackgroundColor3 = COL.red, BorderSizePixel = 0, ZIndex = 20 }, topBar)
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, recDot)
		label(topBar, { Size = UDim2.fromOffset(200, 40), Position = UDim2.fromOffset(40, 0), Text = "KILLCAM  -  REPLAY", TextSize = 22, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 20 })
		for _, top in ipairs({ true, false }) do
			new("Frame", { Size = UDim2.new(1, 0, 0, 70), AnchorPoint = Vector2.new(0, top and 0 or 1), Position = UDim2.fromScale(0, top and 0 or 1), BackgroundColor3 = RGB(0, 0, 0), BorderSizePixel = 0, ZIndex = 17 }, kcGui)
		end
		for _, e in ipairs({ { 90, UDim2.new(1, 0, 0.35, 0), UDim2.fromScale(0, 0), Vector2.new(0, 0) }, { 270, UDim2.new(1, 0, 0.35, 0), UDim2.fromScale(0, 1), Vector2.new(0, 1) },
			{ 0, UDim2.new(0.25, 0, 1, 0), UDim2.fromScale(0, 0), Vector2.new(0, 0) }, { 180, UDim2.new(0.25, 0, 1, 0), UDim2.fromScale(1, 0), Vector2.new(1, 0) } }) do
			local vf = new("Frame", { Size = e[2], Position = e[3], AnchorPoint = e[4], BackgroundColor3 = RGB(0, 0, 0), BorderSizePixel = 0, ZIndex = 16 }, kcGui)
			new("UIGradient", { Rotation = e[1], Transparency = NumberSequence.new(0.35, 1) }, vf)
		end
		local card = new("Frame", { Size = UDim2.fromOffset(640, 128), AnchorPoint = Vector2.new(0.5, 1), Position = UDim2.new(0.5, 0, 1, -88), BackgroundColor3 = COL.bg, BackgroundTransparency = 0.18, BorderSizePixel = 0, ZIndex = 19 }, kcGui)
		corner(card, 4)
		stroke(card, COL.accent, 2, 0.2)
		new("Frame", { Size = UDim2.fromOffset(6, 128), BackgroundColor3 = COL.accent, BorderSizePixel = 0, ZIndex = 20 }, card)
		local gv = new("ViewportFrame", { Size = UDim2.fromOffset(190, 100), Position = UDim2.fromOffset(18, 14), BackgroundColor3 = COL.panel2, BackgroundTransparency = 0.2, BorderSizePixel = 0, ZIndex = 20,
			Ambient = RGB(170, 170, 170), LightColor = RGB(255, 235, 210), LightDirection = Vector3.new(-1, -0.7, -0.6) }, card)
		local gcam = new("Camera", { FieldOfView = 28 }, gv)
		gv.CurrentCamera = gcam
		local gunShow
		label(card, { Size = UDim2.fromOffset(300, 16), Position = UDim2.fromOffset(222, 12), Text = "ELIMINATED BY", TextSize = 13, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 20 })
		local kName = label(card, { Size = UDim2.fromOffset(400, 40), Position = UDim2.fromOffset(222, 28), Text = "", TextSize = 38, Font = Enum.Font.GothamBlack, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 20, TextTruncate = Enum.TextTruncate.AtEnd })
		local kGun = label(card, { Size = UDim2.fromOffset(400, 22), Position = UDim2.fromOffset(222, 70), Text = "", TextSize = 18, TextColor3 = COL.accent2, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 20 })
		local kInfo = label(card, { Size = UDim2.fromOffset(300, 18), Position = UDim2.fromOffset(222, 96), Text = "", TextSize = 14, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 20 })
		local kHs = label(card, { Size = UDim2.fromOffset(120, 24), AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -14, 0, 14), Text = "HEADSHOT", TextSize = 16, Font = Enum.Font.GothamBlack, TextColor3 = Color3.new(1, 1, 1),
			BackgroundColor3 = COL.red, BackgroundTransparency = 0, ZIndex = 21, Visible = false })
		corner(kHs, 3)
		local kTimer = label(card, { Size = UDim2.fromOffset(160, 30), AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -14, 1, -14), Text = "", TextSize = 22, Font = Enum.Font.GothamBlack, TextColor3 = COL.accent2, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 20 })
		local prog = new("Frame", { Size = UDim2.new(1, 0, 0, 4), AnchorPoint = Vector2.new(0, 1), Position = UDim2.fromScale(0, 1), BackgroundColor3 = COL.accent, BorderSizePixel = 0, ZIndex = 21 }, card)
		local ghost, vmGun
		local hidden = {}
		local function hideWorld(killerName)
			-- the live killer and your own body are NOT part of the replay, so hide them or you see a second copy running around
			local function hideModel(m)
				if not m then return end
				for _, d in ipairs(m:GetDescendants()) do
					if d:IsA("BasePart") then
						table.insert(hidden, d)
						d.LocalTransparencyModifier = 1
					elseif d:IsA("BillboardGui") or d:IsA("Highlight") then
						table.insert(hidden, d)
						d.Enabled = false
					end
				end
			end
			hideModel(player.Character)
			for _, pl in ipairs(Players:GetPlayers()) do
				if pl ~= player and (pl.Name == killerName or pl.DisplayName == killerName) then hideModel(pl.Character) end
			end
			local bf = Workspace:FindFirstChild("BO_Bots")
			if bf then
				for _, m in ipairs(bf:GetChildren()) do
					if m.Name == killerName or m:GetAttribute("BotName") == killerName then hideModel(m) end
				end
			end
		end
		local function showWorld()
			for _, d in ipairs(hidden) do
				if d.Parent then
					if d:IsA("BasePart") then d.LocalTransparencyModifier = 0 else d.Enabled = true end
				end
			end
			hidden = {}
		end
		local function anchorAll(m)
			for _, d in ipairs(m:GetDescendants()) do
				if d:IsA("BasePart") then d.Anchored = true d.CanCollide = false d.CanQuery = false d.CanTouch = false end
			end
		end
		local function cleanup()
			if ghost then ghost:Destroy() ghost = nil end
			if vmGun then vmGun:Destroy() vmGun = nil end
			if gunShow then gunShow:Destroy() gunShow = nil end
			kcGui.Visible = false
			showWorld()
			for _, o in ipairs({ HUD.ammoBox, HUD.hpBox, HUD.opCard }) do if o then o.Visible = true end end
			if KC.on then
				KC.on = false
				camera.CameraType = Enum.CameraType.Custom
				local h = myHum()
				if h then camera.CameraSubject = h end
			end
		end
		function XT.kcStart(name, weaponName, headshot, respawn, d)
			cleanup()
			if true then return false end
			if type(d) ~= "table" or type(d.k) ~= "table" or #d.k < 4 then return false end
			KC.d, KC.t0, KC.total = d, os.clock(), tonumber(respawn) or 5
			KC.cur = nil
			KC.on = true
			kcGui.Visible = true
			pcall(hideWorld, name)
			local wd = WEAPONS[d.w or ""]
			kName.Text = string.upper(tostring(name))
			kName.TextColor3 = (d.kt == "Bravo") and COL.cyan or ((d.kt == "Alpha") and COL.orange or COL.text)
			kGun.Text = wd and (string.upper(wd.name) .. "  -  " .. string.upper(wd.class or "")) or string.upper(tostring(weaponName ~= "" and weaponName or "UNKNOWN"))
			kHs.Visible = headshot == true
			local kv, vv = d.k[#d.k], d.v[#d.v]
			local dist = vv and math.floor(((Vector3.new(kv[1], kv[2], kv[3]) - Vector3.new(vv[1], vv[2], vv[3])).Magnitude) + 0.5) or 0
			kInfo.Text = (vv and (dist .. " m  -  ") or "") .. "KILLER'S LAST " .. string.format("%.1f", (#d.k - 1) / 10) .. " SECONDS"
			if wd then
				pcall(function()
					gunShow = XT.gun(wd)
					gunShow.Parent = gv
					gunShow:PivotTo(CFrame.new(0, 0, 0))
					gcam.CFrame = CFrame.lookAt(Vector3.new(0, 0.3, 4.2), Vector3.new(0, 0, 0))
				end)
			end
			pcall(function()
				local myOp = (P.equip and P.equip.operator) or "OP_STANDARD"
				ghost = XT.soldier(myOp)
				anchorAll(ghost)
				ghost.Parent = workspace
				if wd then
					vmGun = XT.gun(wd)
					anchorAll(vmGun)
					vmGun.Parent = workspace
				end
			end)
			return true
		end
		function XT.kcStop() cleanup() end
		RunService:BindToRenderStep("BO_KILLCAM", Enum.RenderPriority.Last.Value + 2, function(dt)
			if not KC.on then return end
			local d = KC.d
			local el = os.clock() - KC.t0
			if el > KC.total + 0.5 then cleanup() return end
			camera.CameraType = Enum.CameraType.Scriptable
			camera.FieldOfView = 78
			for _, o in ipairs({ HUD.ammoBox, HUD.hpBox, HUD.opCard, HUD.cross }) do if o then o.Visible = false end end
			local n = #d.k
			local f = math.clamp((el - 0.35) / ((n - 1) * 0.1), 0, 1) * (n - 1) + 1
			local i0 = math.clamp(math.floor(f), 1, n)
			local i1 = math.min(i0 + 1, n)
			local a = f - i0
			local function samp(arr, off, idx0, idx1)
				local p0, p1 = arr[math.clamp(idx0 + off, 1, #arr)], arr[math.clamp(idx1 + off, 1, #arr)]
				return Vector3.new(p0[1] + (p1[1] - p0[1]) * a, p0[2] + (p1[2] - p0[2]) * a, p0[3] + (p1[3] - p0[3]) * a),
					Vector3.new(p0[4] + (p1[4] - p0[4]) * a, 0, p0[5] + (p1[5] - p0[5]) * a)
			end
			local kp, kl = samp(d.k, 0, i0, i1)
			local eye = kp + Vector3.new(0, 1.6, 0)
			local aimAt
			local vOff = #d.v - n
			local vp, vl
			if #d.v > 0 then vp, vl = samp(d.v, vOff, i0, i1) end
			if vp then aimAt = vp + Vector3.new(0, 1.3, 0) else aimAt = eye + kl * 10 end
			local want = CFrame.lookAt(eye, aimAt)
			KC.cur = KC.cur and KC.cur:Lerp(want, math.clamp(dt * 14, 0, 1)) or want
			camera.CFrame = KC.cur
			if ghost and vp then
				local vf = (vl.Magnitude > 0.05) and vl.Unit or Vector3.new(0, 0, -1)
				ghost:PivotTo(CFrame.lookAt(vp, vp + vf))
			end
			if vmGun then
				local rot = XT.gunHold - XT.gunHold.Position
				vmGun:PivotTo(KC.cur * CFrame.new(0.55, -0.75, -1.7) * rot)
			end
			recDot.BackgroundTransparency = (math.floor(os.clock() * 2) % 2 == 0) and 0 or 0.7
			prog.Size = UDim2.new(math.clamp(el / KC.total, 0, 1), 0, 0, 4)
			kTimer.Text = string.format("RESPAWN  %d", math.max(0, math.ceil(KC.total - el)))
			if gunShow then gunShow:PivotTo(CFrame.Angles(0, os.clock() * 0.9, 0)) end
		end)
		Net.Spawned.OnClientEvent:Connect(function() cleanup() end)
	end
	Net.Died.OnClientEvent:Connect(function(killer, weapon, headshot, respawn, kcData)
		local usedKc = false
		pcall(function() usedKc = XT.kcStart(killer ~= "" and killer or "?", weapon, headshot, respawn, kcData) end)
		HUD.deathBy.Text = string.format("Eliminated by %s\nWeapon: %s%s", tostring(killer or "?"), tostring(weapon or "?"), headshot and "  (HEADSHOT)" or "")
		HUD.deadUntil = serverNow() + (tonumber(respawn) or CFG.RESPAWN_TIME)
		HUD.death.Visible = not usedKc
		HUD.vignette.BackgroundTransparency = 1
		W.ads = false
	end)
	Net.Spawned.OnClientEvent:Connect(function(team)
		HUD.death.Visible = false
		HUD.deadUntil = 0
		HUD.vignette.BackgroundTransparency = 1
		W.reloadUntil, W.swapUntil = 0, 0
		if team and team ~= "" then
			showBanner("YOU ARE ON TEAM " .. string.upper(tostring(team)), "info")
		end
	end)

	-- vote overlay
	local voteFrame = new("Frame", { Size = UDim2.fromOffset(760, 400), Position = UDim2.fromScale(0.5, 0.5), AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = COL.bg,
		BackgroundTransparency = 0.08, Visible = false, ZIndex = 30 }, gui)
	MS.voteScale = new("UIScale", {}, voteFrame)
	local function rescaleVote()
		local vp = camera.ViewportSize
		MS.voteScale.Scale = math.clamp(math.min(vp.X / 800, vp.Y / 440), 0.4, 1.2)
	end
	rescaleVote()
	camera:GetPropertyChangedSignal("ViewportSize"):Connect(rescaleVote)
	corner(voteFrame, 14)
	stroke(voteFrame, COL.orange, 2)
	local voteTitle = label(voteFrame, { Size = UDim2.fromOffset(500, 34), Position = UDim2.fromOffset(20, 10), Text = "VOTE", TextSize = 26, ZIndex = 31, TextXAlignment = Enum.TextXAlignment.Left })
	local voteTimer = label(voteFrame, { Size = UDim2.fromOffset(200, 34), Position = UDim2.new(1, -220, 0, 10), Text = "", TextSize = 26, TextColor3 = COL.gold, ZIndex = 31,
		TextXAlignment = Enum.TextXAlignment.Right })
	label(voteFrame, { Size = UDim2.fromOffset(720, 18), Position = UDim2.fromOffset(20, 52), Text = "MAP", TextSize = 12, TextColor3 = COL.dim, ZIndex = 31, TextXAlignment = Enum.TextXAlignment.Left })
	label(voteFrame, { Size = UDim2.fromOffset(720, 18), Position = UDim2.fromOffset(20, 226), Text = "MODE", TextSize = 12, TextColor3 = COL.dim, ZIndex = 31, TextXAlignment = Enum.TextXAlignment.Left })
	local voteCards = { map = {}, mode = {} }
	local voteData
	local function makeVoteCard(kind, id, x, y, title, sub, accent)
		local b = button(voteFrame, "", UDim2.fromOffset(230, 140), UDim2.fromOffset(x, y), COL.panel2, function()
			Net.Vote:FireServer(kind, id)
		end)
		b.ZIndex = 31
		local bar = new("Frame", { Size = UDim2.new(1, 0, 0, 5), BackgroundColor3 = accent, BorderSizePixel = 0, ZIndex = 32 }, b)
		label(b, { Size = UDim2.fromOffset(210, 26), Position = UDim2.fromOffset(10, 12), Text = title, TextSize = 16, ZIndex = 32, TextXAlignment = Enum.TextXAlignment.Left })
		label(b, { Size = UDim2.fromOffset(210, 60), Position = UDim2.fromOffset(10, 40), Text = sub, TextSize = 11, TextColor3 = COL.dim, Font = Enum.Font.Gotham, ZIndex = 32,
			TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, TextWrapped = true })
		local votes = label(b, { Size = UDim2.fromOffset(210, 22), Position = UDim2.fromOffset(10, 112), Text = "0 votes", TextSize = 14, TextColor3 = COL.cyan, ZIndex = 32,
			TextXAlignment = Enum.TextXAlignment.Left })
		local st = stroke(b, COL.green, 3, 1)
		voteCards[kind][id] = { btn = b, votes = votes, stroke = st }
	end
	local voteBuilt
	local function buildVoteCards(d)
		for _, k in ipairs({ "map", "mode" }) do
			for _, c in pairs(voteCards[k]) do c.btn:Destroy() end
			voteCards[k] = {}
		end
		for i, id in ipairs(d.maps) do
			local m = MAPS[id]
			if m then makeVoteCard("map", id, 20 + (i - 1) * 245, 72, m.name, m.tag .. "\n" .. m.desc, m.accent) end
		end
		for i, id in ipairs(d.modes) do
			local m = MODES[id]
			if m then makeVoteCard("mode", id, 20 + (i - 1) * 245, 246, m.name, m.desc, COL.orange) end
		end
		voteBuilt = true
	end
	local function refreshVote()
		local d = voteData
		if not d then
			voteFrame.Visible = false
			return
		end
		local showV = phase() == "VOTE" and (d.open == true or d.result ~= nil) and player:GetAttribute("Queued") == true and not inMatch()
		voteFrame.Visible = false
		if XT.mmVote then XT.mmVote(d, showV) end
		if XT.autoVote and d.open and player:GetAttribute("Queued") == true then
			if not (d.mine and d.mine.map) then
				Net.Vote:FireServer("map", d.maps[math.random(#d.maps)])
			elseif not (d.mine and d.mine.mode) then
				local want = XT.quickMode or "TDM"
				Net.Vote:FireServer("mode", table.find(d.modes, want) and want or d.modes[1])
			end
		end
		if not voteBuilt or d.rebuild then buildVoteCards(d) end
		for _, kind in ipairs({ "map", "mode" }) do
			for id, c in pairs(voteCards[kind]) do
				local n = d.tally and d.tally[kind] and d.tally[kind][id] or 0
				c.votes.Text = n .. (n == 1 and " vote" or " votes")
				c.stroke.Transparency = (d.mine and d.mine[kind] == id) and 0 or 1
			end
		end
		if d.open and d.endsAt then voteTimer.Text = tostring(math.max(0, math.ceil(d.endsAt - serverNow()))) else voteTimer.Text = "" end
		if d.result and d.result.map then
			local mm, mo = MAPS[d.result.map], MODES[d.result.mode]
			voteTitle.Text = string.format("SELECTED:  %s  -  %s", mm and mm.name or "?", mo and mo.name or "?")
		else
			voteTitle.Text = "VOTE FOR MAP + MODE"
		end
	end
	Net.VoteState.OnClientEvent:Connect(function(d)
		if type(d) ~= "table" or type(d.maps) ~= "table" or type(d.modes) ~= "table" then return end
		if not voteData or #voteData.maps ~= #d.maps or voteData.maps[1] ~= d.maps[1] or voteData.modes[1] ~= d.modes[1] then voteBuilt = false end
		voteData = d
		refreshVote()
	end)

	-- =================================================================
	-- RESULTS SCREEN
	-- =================================================================
	local res = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = COL.bg, BackgroundTransparency = 0.15, Visible = false, ZIndex = 40 }, gui)
	do
	local resRoot = new("Frame", { Size = UDim2.fromOffset(900, 600), Position = UDim2.fromScale(0.5, 0.5), AnchorPoint = Vector2.new(0.5, 0.5), BackgroundTransparency = 1,
		ZIndex = 41 }, res)
	RT.resScale = new("UIScale", {}, resRoot)
	local function rescaleRes()
		local vp = camera.ViewportSize
		RT.resScale.Scale = math.clamp(math.min(vp.X / 940, vp.Y / 640), 0.4, 1.25)
	end
	rescaleRes()
	camera:GetPropertyChangedSignal("ViewportSize"):Connect(rescaleRes)
	local resTitle = label(resRoot, { Size = UDim2.fromOffset(900, 70), Position = UDim2.fromOffset(0, 0), Text = "VICTORY", TextSize = 56, ZIndex = 42 })
	RT.resSub = label(resRoot, { Size = UDim2.fromOffset(900, 24), Position = UDim2.fromOffset(0, 70), Text = "", TextSize = 16, TextColor3 = COL.dim, ZIndex = 42 })
	local resStats = new("Frame", { Size = UDim2.fromOffset(280, 360), Position = UDim2.fromOffset(0, 110), BackgroundColor3 = COL.panel, BorderSizePixel = 0, ZIndex = 42 }, resRoot)
	corner(resStats, 12)
	local resStatLabels = {}
	for i, nm in ipairs({ "KILLS", "DEATHS", "ASSISTS", "HEADSHOTS", "XP EARNED" }) do
		label(resStats, { Size = UDim2.fromOffset(130, 30), Position = UDim2.fromOffset(16, 14 + (i - 1) * 44), Text = nm, TextSize = 14, TextColor3 = COL.dim, ZIndex = 43,
			TextXAlignment = Enum.TextXAlignment.Left })
		resStatLabels[i] = label(resStats, { Size = UDim2.fromOffset(110, 30), Position = UDim2.fromOffset(150, 14 + (i - 1) * 44), Text = "0", TextSize = 24, TextColor3 = COL.cyan, ZIndex = 43,
			TextXAlignment = Enum.TextXAlignment.Right })
	end
	RT.resLvl = label(resStats, { Size = UDim2.fromOffset(250, 26), Position = UDim2.fromOffset(16, 240), Text = "LEVEL 1", TextSize = 20, TextColor3 = COL.gold, ZIndex = 43,
		TextXAlignment = Enum.TextXAlignment.Left })
	local resBarBg = new("Frame", { Size = UDim2.fromOffset(248, 14), Position = UDim2.fromOffset(16, 274), BackgroundColor3 = COL.bg, BorderSizePixel = 0, ZIndex = 43 }, resStats)
	corner(resBarBg, 7)
	local resBar = new("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = COL.orange, BorderSizePixel = 0, ZIndex = 44 }, resBarBg)
	corner(resBar, 7)
	RT.resXp = label(resStats, { Size = UDim2.fromOffset(250, 20), Position = UDim2.fromOffset(16, 294), Text = "", TextSize = 12, TextColor3 = COL.dim, ZIndex = 43,
		TextXAlignment = Enum.TextXAlignment.Left })
	RT.resUnlock = label(resStats, { Size = UDim2.fromOffset(250, 40), Position = UDim2.fromOffset(16, 316), Text = "", TextSize = 12, TextColor3 = COL.green, ZIndex = 43,
		TextXAlignment = Enum.TextXAlignment.Left, TextWrapped = true })
	local resBoard = new("Frame", { Size = UDim2.fromOffset(600, 360), Position = UDim2.fromOffset(300, 110), BackgroundColor3 = COL.panel, BorderSizePixel = 0, ZIndex = 42 }, resRoot)
	corner(resBoard, 12)
	local resList = new("Frame", { Size = UDim2.fromOffset(570, 336), Position = UDim2.fromOffset(15, 12), BackgroundTransparency = 1, ZIndex = 43 }, resBoard)
	new("UIListLayout", { Padding = UDim.new(0, 3), SortOrder = Enum.SortOrder.LayoutOrder }, resList)
	local resContinue = button(resRoot, "CONTINUE", UDim2.fromOffset(300, 56), UDim2.fromOffset(300, 500), COL.orange, function()
		res.Visible = false
		HUD.results = nil
		UI.refresh()
	end, 22)
	resContinue.ZIndex = 42
	resContinue.TextColor3 = RGB(10, 10, 10)

	Net.Results.OnClientEvent:Connect(function(r)
		if type(r) ~= "table" then return end
		HUD.results = r
		HUD.death.Visible = false
		HUD.sbFrame.Visible = false
		W.ads = false
		if r.draw then
			resTitle.Text, resTitle.TextColor3 = "DRAW", COL.dim
		elseif r.won then
			resTitle.Text, resTitle.TextColor3 = "VICTORY", COL.green
			playSound(SND.victory, 1.0, 0.6 * S.g.sfx)
		else
			resTitle.Text, resTitle.TextColor3 = "DEFEAT", COL.red
		end
		RT.resSub.Text = string.format("%s  -  %s  -  %s", tostring(r.winner or ""), tostring(r.mode or ""), tostring(r.map or ""))
		local vals = { r.kills, r.deaths, r.assists, r.headshots, r.xp }
		for i, l in ipairs(resStatLabels) do l.Text = tostring(vals[i] or 0) end
		RT.resLvl.Text = "LEVEL " .. tostring(r.lvl or 1) .. ((r.lvl or 1) > (r.lvlBefore or 1) and "   LEVEL UP!" or "")
		resBar.Size = UDim2.fromScale(0, 1)
		tween(resBar, 1.2, { Size = UDim2.fromScale(math.clamp((r.into or 0) / math.max(1, r.need or 1), 0, 1), 1) })
		RT.resXp.Text = string.format("%d / %d XP to next level", r.into or 0, r.need or 0)
		local ups = {}
		for _, id in ipairs(PRIMARY_ORDER) do
			local d = WEAPONS[id]
			if d.unlock > (r.lvlBefore or 1) and d.unlock <= (r.lvl or 1) then table.insert(ups, d.name) end
		end
		for _, id in ipairs(SECONDARY_ORDER) do
			local d = WEAPONS[id]
			if d.unlock > (r.lvlBefore or 1) and d.unlock <= (r.lvl or 1) then table.insert(ups, d.name) end
		end
		RT.resUnlock.Text = (#ups > 0) and ("NEW WEAPON UNLOCKED: " .. table.concat(ups, ", ")) or ""
		fillRows(resList, r.rows or {}, r.you, 10, 44)
		res.Visible = true
		UI.refresh()
	end)
	end

	-- =================================================================
	-- MENU / HUD VISIBILITY + HUB CAMERA
	-- =================================================================
	function UI.blocking()
		return menuHolder.Visible or res.Visible or editorFrame.Visible
	end
	function UI.refresh()
		local match = inMatch()
		local resultsOpen = res.Visible
		local showMenu
		if resultsOpen then
			showMenu = false
		elseif match then
			showMenu = UI.menuOpen and UI.inGameMenu == true
		else
			showMenu = not UI.explore
		end
		menuHolder.Visible = showMenu
		exploreBtn.Visible = not match
		hud.Visible = match and not resultsOpen
		if match and showMenu then showPage("SETTINGS") end
		if not match then UI.inGameMenu = false end
		refreshDeploy()
		refreshVote()
		refreshLighting(false)
	end
	player:GetAttributeChangedSignal("InMatch"):Connect(function()
		if inMatch() then
			UI.explore = false
			UI.inGameMenu = false
			UI.menuOpen = false
			res.Visible = false
		end
		UI.refresh()
	end)
	player:GetAttributeChangedSignal("Queued"):Connect(refreshVote)
	-- the vote overlay can never outlive the VOTE phase
	State:GetAttributeChangedSignal("Phase"):Connect(function()
		if phase() ~= "VOTE" then voteFrame.Visible = false end
		pcall(refreshVote)
	end)
	player:GetAttributeChangedSignal("InMatch"):Connect(function()
		if inMatch() then voteFrame.Visible = false end
	end)
	task.spawn(function()
		while true do
			task.wait(0.3)
			if voteFrame.Visible and (phase() ~= "VOTE" or inMatch()) then voteFrame.Visible = false end
		end
	end)
	State:GetAttributeChangedSignal("MapId"):Connect(function() refreshLighting(false) end)

	function UI.toggleMenu()
		if res.Visible then return end
		if inMatch() then
			UI.inGameMenu = not UI.inGameMenu
			UI.menuOpen = UI.inGameMenu
		else
			UI.explore = not UI.explore
		end
		UI.refresh()
	end

	-- =================================================================
	-- WEAPON + MOVEMENT CONTROLS (all keys come from the rebindable S table)
	-- =================================================================
	local Held = { MoveF = false, MoveB = false, MoveL = false, MoveR = false, Jump = false, Sprint = false, Fire = false }
	local R = { pitch = 0, yaw = 0 }            -- visual+aim recoil offset (degrees)
	local VM = { model = nil, id = nil, kick = 0, bob = 0, swapAt = 0, meleeAt = 0, flash = nil, muzzle = nil }
	local aimCF = CFrame.new()
	local touchToggles = { ADS = false, Sprint = false }

	local function viewmodelRebuild()
		if VM.model then VM.model:Destroy() end
		VM.model, VM.flash, VM.muzzle = nil, nil, nil
		local def = curDef()
		if not def then return end
		def = camoDef(def, P.equip and P.equip.camo)
		local m = buildGunModel(def, true)
		for _, p in ipairs(m:GetDescendants()) do
			if p:IsA("BasePart") then
				p.CastShadow = false
				p.CanQuery = false
			end
		end
		local muz = m:FindFirstChild("Muzzle")
		if muz then
			VM.muzzle = muz
			VM.flash = new("PointLight", { Color = def.color, Brightness = 4, Range = 10, Enabled = false }, muz)
		end
		m.Parent = camera
		VM.model, VM.id = m, def.id
		VM.swapAt = os.clock()
	end

	local function updateAmmoHud()
		local def = curDef()
		local a = W.ammo[W.slot]
		if not def or not a then return end
		HUD.wName.Text = def.name
		hudSetGun(HUD.gunMain, def.id)
		for i = 1, 2 do
			local sb = HUD.slotBox[i]
			if sb then
				hudSetGun(sb.icon, W.ids[i])
				sb.st.Color = (W.slot == i) and COL.accent2 or COL.line
				sb.st.Thickness = (W.slot == i) and 2 or 1
				sb.f.BackgroundColor3 = (W.slot == i) and RGB(56, 44, 16) or RGB(24, 26, 26)
			end
		end
		HUD.mag.Text = tostring(a[1])
		HUD.mag.TextColor3 = (a[1] <= math.max(2, def.mag * 0.2)) and COL.red or COL.text
		HUD.res.Text = "/ " .. tostring(a[2])
		HUD.slotTxt.Text = W.slot == 1 and "PRIMARY" or "SIDEARM"
	end

	local function startReload()
		local def, a = curDef(), W.ammo[W.slot]
		local now = os.clock()
		if not def or not a or not isAlive() then return end
		if now < W.reloadUntil or now < W.swapUntil then return end
		if a[1] >= def.mag or a[2] <= 0 then return end
		local dur = (a[1] == 0) and def.reloadEmpty or def.reload
		W.reloadUntil = now + dur
		W.reloadStart = now
		W.reloadDur = dur
		W.burstQ = 0
		W.ads = false
		Net.Reload:FireServer()
		playSound(SND.clickfast, 0.8, 0.5 * S.g.sfx)
		task.delay(dur * 0.45, function()
			if W.reloadUntil > os.clock() then playSound(SND.click2, 0.9, 0.5 * S.g.sfx) end
		end)
		task.delay(dur * 0.92, function()
			if W.reloadUntil > os.clock() then playSound(SND.click, 1.2, 0.5 * S.g.sfx) end
		end)
	end

	local function switchSlot(slot)
		local now = os.clock()
		if slot == W.slot or not isAlive() or now < W.swapUntil then return end
		W.slot = slot
		W.swapUntil = now + curDef().switch
		W.reloadUntil = 0
		W.burstQ = 0
		W.ads = false
		Net.Swap:FireServer(slot)
		playSound(SND.unsheath, 1.6, 0.35 * S.g.sfx)
		viewmodelRebuild()
		updateAmmoHud()
	end
	switchSlotRef = switchSlot

	local function doMelee()
		if not isAlive() then return end
		if os.clock() - VM.meleeAt < CFG.MELEE_COOLDOWN then return end
		VM.meleeAt = os.clock()
		Net.Melee:FireServer()
		playSound(SND.slash, 1.1, 0.5 * S.g.sfx)
	end

	local EQ = { throwAt = 0, stimAt = 0 }
	function EQ.grenade()
		if not isAlive() or (player:GetAttribute("Frags") or 0) <= 0 then return end
		if os.clock() - EQ.throwAt < 1.1 or EQ.anim then return end
		EQ.throwAt = os.clock()
		EQ.begin("grenade") -- the grenade itself is released (and sent to the server) mid-animation
		if EQ.anim then EQ.anim.held = true end -- held key = aim (up to 5 s); letting go throws
	end
	function EQ.grenadeRelease()
		local a = EQ.anim
		if a and a.kind == "grenade" then a.held = false end
	end
	function EQ.stim()
		if not isAlive() or (player:GetAttribute("Stims") or 0) <= 0 then return end
		if os.clock() - EQ.stimAt < 2.6 or EQ.anim then return end
		EQ.stimAt = os.clock()
		EQ.begin("stim")
	end

	-- ---------- first-person equipment animations: grenade pull-pin + throw, stim jab ----------
	do
		local function K(t, x, y, z, rx, rz) return { t = t, pos = Vector3.new(x, y, z), rx = rx, rz = rz } end
		local KEYS = {
			grenade = { dur = 1.05, K(0, 0.95, -1.65, -0.6, 12, 0), K(0.25, 0.55, -0.6, -1.2, -8, 0), K(0.38, 0.5, -0.55, -1.15, -8, 6), K(0.5, 0.8, -0.2, -0.75, -14, -6),
				K(0.62, 0.75, -0.05, -1.25, 8, -2), K(0.74, 0.35, 0.18, -2.0, -32, -6), K(0.88, 0.4, -0.05, -1.75, -14, 0), K(1, 0.95, -1.65, -0.6, 12, 0) },
			stim = { dur = 1.25, K(0, 0.95, -1.65, -0.6, 12, 0), K(0.22, 0.6, -0.55, -1.15, -5, 0), K(0.36, 0.55, -0.5, -1.1, -5, 0), K(0.52, -0.1, -0.85, -1.0, -55, 20),
				K(0.62, -0.3, -0.98, -0.95, -62, 24), K(0.82, -0.3, -0.98, -0.95, -62, 24), K(1, 0.95, -1.65, -0.6, 12, 0) },
		}
		-- Catmull-Rom style cubic through the keys: continuous velocity, so the motion flows instead of stopping at each key
		local function sample(keys, p)
			local n = #keys
			for i = 2, n do
				if p <= keys[i].t or i == n then
					local k0, k1 = keys[i - 1], keys[i]
					local km, kp = keys[i - 2] or k0, keys[i + 1] or k1
					local dt = k1.t - k0.t
					local f = math.clamp((p - k0.t) / dt, 0, 1)
					local f2, f3 = f * f, f * f * f
					local h00, h10, h01, h11 = 2 * f3 - 3 * f2 + 1, f3 - 2 * f2 + f, -2 * f3 + 3 * f2, f3 - f2
					local function herm(v0, v1, vm, vp)
						local m0 = (v1 - vm) / ((k1.t - km.t) > 0 and (k1.t - km.t) or 1)
						local m1 = (vp - v0) / ((kp.t - k0.t) > 0 and (kp.t - k0.t) or 1)
						return v0 * h00 + m0 * dt * h10 + v1 * h01 + m1 * dt * h11
					end
					return herm(k0.pos, k1.pos, km.pos, kp.pos), herm(k0.rx, k1.rx, km.rx, kp.rx), herm(k0.rz, k1.rz, km.rz, kp.rz)
				end
			end
			return keys[n].pos, keys[n].rx, keys[n].rz
		end
		EQ.sample = sample
		local aimFx = { beads = {}, mark = nil }
		local function hideAim()
			for _, b in ipairs(aimFx.beads) do b.Transparency = 1 end
			if aimFx.mark then aimFx.mark.Transparency = 1 end
		end
		local function drawAim(frac)
			if #aimFx.beads == 0 then
				for i = 1, 36 do
					aimFx.beads[i] = new("Part", { Name = "AimBead", Shape = Enum.PartType.Ball, Size = Vector3.new(0.28, 0.28, 0.28), Anchored = true, CanCollide = false,
						CanQuery = false, CanTouch = false, CastShadow = false, Material = Enum.Material.Neon, Transparency = 1 }, fxFolder)
				end
				aimFx.mark = new("Part", { Name = "AimMark", Shape = Enum.PartType.Cylinder, Size = Vector3.new(0.12, 3.4, 3.4), Anchored = true, CanCollide = false,
					CanQuery = false, CanTouch = false, CastShadow = false, Material = Enum.Material.Neon, Transparency = 1 }, fxFolder)
			end
			local cf = camera.CFrame
			local char = player.Character
			local hrp = char and char:FindFirstChild("HumanoidRootPart")
			local pos = cf.Position + cf.LookVector * 2.4
			local vel = cf.LookVector * 62 + Vector3.new(0, 14, 0) + (hrp and hrp.AssemblyLinearVelocity * 0.5 or Vector3.zero)
			local g = Vector3.new(0, -Workspace.Gravity, 0)
			local rp = RaycastParams.new()
			rp.FilterType = Enum.RaycastFilterType.Exclude
			rp.FilterDescendantsInstances = { char, camera, fxFolder }
			local col = Color3.new(1, 1, 1):Lerp(RGB(255, 60, 40), frac)
			local dt, n, hitPos, hitN = 0.045, 0, nil, nil
			for i = 1, 36 do
				local nv = vel + g * dt
				local np = pos + vel * dt + g * (0.5 * dt * dt)
				local r = Workspace:Raycast(pos, np - pos, rp)
				if r then hitPos, hitN = r.Position, r.Normal break end
				pos, vel = np, nv
				n = i
				local b = aimFx.beads[i]
				b.Position, b.Color, b.Transparency = pos, col, 0.15
			end
			for i = n + 1, 36 do aimFx.beads[i].Transparency = 1 end
			local m = aimFx.mark
			if hitPos then
				m.CFrame = CFrame.lookAt(hitPos + hitN * 0.1, hitPos + hitN) * CFrame.Angles(0, math.rad(90), 0)
				m.Color, m.Transparency = col, 0.35
			else
				m.Transparency = 1
			end
		end
		local function stop()
			if EQ.anim and EQ.anim.model then EQ.anim.model:Destroy() end
			EQ.anim = nil
			hideAim()
		end
		function EQ.begin(kind)
			stop()
			local now = os.clock()
			local def = KEYS[kind]
			local m = Instance.new("Model")
			m.Name = "BO_EquipVM"
			local list = {}
			local function pr(name, size, rel, col, mat, shape)
				local x = Instance.new("Part")
				x.Name, x.Size, x.Color, x.Material = name, size, col, mat or Enum.Material.SmoothPlastic
				x.Anchored, x.CanCollide, x.CanQuery, x.CanTouch, x.CastShadow = true, false, false, false, false
				if shape then x.Shape = shape end
				x.Parent = m
				local e = { p = x, rel = rel }
				table.insert(list, e)
				return e
			end
			local op = P.equip and OPERATORS[P.equip.operator]
			local suit = (op and op.suit) or RGB(70, 78, 64)
			local glove = RGB(26, 28, 32)
			-- arm: forearm + cuff + glove (+ thumb)
			local arm = {}
			arm.upper = pr("UpperArm", Vector3.new(0.44, 0.44, 1.6), CFrame.new(), suit)
			arm.fore = pr("Forearm", Vector3.new(0.38, 0.38, 1.6), CFrame.new(), suit)
			arm.elbow = pr("Elbow", Vector3.new(0.46, 0.46, 0.46), CFrame.new(), suit, nil, Enum.PartType.Ball)
			for _, k in ipairs({ "upper", "fore", "elbow" }) do table.remove(list) end
			pr("Cuff", Vector3.new(0.4, 0.4, 0.18), CFrame.new(0.03, -0.03, 0.3), glove:Lerp(suit, 0.3))
			pr("Glove", Vector3.new(0.44, 0.34, 0.52), CFrame.new(0, 0, 0), glove)
			pr("Thumb", Vector3.new(0.14, 0.14, 0.34), CFrame.new(-0.22, 0.1, -0.12) * CFrame.Angles(0, math.rad(12), 0), glove)
			local item = {}
			if kind == "grenade" then
				item.body = pr("Grenade", Vector3.new(0.58, 0.7, 0.58), CFrame.new(0, 0.34, -0.12), RGB(78, 92, 66), Enum.Material.Metal, Enum.PartType.Ball)
				pr("GrenadeCap", Vector3.new(0.26, 0.16, 0.26), CFrame.new(0, 0.7, -0.12), RGB(190, 195, 205), Enum.Material.Metal)
				item.spoon = pr("GrenadeSpoon", Vector3.new(0.08, 0.5, 0.16), CFrame.new(0.28, 0.42, -0.12), RGB(170, 175, 185), Enum.Material.Metal)
				item.ring = pr("GrenadePin", Vector3.new(0.04, 0.24, 0.24), CFrame.new(0, 0.82, -0.12), RGB(235, 235, 240), Enum.Material.Metal, Enum.PartType.Cylinder)
			else
				local face = CFrame.Angles(0, math.rad(90), 0)
				local base = CFrame.new(0, 0.3, -0.5)
				item.barrel = pr("SyringeBarrel", Vector3.new(0.9, 0.27, 0.27), base * face, RGB(190, 230, 255), Enum.Material.Glass, Enum.PartType.Cylinder)
				item.barrel.p.Transparency = 0.3
				item.liquid = pr("SyringeLiquid", Vector3.new(0.7, 0.18, 0.18), base * face, RGB(80, 255, 150), Enum.Material.Neon, Enum.PartType.Cylinder)
				item.rod = pr("SyringeRod", Vector3.new(0.55, 0.07, 0.07), base * CFrame.new(0, 0, 0.6) * face, RGB(210, 214, 222), Enum.Material.Metal, Enum.PartType.Cylinder)
				item.plate = pr("SyringePlate", Vector3.new(0.05, 0.32, 0.32), base * CFrame.new(0, 0, 0.9) * face, RGB(60, 64, 72), nil, Enum.PartType.Cylinder)
				item.needle = pr("SyringeNeedle", Vector3.new(0.42, 0.04, 0.04), base * CFrame.new(0, 0, -0.68) * face, RGB(220, 224, 232), Enum.Material.Metal, Enum.PartType.Cylinder)
				item.cap = pr("SyringeCap", Vector3.new(0.5, 0.1, 0.1), base * CFrame.new(0, 0, -0.7) * face, RGB(255, 140, 30), nil, Enum.PartType.Cylinder)
				item.base = base
				item.face = face
			end
			m.Parent = camera
			Debris:AddItem(m, def.dur + 0.4) -- safety: never leave the arm on screen if the viewmodel step stops (death, menu)
			EQ.anim = { kind = kind, t0 = now, dur = def.dur, model = m, list = list, item = item, arm = arm, fired = false, pinned = false }
			W.ads = false
			touchToggles.ADS = false
			W.burstQ = 0
			W.swapUntil = math.max(W.swapUntil or 0, now + def.dur * 0.72)
			playSound(SND.click2, kind == "grenade" and 0.8 or 1.2, 0.5 * S.g.sfx)
		end
		-- called every frame from the viewmodel step; returns how far the gun should be lowered (0..1)
		function EQ.update(now)
			local a = EQ.anim
			if not a then return 0 end
			if not isAlive() or not a.model.Parent then stop() return 0 end
			if a.kind == "grenade" and not a.fired and a.held ~= nil then
				-- hold the pose (pin pulled, arm cocked) while the key is down, up to 5 s, with a live trajectory preview
				local hp = 0.5 * a.dur
				if a.held and now - a.t0 >= hp then
					a.holdAt = a.holdAt or now
					local waited = now - a.holdAt
					if waited < 5 then
						a.t0 = now - hp
						drawAim(waited / 5)
					else
						a.held = false
					end
				elseif not a.held then
					hideAim()
				end
			end
			local p = (now - a.t0) / a.dur
			if p >= 1 then stop() return 0 end
			local pos, rx, rz = sample(KEYS[a.kind], p)
			local hand = camera.CFrame * CFrame.new(pos) * CFrame.Angles(math.rad(rx), 0, math.rad(rz))
			local it = a.item
			if a.kind == "grenade" then
				if p > 0.4 and not a.pinned then
					a.pinned = true
					playSound(SND.ping, 1.6, 0.4 * S.g.sfx)
				end
				local rp = math.max(0, p - 0.42)
				it.ring.rel = (p < 0.42) and CFrame.new(0, 0.82, -0.12) or CFrame.new(0.1 + rp * 2, 0.82 - rp * 5, -0.12 + rp * 2.4) * CFrame.Angles(rp * 12, 0, 0)
				it.ring.p.Transparency = (p > 0.66) and 1 or 0
				local held = p < 0.7
				for _, e in ipairs(a.list) do
					if e.p.Name:sub(1, 7) == "Grenade" and e.p.Name ~= "GrenadePin" then e.p.Transparency = held and 0 or 1 end
				end
				if not a.fired and p >= 0.7 then
					a.fired = true
					local cf = camera.CFrame
					Net.Throw:FireServer(cf.Position + cf.LookVector * 1.2, cf.LookVector)
					playSound(SND.lunge, 1.5, 0.45 * S.g.sfx)
				end
			else
				local f = it.face
				local capGone = p > 0.34
				local cp = math.max(0, p - 0.34)
				it.cap.rel = capGone and (it.base * CFrame.new(0.2 * cp * 4, -cp * 3, -0.7 - cp * 2) * f) or (it.base * CFrame.new(0, 0, -0.7) * f)
				it.cap.p.Transparency = (p > 0.6) and 1 or 0
				local push = math.clamp((p - 0.6) / 0.22, 0, 1)
				local len = 0.7 * (1 - push * 0.85)
				it.liquid.p.Size = Vector3.new(math.max(0.05, len), 0.18, 0.18)
				it.liquid.rel = it.base * CFrame.new(0, 0, -0.35 + len / 2 + 0.0) * f
				it.rod.rel = it.base * CFrame.new(0, 0, 0.6 - push * 0.4) * f
				it.plate.rel = it.base * CFrame.new(0, 0, 0.9 - push * 0.4) * f
				if not a.fired and p >= 0.58 then
					a.fired = true
					Net.Stim:FireServer()
					playSound(SND.ping, 0.7, 0.6 * S.g.sfx)
				end
				if push > 0 and push < 1 then it.liquid.p.Color = RGB(80, 255, 150):Lerp(RGB(255, 255, 255), push * 0.5) end
			end
			for _, e in ipairs(a.list) do e.p.CFrame = hand * e.rel end
			-- two-bone arm from an off-screen shoulder to the wrist, so the elbow bends naturally and the arm never stretches
			local L1, L2 = 1.45, 1.45
			local cam = camera.CFrame
			local S = cam:PointToWorldSpace(Vector3.new(0.85, -1.45, 0.45))
			local T = (hand * CFrame.new(0, -0.02, 0.3)).Position
			local v = T - S
			local d = v.Magnitude
			local maxd = (L1 + L2) * 0.97
			if d > maxd then T = S + v.Unit * maxd d = maxd end
			if d < 0.3 then d = 0.3 end
			local dir = (T - S).Unit
			local aa = (L1 * L1 - L2 * L2 + d * d) / (2 * d)
			local hh = math.sqrt(math.max(0, L1 * L1 - aa * aa))
			local pole = cam:VectorToWorldSpace(Vector3.new(0.5, -1, 0.35))
			pole = pole - dir * pole:Dot(dir)
			pole = (pole.Magnitude > 1e-3) and pole.Unit or Vector3.yAxis
			local E = S + dir * aa + pole * hh
			a.arm.upper.p.CFrame = CFrame.lookAt((S + E) / 2, E)
			a.arm.fore.p.CFrame = CFrame.lookAt((E + T) / 2, T)
			a.arm.elbow.p.CFrame = CFrame.new(E)
			local q = math.clamp(math.min(p / 0.2, (1 - p) / 0.2), 0, 1)
			return q * q * (3 - 2 * q)
		end
	end
	function EQ.boosted() return Workspace:GetServerTimeNow() < (player:GetAttribute("StimUntil") or 0) end

	-- ---------- scorestreak: AIRSTRIKE (earned with 2 kills): phone animation, then a top-down guide on the phone ----------
	local RL, FXS = {}, {}
	local AS = { active = false, phase = nil, hide = 0 }
	do
		local TAP_END, GUIDE_MAX = 2.25, 2.9
		local function K(t, x, y, z, rx, rz) return { t = t, pos = Vector3.new(x, y, z), rx = rx, rz = rz } end
		local HAND = { K(0, 0.95, -1.75, -0.5, 20, 0), K(0.55, 0.5, -1.15, -1.2, 6, 0), K(1.25, 0.35, -1.0, -1.5, 10, 0), K(2.5, 0.35, -1.0, -1.5, 10, 0) }
		local PUT = { K(0, 0.35, -1.0, -1.5, 10, 0), K(0.35, 0.95, -1.75, -0.5, 20, 0) }
		local PH = CFrame.new(0, 0.62, 0.12) * CFrame.Angles(math.rad(-15), 0, 0)
		local red = RGB(235, 36, 36)

		-- two-bone arm: keeps the elbow bent so the arm never over-stretches
		local function ik(S0, T0, pole, upper, fore, elbow)
			local L1, L2 = 1.45, 1.45
			local v = T0 - S0
			local d = v.Magnitude
			if d < 0.05 then return end
			local maxd = (L1 + L2) * 0.97
			if d > maxd then T0 = S0 + v.Unit * maxd d = maxd end
			if d < 0.3 then d = 0.3 end
			local dir = (T0 - S0).Unit
			local aa = (L1 * L1 - L2 * L2 + d * d) / (2 * d)
			local hh = math.sqrt(math.max(0, L1 * L1 - aa * aa))
			pole = pole - dir * pole:Dot(dir)
			pole = (pole.Magnitude > 1e-3) and pole.Unit or Vector3.yAxis
			local E = S0 + dir * aa + pole * hh
			upper.CFrame = CFrame.lookAt((S0 + E) / 2, E)
			fore.CFrame = CFrame.lookAt((E + T0) / 2, T0)
			elbow.CFrame = CFrame.new(E)
		end

		AS.ik = ik
		local function build()
			local m = Instance.new("Model")
			m.Name = "BO_PhoneVM"
			local o = {}
			local function pr(name, size, col, mat, shape)
				local x = Instance.new("Part")
				x.Name, x.Size, x.Color, x.Material = name, size, col, mat or Enum.Material.SmoothPlastic
				x.Anchored, x.CanCollide, x.CanQuery, x.CanTouch, x.CastShadow = true, false, false, false, false
				if shape then x.Shape = shape end
				x.Parent = m
				return x
			end
			local op = P.equip and OPERATORS[P.equip.operator]
			local suit = (op and op.suit) or RGB(70, 78, 64)
			local glove = RGB(26, 28, 32)
			local cuffCol = glove:Lerp(suit, 0.3)
			for _, side in ipairs({ "R", "L" }) do
				o["upper" .. side] = pr("Upper" .. side, Vector3.new(0.44, 0.44, 1.6), suit)
				o["fore" .. side] = pr("Fore" .. side, Vector3.new(0.38, 0.38, 1.6), suit)
				o["elbow" .. side] = pr("Elbow" .. side, Vector3.new(0.46, 0.46, 0.46), suit, nil, Enum.PartType.Ball)
			end
			o.glove = pr("GloveR", Vector3.new(0.44, 0.34, 0.52), glove)
			o.cuff = pr("CuffR", Vector3.new(0.4, 0.4, 0.18), cuffCol)
			o.thumb = pr("ThumbR", Vector3.new(0.14, 0.14, 0.34), glove)
			o.glove2 = pr("GloveL", Vector3.new(0.4, 0.3, 0.5), glove)
			o.cuff2 = pr("CuffL", Vector3.new(0.38, 0.38, 0.18), cuffCol)
			o.finger = pr("Finger", Vector3.new(0.13, 0.13, 0.46), glove)
			o.body = pr("Phone", Vector3.new(0.78, 1.36, 0.09), RGB(16, 17, 21), Enum.Material.Metal)
			o.screen = pr("Screen", Vector3.new(0.68, 1.24, 0.02), RGB(8, 10, 16))
			local sg = new("SurfaceGui", { Face = Enum.NormalId.Back, SizingMode = Enum.SurfaceGuiSizingMode.FixedSize,
				CanvasSize = Vector2.new(300, 547), LightInfluence = 0, Brightness = 1.5 }, o.screen)
			local bg = new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(8, 10, 16), BorderSizePixel = 0 }, sg)
			label(bg, { Size = UDim2.new(1, -24, 0, 26), Position = UDim2.fromOffset(12, 14), Text = "BLACKOUT NET", TextSize = 20, TextColor3 = COL.dim, TextXAlignment = Enum.TextXAlignment.Left })
			o.stat = label(bg, { Size = UDim2.new(1, -24, 0, 40), Position = UDim2.fromOffset(12, 60), Text = "STRIKE SUPPORT ONLINE", TextSize = 20, TextColor3 = COL.green })
			o.ring = new("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(150, 273), Size = UDim2.fromOffset(230, 230), BackgroundTransparency = 1 }, bg)
			corner(o.ring, 115)
			stroke(o.ring, RGB(255, 60, 60), 4, 0.4)
			o.btn = new("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(150, 273), Size = UDim2.fromOffset(180, 180), BackgroundColor3 = red, BorderSizePixel = 0 }, bg)
			corner(o.btn, 90)
			stroke(o.btn, RGB(255, 255, 255), 5, 0.35)
			o.btxt = label(o.btn, { Size = UDim2.fromScale(1, 1), Text = "AIRSTRIKE", TextSize = 24, TextColor3 = RGB(255, 255, 255) })
			o.sub = label(bg, { Size = UDim2.new(1, -24, 0, 30), Position = UDim2.fromOffset(12, 470), Text = "TAP TO CALL IN STRIKE", TextSize = 18, TextColor3 = COL.dim })
			m.Parent = camera
			Debris:AddItem(m, 14)
			return m, o
		end

		-- ---- guide overlay (looks like the phone screen) ----
		local ui = new("TextButton", { Name = "BO_StrikeUI", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "", AutoButtonColor = false,
			Active = true, Visible = false, ZIndex = 60 }, gui)
		new("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = RGB(255, 30, 30), BackgroundTransparency = 0.93, BorderSizePixel = 0, ZIndex = 61 }, ui)
		local bez = RGB(6, 7, 10)
		new("Frame", { Size = UDim2.new(1, 0, 0, 56), BackgroundColor3 = bez, BorderSizePixel = 0, ZIndex = 62 }, ui)
		new("Frame", { AnchorPoint = Vector2.new(0, 1), Position = UDim2.fromScale(0, 1), Size = UDim2.new(1, 0, 0, 56), BackgroundColor3 = bez, BorderSizePixel = 0, ZIndex = 62 }, ui)
		new("Frame", { Size = UDim2.new(0, 34, 1, 0), BackgroundColor3 = bez, BorderSizePixel = 0, ZIndex = 62 }, ui)
		new("Frame", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromScale(1, 0), Size = UDim2.new(0, 34, 1, 0), BackgroundColor3 = bez, BorderSizePixel = 0, ZIndex = 62 }, ui)
		new("Frame", { Position = UDim2.fromOffset(34, 56), Size = UDim2.new(1, -68, 0, 2), BackgroundColor3 = red, BorderSizePixel = 0, ZIndex = 63 }, ui)
		new("Frame", { AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 34, 1, -56), Size = UDim2.new(1, -68, 0, 2), BackgroundColor3 = red, BorderSizePixel = 0, ZIndex = 63 }, ui)
		local recDot = new("Frame", { Position = UDim2.fromOffset(48, 22), Size = UDim2.fromOffset(12, 12), BackgroundColor3 = red, BorderSizePixel = 0, ZIndex = 64 }, ui)
		corner(recDot, 6)
		label(ui, { Size = UDim2.new(1, 0, 0, 56), Text = "AIRSTRIKE  -  SATELLITE LINK", TextSize = 24, ZIndex = 64 })
		local hint = label(ui, { AnchorPoint = Vector2.new(0, 1), Position = UDim2.fromScale(0, 1), Size = UDim2.new(1, 0, 0, 52), Text = "", TextSize = 18, TextColor3 = COL.dim, ZIndex = 64 })
		local timer = new("Frame", { AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 34, 1, -56), Size = UDim2.new(1, -68, 0, 5), BackgroundColor3 = COL.orange, BorderSizePixel = 0, ZIndex = 64 }, ui)
		local ret = new("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(86, 86), BackgroundTransparency = 1, ZIndex = 65 }, ui)
		corner(ret, 43)
		stroke(ret, red, 3, 0)
		new("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(6, 6), BackgroundColor3 = red, BorderSizePixel = 0, ZIndex = 66 }, ret)
		for i = 0, 3 do
			local horiz = i % 2 == 0
			local sgn = i < 2 and -1 or 1
			new("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = red, BorderSizePixel = 0, ZIndex = 66,
				Position = horiz and UDim2.new(0.5, sgn * 56, 0.5, 0) or UDim2.new(0.5, 0, 0.5, sgn * 56),
				Size = horiz and UDim2.fromOffset(26, 3) or UDim2.fromOffset(3, 26) }, ret)
		end

		-- ---- "AIRSTRIKE READY" prompt ----
		local prompt = new("Frame", { Name = "StrikePrompt", AnchorPoint = Vector2.new(0.5, 1), Position = UDim2.new(0.5, 0, 1, -130), Size = UDim2.fromOffset(330, 42),
			BackgroundColor3 = RGB(14, 16, 22), BackgroundTransparency = 0.15, BorderSizePixel = 0, Visible = false, ZIndex = 3 }, hud)
		corner(prompt, 8)
		stroke(prompt, red, 2, 0.2)
		local pdot = new("Frame", { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 12, 0.5, 0), Size = UDim2.fromOffset(20, 20), BackgroundColor3 = red, BorderSizePixel = 0, ZIndex = 4 }, prompt)
		corner(pdot, 10)
		local ptxt = label(prompt, { Position = UDim2.fromOffset(40, 0), Size = UDim2.new(1, -48, 1, 0), Text = "", TextSize = 16, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 4 })

		local marker
		local function smooth(x) x = math.clamp(x, 0, 1) return x * x * (3 - 2 * x) end

		local function restoreCam()
			if AS.camOn then
				AS.camOn = false
				camera.CameraType = Enum.CameraType.Custom
				local h = myHum()
				if h then camera.CameraSubject = h end
			end
			ui.Visible = false
			HUD.cross.Visible = true
			if marker then marker:Destroy() marker = nil end
		end
		function AS.finish()
			if AS.model then AS.model:Destroy() end
			AS.model, AS.o = nil, nil
			AS.active, AS.phase, AS.hide = false, nil, 0
			W.swapUntil = math.max(W.swapUntil or 0, os.clock() + 0.25)
		end
		function AS.abort()
			restoreCam()
			AS.finish()
		end
		function AS.start()
			if AS.active or EQ.anim or not isAlive() or UI.blocking() then return end
			if (player:GetAttribute("Airstrikes") or 0) <= 0 then return end
			local now = os.clock()
			W.ads, W.sliding, W.burstQ = false, false, 0
			touchToggles.ADS = false
			Held.Fire = false
			AS.active, AS.phase, AS.t0, AS.tapped, AS.hide, AS.target = true, "pull", now, false, 0, nil
			AS.model, AS.o = build()
			playSound(SND.click2, 0.9, 0.5 * S.g.sfx)
		end
		local function beginOut()
			AS.phase, AS.ot, AS.topCF = "out", os.clock(), camera.CFrame
		end
		function AS.confirm() end -- the missile is steered with the mouse; nothing to confirm
		ui.InputChanged:Connect(function(input)
			if AS.phase == "missile" and input.UserInputType == Enum.UserInputType.Touch then
				AS.tdx = (AS.tdx or 0) + input.Delta.X * 1.4
				AS.tdy = (AS.tdy or 0) + input.Delta.Y * 1.4
			end
		end)

		local function uOf(t)
			if t < 1.0 then return 1 end
			if t < 1.65 then return 1 - smooth((t - 1.0) / 0.65) end
			if t < 1.75 then return -0.05 * (t - 1.65) / 0.1 end
			if t < 1.85 then return -0.05 * (1 - (t - 1.75) / 0.1) end
			if t < 2.0 then return 0 end
			return smooth((t - 2.0) / 0.25)
		end
		local function pose(hp, rx, rz, u, now)
			local o = AS.o
			if not o then return end
			local cam = camera.CFrame
			hp = hp + Vector3.new(math.sin(now * 1.7) * 0.012, math.sin(now * 2.3) * 0.012, 0)
			local hand = cam * CFrame.new(hp) * CFrame.Angles(math.rad(rx), 0, math.rad(rz))
			o.glove.CFrame = hand
			o.cuff.CFrame = hand * CFrame.new(0.03, -0.03, 0.3)
			o.thumb.CFrame = hand * CFrame.new(-0.22, 0.1, -0.12) * CFrame.Angles(0, math.rad(12), 0)
			local pcf = hand * PH
			o.body.CFrame = pcf
			o.screen.CFrame = pcf * CFrame.new(0, 0, 0.055)
			ik(cam:PointToWorldSpace(Vector3.new(0.85, -1.45, 0.45)), (hand * CFrame.new(0, -0.02, 0.3)).Position,
				cam:VectorToWorldSpace(Vector3.new(0.5, -1, 0.35)), o.upperR, o.foreR, o.elbowR)
			local btn = pcf:PointToWorldSpace(Vector3.new(0, 0, 0.075))
			local nrm = -pcf.LookVector
			local tip = btn + nrm * (0.9 * u) + (-cam.RightVector * 1.8 - cam.UpVector * 1.4) * u
			local S2 = cam:PointToWorldSpace(Vector3.new(-0.85, -1.45, 0.45))
			local dir = (tip - S2)
			if dir.Magnitude < 0.05 then return end
			dir = dir.Unit
			local h2 = CFrame.lookAt(tip - dir * 0.45, tip)
			o.glove2.CFrame = h2 * CFrame.new(0, 0, 0.2)
			o.cuff2.CFrame = h2 * CFrame.new(0, 0, 0.5)
			o.finger.CFrame = h2 * CFrame.new(0, 0, -0.2)
			ik(S2, (h2 * CFrame.new(0, 0, 0.55)).Position, cam:VectorToWorldSpace(Vector3.new(-0.5, -1, 0.35)), o.upperL, o.foreL, o.elbowL)
		end

		RunService:BindToRenderStep("BO_Strike", Enum.RenderPriority.Camera.Value + 10, function(dt)
			local now = os.clock()
			local n = player:GetAttribute("Airstrikes") or 0
			local nb = player:GetAttribute("Bombers") or 0
			FXS.update(now)
			if RL.model and (now >= W.reloadUntil or not isAlive()) then RL.stop(VM.model) end
			local ready = inMatch() and (n > 0 or nb > 0) and not AS.active and isAlive() and not UI.blocking()
			prompt.Visible = ready
			if ready then
				local kn = UI.touch and "TAP STRIKE" or ("PRESS [" .. prettyKey(S.keys.Airstrike or "T") .. "]")
				if n > 0 then
					ptxt.Text = (n > 1 and ("AIRSTRIKE x" .. n) or "AIRSTRIKE READY") .. "  -  " .. kn
				else
					ptxt.Text = (nb > 1 and ("SUICIDE BOMBER x" .. nb) or "SUICIDE BOMBER READY") .. "  -  " .. kn
				end
				pdot.BackgroundTransparency = 0.45 + 0.45 * math.sin(now * 6)
			end
			if not AS.active then return end
			local hum, root = myHum(), myRoot()
			if not isAlive() or not hum or not root or UI.blocking() or not inMatch() then
				AS.abort()
				return
			end
			W.swapUntil = math.max(W.swapUntil or 0, now + 0.3)
			W.ads = false
			local ph = AS.phase
			local t = now - AS.t0
			if ph == "pull" or ph == "tap" then
				AS.phase = (t >= 1.25) and "tap" or "pull"
				AS.hide = smooth(t / 0.3)
				local o = AS.o
				if t >= 1.74 and not AS.tapped then
					AS.tapped = true
					playSound(SND.ping, 1.1, 0.5 * S.g.sfx)
					o.btn.BackgroundColor3 = RGB(255, 255, 255)
					o.btxt.TextColor3 = red
					o.btxt.Text = "LOCKED"
					o.stat.Text = "UPLINK ESTABLISHED"
					o.sub.Text = "OPENING TARGETING FEED"
					task.delay(0.18, function() if o.btn.Parent then o.btn.BackgroundColor3 = red o.btxt.TextColor3 = RGB(255, 255, 255) end end)
				end
				o.ring.Size = UDim2.fromOffset(230 + math.sin(now * 7) * 8, 230 + math.sin(now * 7) * 8)
				local pos, rx, rz = EQ.sample(HAND, math.min(t, 2.5))
				pose(pos, rx, rz, uOf(t), now)
				if t >= TAP_END then
					AS.phase, AS.gt = "missile", now
					AS.fp, AS.fov = camera.CFrame, camera.FieldOfView
					local lv = AS.fp.LookVector
					AS.yaw, AS.pitch = math.atan2(-lv.X, -lv.Z), math.rad(-74)
					AS.tdx, AS.tdy, AS.sentAt, AS.mpart, AS.cam0, AS.seen, AS.camPrev, AS.lastCF = 0, 0, 0, nil, false, false, nil, nil
					player:SetAttribute("StrikeLive", nil)
					camera.CameraType = Enum.CameraType.Scriptable
					AS.camOn = true
					ui.Visible = true
					HUD.cross.Visible = false
					ret.Position = UDim2.fromScale(0.5, 0.5)
					hint.Text = UI.touch and "DRAG TO STEER THE MISSILE" or "MOVE THE MOUSE TO STEER THE MISSILE"
					hint.TextColor3 = COL.dim
					Net.Airstrike:FireServer("launch", AS.yaw, AS.pitch)
					playSound(SND.ping, 1.3, 0.6 * S.g.sfx)
				end
			elseif ph == "missile" or ph == "boom" then
				local g = now - AS.gt
				AS.hide = 1
				if AS.model and AS.model.Parent then AS.model.Parent = nil end
				recDot.BackgroundTransparency = (math.floor(now * 2) % 2 == 0) and 0 or 0.7
				if ph == "missile" then
					pcall(function() UIS.MouseBehavior = Enum.MouseBehavior.LockCenter end)
					local d = UIS:GetMouseDelta()
					local sens = 0.0026 * (S.g.sensSet and S.g.sens or 1)
					AS.yaw = AS.yaw - (d.X + AS.tdx) * sens
					AS.pitch = math.clamp(AS.pitch - (d.Y + AS.tdy) * sens, math.rad(-85), math.rad(-45))
					AS.tdx, AS.tdy = 0, 0
					if now - AS.sentAt > 0.05 then
						AS.sentAt = now
						Net.Airstrike:FireServer("steer", AS.yaw, AS.pitch)
					end
				end
				local m = AS.mpart
				if not (m and m.Parent) then m = Workspace:FindFirstChild("BO_Missile_" .. player.UserId) end
				AS.mpart = m
				local live = player:GetAttribute("StrikeLive") == true
				if ph == "missile" then
					if m and m.Parent then
						AS.seen = true
						AS.lastCF = m.CFrame
					elseif AS.lastCF and live then
						-- the part is not on this client right now: keep flying along the steered heading so the view never freezes
						local dirv = Vector3.new(-math.sin(AS.yaw) * math.cos(AS.pitch), math.sin(AS.pitch), -math.cos(AS.yaw) * math.cos(AS.pitch))
						local np = AS.lastCF.Position + dirv * 120 * dt
						AS.lastCF = CFrame.lookAt(np, np + dirv)
					end
					if AS.lastCF then
						local target = AS.lastCF * CFrame.new(0, 0, -9)
						camera.CFrame = AS.camPrev and AS.camPrev:Lerp(target, math.clamp(dt * 22, 0, 1)) or target
						AS.camPrev = camera.CFrame
						camera.FieldOfView = 78 + math.sin(now * 30) * 0.4
						hint.Text = string.format("ALT %d    MOVE THE MOUSE TO STEER", math.max(0, math.floor(AS.lastCF.Position.Y)))
						timer.Size = UDim2.new(0, math.max(0, (camera.ViewportSize.X - 68) * (1 - math.min(1, g / 8))), 0, 5)
					end
					if (not live and g > 1.2) or g > 16 then
						AS.phase, AS.ct = "boom", now
						hint.Text = "STRIKE COMPLETE"
						hint.TextColor3 = COL.green
					end
				end
				if ph == "boom" and now - AS.ct >= 1.1 then
					AS.camPrev, AS.seen, AS.mpart, AS.lastCF = nil, false, nil, nil
					beginOut()
				end
			elseif ph == "out" then
				local b2 = smooth((now - AS.ot) / 0.4)
				camera.CFrame = AS.topCF:Lerp(AS.fp, b2)
				camera.FieldOfView = 60 + (AS.fov - 60) * b2
				ui.BackgroundTransparency = 1
				if b2 >= 1 then
					restoreCam()
					camera.CFrame = AS.fp
					camera.FieldOfView = AS.fov
					if AS.model then AS.model.Parent = camera end
					AS.phase, AS.ot = "out2", now
				end
			elseif ph == "out2" then
				local tt = now - AS.ot
				AS.hide = 1 - smooth(tt / 0.35)
				local pos, rx, rz = EQ.sample(PUT, math.min(tt, 0.35))
				pose(pos, rx, rz, 1, now)
				if tt >= 0.35 then AS.finish() end
			end
		end)
	end

	-- ---------- first-person RELOAD animation (left hand pulls the mag, grabs a new one, seats it, slaps the charging handle) + STIM screen glow ----------
	do
		local function K(t, v) return { t = t, pos = v, rx = 0, rz = 0 } end
		local MAGN = { Mag = true, AttMag = true, AttMagBand = true, AttMagTab = true }
		function RL.stop(vm)
			if RL.model then RL.model:Destroy() end
			if RL.drop then RL.drop:Destroy() end
			if RL.parts then
				local pv = (vm and vm.Parent) and vm:GetPivot() or nil
				for part, o in pairs(RL.parts) do
					if part.Parent then
						part.Transparency = o.t
						if pv then part.CFrame = pv * o.rel end
					end
				end
			end
			RL.model, RL.parts, RL.drop, RL.vm, RL.dropped = nil, nil, nil, nil, false
		end
		local function build(vm)
			local pv = vm:GetPivot()
			RL.parts = {}
			local template
			for _, d in ipairs(vm:GetChildren()) do
				if d:IsA("BasePart") and MAGN[d.Name] then
					RL.parts[d] = { rel = pv:ToObjectSpace(d.CFrame), t = d.Transparency }
					if d.Name == "AttMag" or (d.Name == "Mag" and not template) then template = d end
				end
			end
			RL.template = template
			RL.magRel = template and pv:ToObjectSpace(template.CFrame) or CFrame.new(0, -0.5, 0)
			local m = Instance.new("Model")
			m.Name = "BO_ReloadVM"
			local o = {}
			local function pr(name, size, col, mat, shape)
				local x = Instance.new("Part")
				x.Name, x.Size, x.Color, x.Material = name, size, col, mat or Enum.Material.SmoothPlastic
				x.Anchored, x.CanCollide, x.CanQuery, x.CanTouch, x.CastShadow = true, false, false, false, false
				if shape then x.Shape = shape end
				x.Parent = m
				return x
			end
			local op = P.equip and OPERATORS[P.equip.operator]
			local suit = (op and op.suit) or RGB(70, 78, 64)
			o.upper = pr("Upper", Vector3.new(0.44, 0.44, 1.6), suit)
			o.fore = pr("Fore", Vector3.new(0.38, 0.38, 1.6), suit)
			o.elbow = pr("Elbow", Vector3.new(0.46, 0.46, 0.46), suit, nil, Enum.PartType.Ball)
			o.glove = pr("Glove", Vector3.new(0.42, 0.34, 0.5), RGB(20, 21, 25))
			o.cuff = pr("Cuff", Vector3.new(0.4, 0.4, 0.18), RGB(26, 28, 32):Lerp(suit, 0.3))
			if template then
				o.spare = template:Clone()
				for _, c in ipairs(o.spare:GetDescendants()) do if not c:IsA("BasePart") and not c:IsA("SpecialMesh") then c:Destroy() end end
				o.spare.Transparency = 1
				o.spare.Parent = m
			end
			m.Parent = camera
			Debris:AddItem(m, 12)
			RL.model, RL.o, RL.vm, RL.dropped = m, o, vm, false
		end
		function RL.update(now, vm, def)
			local active = vm and vm.Parent == camera and now < W.reloadUntil and W.reloadDur and isAlive()
			if not active then
				if RL.model or RL.parts then RL.stop(vm) end
				return
			end
			if RL.vm ~= vm then
				RL.stop(nil)
				build(vm)
			end
			local o = RL.o
			local p = math.clamp((now - (W.reloadStart or now)) / W.reloadDur, 0, 1)
			local pv = vm:GetPivot()
			local cam = camera.CFrame
			local rot = pv - pv.Position
			local function G(v) return pv:PointToWorldSpace(v) end
			local function C(v) return cam:PointToWorldSpace(v) end
			local mr = RL.magRel.Position
			local bl = def.len * 0.46
			local grip = Vector3.new(-0.32, -0.05, 0.05)
			local away = Vector3.new(-1.4, -1.6, -0.9)
			local keys = {
				K(0, C(away)), K(0.18, G(mr + Vector3.new(-0.32, -0.1, 0.05))), K(0.3, G(mr + grip)),
				K(0.44, G(mr + Vector3.new(-0.38, -1.0, 0.25))), K(0.52, C(Vector3.new(-0.95, -1.35, -0.8))),
				K(0.64, C(Vector3.new(-0.7, -1.05, -1.0))), K(0.78, G(mr + Vector3.new(-0.32, -0.95, 0.05))),
				K(0.86, G(mr + Vector3.new(-0.32, -0.12, 0.05))), K(0.93, G(Vector3.new(-0.05, 0.28, bl / 2 + 0.1))), K(1, C(away)),
			}
			local H = EQ.sample(keys, p)
			-- arm
			local S2 = cam:PointToWorldSpace(Vector3.new(-0.85, -1.45, 0.45))
			local dv = H - S2
			if dv.Magnitude > 0.1 then
				local dir = dv.Unit
				local h2 = CFrame.lookAt(H - dir * 0.2, H)
				o.glove.CFrame = h2
				o.cuff.CFrame = h2 * CFrame.new(0, 0, 0.3)
				AS.ik(S2, (h2 * CFrame.new(0, 0, 0.5)).Position, cam:VectorToWorldSpace(Vector3.new(-0.5, -1, 0.35)), o.upper, o.fore, o.elbow)
			end
			-- magazines
			local function setMags(fn, hide)
				for part, d in pairs(RL.parts) do
					if part.Parent then
						part.Transparency = hide and 1 or d.t
						if fn then part.CFrame = fn(pv * d.rel) end
					end
				end
			end
			if p < 0.3 or p >= 0.86 then
				setMags(nil, false)
				if p >= 0.86 then
					for part, d in pairs(RL.parts) do if part.Parent then part.CFrame = pv * d.rel end end
				end
			elseif p < 0.44 then
				local gp = G(mr + grip)
				local delta = H - gp
				setMags(function(w) return CFrame.new(delta) * w end, false)
			else
				setMags(nil, true)
				if not RL.dropped and RL.template then
					RL.dropped = true
					RL.dropT = now
					RL.dropCF = CFrame.new(H - rot:VectorToWorldSpace(grip)) * rot
					RL.drop = RL.template:Clone()
					for _, c in ipairs(RL.drop:GetDescendants()) do if not c:IsA("BasePart") and not c:IsA("SpecialMesh") then c:Destroy() end end
					RL.drop.Transparency = 0
					RL.drop.Parent = RL.model
					playSound(SND.click, 0.7, 0.4 * S.g.sfx)
				end
			end
			if RL.drop then
				local td = now - RL.dropT
				if td > 0.7 then
					RL.drop:Destroy()
					RL.drop = nil
				else
					RL.drop.CFrame = CFrame.new(0, -(1.0 * td + 3.2 * td * td), 0) * RL.dropCF * CFrame.Angles(td * 3, 0, td * 2)
				end
			end
			if o.spare then
				if p >= 0.5 and p < 0.86 then
					o.spare.Transparency = RL.template.Transparency
					o.spare.CFrame = CFrame.new(H + rot:VectorToWorldSpace(Vector3.new(0.32, 0.12, -0.05))) * rot
				else
					o.spare.Transparency = 1
				end
			end
			if p > 0.78 and not RL.seated and p < 0.86 then RL.seated = true playSound(SND.collide, 1.6, 0.35 * S.g.sfx) end
			if p < 0.5 then RL.seated = false end
			if p > 0.9 and not RL.slap then RL.slap = true playSound(SND.snap, 1.2, 0.4 * S.g.sfx) end
			if p < 0.5 then RL.slap = false end
		end

		-- ---- stim: white glow around the screen while the stim is working ----
		local function edge(rot, pos, size, anchor)
			local f = new("Frame", { BackgroundColor3 = RGB(255, 255, 255), BorderSizePixel = 0, Size = size, Position = pos, AnchorPoint = anchor, ZIndex = 40, Visible = false }, gui)
			local g = new("UIGradient", { Rotation = rot, Transparency = NumberSequence.new(1, 1) }, f)
			return { f = f, g = g }
		end
		FXS.edges = {
			edge(90, UDim2.fromScale(0, 0), UDim2.fromScale(1, 0.3), Vector2.new(0, 0)),
			edge(270, UDim2.fromScale(0, 1), UDim2.fromScale(1, 0.3), Vector2.new(0, 1)),
			edge(0, UDim2.fromScale(0, 0), UDim2.fromScale(0.22, 1), Vector2.new(0, 0)),
			edge(180, UDim2.fromScale(1, 0), UDim2.fromScale(0.22, 1), Vector2.new(1, 0)),
		}
		FXS.wash = new("Frame", { BackgroundColor3 = RGB(235, 255, 245), BorderSizePixel = 0, Size = UDim2.fromScale(1, 1), ZIndex = 39, Visible = false, BackgroundTransparency = 1 }, gui)
		function FXS.update(now)
			local left = (player:GetAttribute("StimUntil") or 0) - Workspace:GetServerTimeNow()
			local on = inMatch() and isAlive() and left > 0 and not UI.blocking()
			if on and not FXS.wasOn then FXS.flashAt = now end
			FXS.wasOn = on
			local inten = on and (math.min(1, left / 1.2) * (0.62 + 0.08 * math.sin(now * 5))) or 0
			local flash = FXS.flashAt and math.max(0, 1 - (now - FXS.flashAt) / 0.7) or 0
			if not on then flash = 0 end
			local tot = math.min(0.95, inten + flash * 0.55)
			local vis = tot > 0.01
			for _, e in ipairs(FXS.edges) do
				e.f.Visible = vis
				if vis then e.g.Transparency = NumberSequence.new(1 - tot, 1) end
			end
			FXS.wash.Visible = vis
			if vis then FXS.wash.BackgroundTransparency = 1 - tot * 0.2 end
		end
	end

	-- ---------- persistent first-person ARMS: both hands stay on the gun (right on the grip, left on the handguard), CoD style ----------
	local FA = { arms = {} }
	do
		local function mkPart(m, size, col, mat, shape)
			local x = Instance.new("Part")
			x.Size, x.Color, x.Material = size, col, mat or Enum.Material.Fabric
			x.Anchored, x.CanCollide, x.CanQuery, x.CanTouch, x.CastShadow = true, false, false, false, false
			x.TopSurface, x.BottomSurface = Enum.SurfaceType.Smooth, Enum.SurfaceType.Smooth
			if shape then x.Shape = shape end
			x.Parent = m
			return x
		end
		function FA.destroy()
			if FA.model then FA.model:Destroy() end
			FA.model, FA.arms, FA.opId = nil, {}, nil
		end
		function FA.build(opId)
			FA.destroy()
			local op = OPERATORS[opId] or OPERATORS.OP_STANDARD
			local L = op.look or {}
			local sleeve = L.fur or L.top or op.suit
			local hand = L.bareHands and (L.skin or RGB(206, 162, 130)) or (L.glove or RGB(20, 21, 25))
			local cuff = RGB(26, 28, 32):Lerp(sleeve, 0.25)
			local m = Instance.new("Model")
			m.Name = "BO_FPArms"
			for _, side in ipairs({ "L", "R" }) do
				FA.arms[side] = {
					upper = mkPart(m, Vector3.new(0.5, 0.5, 1.45), sleeve),
					fore = mkPart(m, Vector3.new(0.44, 0.44, 1.45), sleeve),
					elbow = mkPart(m, Vector3.new(0.52, 0.52, 0.52), sleeve, nil, Enum.PartType.Ball),
					glove = mkPart(m, Vector3.new(0.5, 0.42, 0.62), hand),
					cuff = mkPart(m, Vector3.new(0.5, 0.5, 0.22), cuff),
					pad = mkPart(m, Vector3.new(0.3, 0.14, 0.5), op.accent, Enum.Material.Neon),
				}
			end
			m.Parent = camera
			FA.model, FA.opId = m, opId
		end
		local function setVis(a, on)
			for _, q in pairs(a) do q.LocalTransparencyModifier = on and 0 or 1; q.Transparency = on and 0 or 1 end
		end
		function FA.update(now, vm, def, hide, ease)
			local opId = (P.equip and P.equip.operator) or "OP_STANDARD"
			if FA.opId ~= opId or not FA.model or FA.model.Parent ~= camera then FA.build(opId) end
			local showAll = vm and vm.Parent == camera and hide < 0.05 and ease < 0.55 and isAlive()
			local reloading = RL.model ~= nil
			local pv = vm and vm:GetPivot() or CFrame.new()
			local cam = camera.CFrame
			local bl = (def and def.len or 3) * 0.46
			local function spot(names, fallback)
				for _, n in ipairs(names) do
					local x = vm and vm:FindFirstChild(n)
					if x and x:IsA("BasePart") then return pv:PointToObjectSpace(x.Position), x end
				end
				return fallback
			end
			local gripOS, gripPart = spot({ "Grip" }, Vector3.new(0, -0.45, bl * 0.12))
			local handOS, handPart = spot({ "Handguard", "Barrel" }, Vector3.new(0, -0.1, -bl * 0.35))
			if showAll then
				-- right hand: wraps the pistol grip; left hand: cups under the handguard
				local r = gripOS + Vector3.new(0.02, -((gripPart and gripPart.Size.Y or 0.5) * 0.15), 0)
				local lz = handOS.Z
				if handPart and handPart.Name == "Barrel" then lz = handOS.Z + (handPart.Size.Z or 1) * 0.15 end
				local l = Vector3.new(handOS.X - 0.06, handOS.Y - ((handPart and handPart.Size.Y or 0.4) * 0.5 + 0.12), math.min(lz, -bl * 0.18))
				local targets = {
					R = { pv:PointToWorldSpace(r), cam:PointToWorldSpace(Vector3.new(0.85, -1.5, 0.5)), cam:VectorToWorldSpace(Vector3.new(0.7, -1, 0.3)) },
					L = { pv:PointToWorldSpace(l), cam:PointToWorldSpace(Vector3.new(-0.85, -1.5, 0.5)), cam:VectorToWorldSpace(Vector3.new(-0.7, -1, 0.3)) },
				}
				for side, a in pairs(FA.arms) do
					local on = not (side == "L" and reloading)
					setVis(a, on)
					if on then
						local H, S0, pole = targets[side][1], targets[side][2], targets[side][3]
						local dv = H - S0
						if dv.Magnitude > 0.1 then
							local h2 = CFrame.lookAt(H - dv.Unit * 0.12, H)
							a.glove.CFrame = h2
							a.cuff.CFrame = h2 * CFrame.new(0, 0, 0.42)
							a.pad.CFrame = h2 * CFrame.new(0, 0.24, -0.02)
							AS.ik(S0, (h2 * CFrame.new(0, 0, 0.62)).Position, pole, a.upper, a.fore, a.elbow)
						end
					end
				end
			else
				for _, a in pairs(FA.arms) do setVis(a, false) end
			end
		end
	end

	local slideDir = Vector3.zero
	local function startSlide()
		local hum, root = myHum(), myRoot()
		if not hum or not root or not isAlive() then return end
		local now = os.clock()
		if W.sliding then W.slideCancel = true return end -- pressing slide/crouch again cancels it
		local fast = root.AssemblyLinearVelocity * Vector3.new(1, 0, 1)
		if W.sprinting and fast.Magnitude > 12 and now > W.slideCd and hum.FloorMaterial ~= Enum.Material.Air then
			W.sliding = true
			W.slideCancel = false
			W.slideDur = 1.15
			W.slideStart = now
			W.slideUntil = now + W.slideDur
			-- CoD style: an instant burst past sprint speed, then friction bleeds it off
			W.slideS0 = math.max(fast.Magnitude * 1.22, CFG.BASE_WALK * CFG.SPRINT_MULT * 1.3) * (W.tact and 1.06 or 1)
			W.slideSpd = W.slideS0
			W.slideCd = now + 1.3
			W.tact = false
			slideDir = fast.Unit
			playSound(SND.lunge, 0.6, 0.4 * S.g.sfx)
		else
			W.crouch = not W.crouch
		end
	end

	local function fireOnce(def)
		local now = os.clock()
		local a = W.ammo[W.slot]
		if now < W.reloadUntil or now < W.swapUntil or not isAlive() or not a then return false end
		if a[1] <= 0 then
			if now - (W.lastClick or 0) > 0.35 then
				W.lastClick = now
				playSound(SND.clickfast, 1.4, 0.5 * S.g.sfx)
			end
			if a[2] > 0 then startReload() end
			return false
		end
		a[1] -= 1
		if W.sliding then W.sliding = false end
		W.sprinting = false
		local adsOn = W.ads and W.adsT > 0.5
		Net.Fire:FireServer(camera.CFrame.Position, aimCF.LookVector, adsOn)
		local k = S.g.shake
		R.pitch += def.kickP * 0.27 * (adsOn and 0.6 or 1) * (0.5 + 0.5 * k)
		R.yaw += (math.random() - 0.5) * 2 * def.kickY * 0.27 * (adsOn and 0.6 or 1) * (0.5 + 0.5 * k)
		VM.kick = math.min(1, VM.kick + 0.5)
		playGunShot(def.id, nil, S.g.sfx)
		if VM.flash then
			VM.flash.Enabled = true
			task.delay(0.045, function()
				if VM.flash then VM.flash.Enabled = false end
			end)
		end
		-- local tracer (server sends tracers to everyone else)
		local params = RaycastParams.new()
		params.FilterType = Enum.RaycastFilterType.Exclude
		params.FilterDescendantsInstances = { player.Character, camera, fxFolder }
		local hit = Workspace:Raycast(camera.CFrame.Position, aimCF.LookVector * 400, params)
		local endPos = hit and hit.Position or (camera.CFrame.Position + aimCF.LookVector * 400)
		local from = (VM.muzzle and VM.muzzle.Position) or (camera.CFrame.Position + aimCF.LookVector * 2)
		if not adsOn or not (def.scope or def.scopeOpt) then tracer(from, endPos, def.color) end
		if hit then impact(endPos, def.color) end
		updateAmmoHud()
		return true
	end

	local function weaponStep(now)
		local def = curDef()
		if not def or not isAlive() or UI.blocking() then
			W.burstQ = 0
			return
		end
		local interval = 60 / def.rpm + 0.005
		if def.burst then
			if W.pressEdge and now - W.pressEdge > 0.2 then W.pressEdge = false end
			if W.pressEdge and W.burstQ <= 0 and now >= W.nextFire then
				W.burstQ = def.burst
				W.pressEdge = false
			end
			while W.burstQ > 0 and now >= W.nextFire do
				if not fireOnce(def) then
					W.burstQ = 0
					break
				end
				W.burstQ -= 1
				W.nextFire = now + (W.burstQ > 0 and interval or def.burstDelay + 0.01)
				now = os.clock()
			end
		elseif def.auto then
			if Held.Fire and now >= W.nextFire then
				if fireOnce(def) then W.nextFire = now + interval end
			end
		else
			if W.pressEdge and now - W.pressEdge > 0.2 then W.pressEdge = false end
			if W.pressEdge and now >= W.nextFire then
				W.pressEdge = false
				if fireOnce(def) then W.nextFire = now + interval end
			end
		end
		if def.auto then W.pressEdge = false end
	end

	-- ---------- input dispatch ----------
	local keyAct, padAct = {}, {}
	local function rebuildMaps()
		keyAct, padAct = {}, {}
		for id, name in pairs(S.keys) do keyAct[name] = id end
		for id, name in pairs(S.pad) do padAct[name] = id end
	end

	local function dispatch(id, down)
		if id == "Menu" then
			if down then UI.toggleMenu() end
			return
		end
		if not inMatch() then return end
		if AS.active then
			if Held[id] ~= nil then Held[id] = down end
			if down and (id == "Fire" or id == "Airstrike") then AS.confirm() end
			return
		end
		if UI.blocking() then
			if not down then
				Held[id] = false
				if id == "Scoreboard" then HUD.showScoreboard(false) end
			end
			return
		end
		if Held[id] ~= nil then Held[id] = down end
		if id == "Fire" then
			if down then W.pressEdge = os.clock() end
		elseif id == "ADS" then
			if UI.touch or S.g.toggleAds then
				if down then touchToggles.ADS = not touchToggles.ADS end
				W.ads = touchToggles.ADS
			else
				W.ads = down
			end
			if W.ads and os.clock() < W.reloadUntil then W.ads = false end
		elseif id == "Reload" then
			if down then startReload() end
		elseif id == "Swap" then
			if down then switchSlot(W.slot == 1 and 2 or 1) end
		elseif id == "Slot1" then
			if down then switchSlot(1) end
		elseif id == "Slot2" then
			if down then switchSlot(2) end
		elseif id == "Melee" then
			if down then doMelee() end
		elseif id == "Grenade" then
			if down then EQ.grenade() else EQ.grenadeRelease() end
		elseif id == "Stim" then
			if down then EQ.stim() end
		elseif id == "Airstrike" then
			if down then
				if (player:GetAttribute("Bombers") or 0) > 0 and (player:GetAttribute("Airstrikes") or 0) <= 0 then
					if not AS.active and isAlive() and not UI.blocking() then Net.Streak:FireServer("bomber") end
				else
					AS.start()
				end
			end
		elseif id == "Scoreboard" then
			HUD.showScoreboard(down)
		elseif id == "Slide" then
			if down then startSlide() end
		elseif id == "Sprint" then
			if down then
				local t = os.clock()
				if t - (W.lastSprintTap or 0) < 0.3 then W.tact = true end -- double-tap sprint = tactical sprint
				W.lastSprintTap = t
			end
			if down and W.sliding then W.slideCancel = true end -- sprint cancels a slide straight back into a run
			if UI.touch and down then
				touchToggles.Sprint = not touchToggles.Sprint
				Held.Sprint = touchToggles.Sprint
			end
		end
	end

	local function inputName(input)
		local t = input.UserInputType
		if t == Enum.UserInputType.Keyboard or t == Enum.UserInputType.Gamepad1 then return input.KeyCode.Name, (t == Enum.UserInputType.Gamepad1) end
		if t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.MouseButton2 or t == Enum.UserInputType.MouseButton3 then return t.Name, false end
		return nil, false
	end

	local function onBoundInput(_, state, input)
		if capture then return Enum.ContextActionResult.Pass end
		if UIS:GetFocusedTextBox() then return Enum.ContextActionResult.Pass end
		local name, isPad = inputName(input)
		if not name then return Enum.ContextActionResult.Pass end
		local id = isPad and padAct[name] or keyAct[name]
		if not inMatch() then
			if id == "Menu" and state == Enum.UserInputState.Begin then dispatch("Menu", true) end
			return Enum.ContextActionResult.Pass
		end
		if state == Enum.UserInputState.Begin then
			if id then dispatch(id, true) end
		elseif state == Enum.UserInputState.End or state == Enum.UserInputState.Cancel then
			if id then dispatch(id, false) end
		end
		return Enum.ContextActionResult.Sink
	end

	local function rebindCAS()
		CAS:UnbindAction("BO_Input")
		local list, seen = {}, {}
		local function add(name)
			if seen[name] then return end
			seen[name] = true
			if name:sub(1, 11) == "MouseButton" then
				local ok, v = pcall(function() return Enum.UserInputType[name] end)
				if ok and v then table.insert(list, v) end
			else
				local ok, v = pcall(function() return Enum.KeyCode[name] end)
				if ok and v then table.insert(list, v) end
			end
		end
		for _, n in ipairs({ "W", "A", "S", "D", "Up", "Down", "Left", "Right", "Space", "ButtonA", "ButtonB", "ButtonX", "ButtonY", "ButtonL1", "ButtonR1",
			"ButtonL2", "ButtonR2", "ButtonL3", "ButtonR3", "ButtonSelect", "ButtonStart" }) do add(n) end
		for _, v in pairs(S.keys) do add(v) end
		for _, v in pairs(S.pad) do add(v) end
		CAS:BindActionAtPriority("BO_Input", onBoundInput, false, 3000, table.unpack(list))
	end
	table.insert(settingsHooks, function()
		rebuildMaps()
		rebindCAS()
	end)
	rebuildMaps()
	rebindCAS()

	-- weapon state from the server
	Net.Ammo.OnClientEvent:Connect(function(t)
		if type(t) ~= "table" or type(t.ids) ~= "table" or type(t.a) ~= "table" then return end
		local attKey = table.concat(t.att1 or {}, ",") .. "|" .. table.concat(t.att2 or {}, ",")
		local changed = (W.ids[1] ~= t.ids[1]) or (W.ids[2] ~= t.ids[2]) or (W.slot ~= t.slot) or (W.attKey ~= attKey)
		W.attKey = attKey
		W.att = { cleanAttach(t.att1, LEVEL_CAP, true), cleanAttach(t.att2, LEVEL_CAP, true) }
		W.ids = { t.ids[1], t.ids[2] }
		W.slot = t.slot
		W.ammo = { { t.a[1][1], t.a[1][2] }, { t.a[2][1], t.a[2][2] } }
		if t.why == "reloaded" then
			W.reloadUntil = 0
		elseif t.why == "swap" or changed then
			W.reloadUntil = 0
			W.swapUntil = os.clock() + (curDef() and curDef().switch or 0.4)
		end
		if changed or not VM.model or VM.id ~= W.ids[W.slot] then viewmodelRebuild() end
		updateAmmoHud()
	end)

	-- =================================================================
	-- PER-FRAME: movement, first-person camera, viewmodel, weapon loop
	-- =================================================================
	local lastCamMode
	local function applyCameraMode(wantFps)
		if lastCamMode == wantFps then return end
		lastCamMode = wantFps
		pcall(function()
			if wantFps then
				player.CameraMode = Enum.CameraMode.LockFirstPerson
			else
				player.CameraMode = Enum.CameraMode.Classic
				UIS.MouseDeltaSensitivity = 1
			end
		end)
	end

	XT.modalBtn = new("TextButton", { Size = UDim2.fromOffset(0, 0), BackgroundTransparency = 1, Text = "", Modal = true, Visible = false }, gui)

	-- separate X / Y look sensitivity, controller + touch sensitivity, dead zone and invert: wrap Roblox's own camera input
	UI.yRatio, UI.padMul = 1, 1
	task.spawn(function()
		pcall(function()
			local pm = player:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule", 15)
			local ci = require(pm:WaitForChild("CameraModule", 15):WaitForChild("CameraInput", 15))
			local orig = ci.getRotation
			if type(orig) ~= "function" then return end
			ci.getRotation = function(...)
				local r = orig(...)
				if typeof(r) ~= "Vector2" then return r end
				local x, y = r.X, r.Y
				local lt = UIS:GetLastInputType()
				if lt == Enum.UserInputType.Gamepad1 then
					local dead = S.g.padDead or 0
					if dead > 0 then
						for _, o in ipairs(UIS:GetGamepadState(Enum.UserInputType.Gamepad1)) do
							if o.KeyCode == Enum.KeyCode.Thumbstick2 and o.Position.Magnitude < dead then x, y = 0, 0 end
						end
					end
					x = x * S.g.padX * UI.padMul
					y = y * S.g.padY * UI.padMul
					if S.g.padInvert then y = -y end
				elseif lt == Enum.UserInputType.Touch then
					x, y = x * S.g.touchSens, y * S.g.touchSens
				else
					y = y * UI.yRatio
					if S.g.invertY then y = -y end
				end
				return Vector2.new(x, y)
			end
		end)
	end)
	RunService:BindToRenderStep("BO_Move", Enum.RenderPriority.Input.Value + 1, function(dt)
		local hum, root = myHum(), myRoot()
		local active = inMatch() and hum ~= nil and root ~= nil and hum.Health > 0
		applyCameraMode(active)
		XT.modalBtn.Visible = active and (UI.blocking() or AS.phase == "guide" or AS.phase == "confirm")
		if not active then
			pcall(function() UIS.MouseIconEnabled = true end)
			W.sprinting, W.sliding, W.ads = false, false, false
			return
		end
		local def = curDef()
		local now = os.clock()
		local blocked = UI.blocking() or AS.active
		pcall(function() UIS.MouseIconEnabled = UI.blocking() end) -- no mouse arrow over the gun while playing
		local baseX = (S.g.sensSet and S.g.sens or 1)
		local scoped = (def.scope or def.scopeOpt) and W.ads
		local adsMul = W.ads and (scoped and S.g.scopeSens or S.g.adsSens) or 1
		UIS.MouseDeltaSensitivity = math.max(0.05, baseX * adsMul)
		UI.yRatio = math.clamp(S.g.ySens / math.max(0.05, baseX), 0.05, 20) -- Y-axis sensitivity relative to X (applied in the camera hook)
		UI.padMul = W.ads and (scoped and S.g.padAds * 0.7 or S.g.padAds) or 1
		-- smooth movement: the input vector eases in/out (quick pick-up, short glide when you let go; less air control) instead of snapping
		local tgtV = Vector3.zero
		if not blocked then
			local x = (Held.MoveR and 1 or 0) - (Held.MoveL and 1 or 0)
			local z = (Held.MoveB and 1 or 0) - (Held.MoveF and 1 or 0)
			if x ~= 0 or z ~= 0 then tgtV = Vector3.new(x, 0, z).Unit end
			if Held.Jump then hum.Jump = true end
		end
		local inAir = hum.FloorMaterial == Enum.Material.Air
		local ease = (tgtV.Magnitude > 0) and (inAir and 6 or 16) or (inAir and 2.2 or 11)
		W.mvV = (W.mvV or Vector3.zero):Lerp(tgtV, 1 - math.exp(-dt * ease))
		if W.mvV.Magnitude > 0.04 then
			hum:Move(W.mvV, true)
		else
			W.mvV = Vector3.zero
			hum:Move(Vector3.zero, true)
		end
		local moving = hum.MoveDirection.Magnitude > 0.1
		local look = camera.CFrame.LookVector * Vector3.new(1, 0, 1)
		local fwd = moving and look.Magnitude > 0.01 and hum.MoveDirection:Dot(look.Unit) > 0.5
		local want = Held.Sprint or (S.g.autoSprint and fwd)
		W.sprinting = (want and moving and not W.ads and not Held.Fire and not W.crouch and not W.sliding and not blocked) and true or false
		if not Held.Sprint and not S.g.autoSprint then W.sprinting = false end
		local spd = CFG.BASE_WALK * def.speed
		if W.ads then spd *= 0.65 end
		if W.crouch then spd *= 0.55 end
		if W.sprinting then spd *= CFG.SPRINT_MULT * (W.tact and 1.1 or 1) else W.tact = false end
		if EQ.boosted() then spd *= 1.18 end
		local camY = 0
		local vy = root.AssemblyLinearVelocity.Y
		if W.sliding then
			local left = W.slideUntil - now
			local age = now - (W.slideStart or now)
			local wantCancel = W.slideCancel or Held.Jump
			local cancel = wantCancel and age > 0.22 -- a slide has to commit for a beat before it can be cancelled
			if wantCancel and not cancel then W.slideCancel = true end
			if left <= 0 or cancel then
				-- slide cancel / slide hop: jump or sprint/slide again carries the slide speed straight out of the slide
				W.sliding = false
				W.crouch = false
				W.slideCancel = false
				W.mvSpd = math.max(W.mvSpd or 0, (W.slideSpd or spd) * (cancel and 0.92 or 1))
				if cancel then
					-- slide cancel: momentum carries, the gun snaps back up fast, but you can't chain slides at full speed
					W.slideCd = now + 0.55
					W.fastAds = now + 0.45
					W.cancelKick = 1
					local v = slideDir * (W.slideSpd or spd) * 0.92
					root.AssemblyLinearVelocity = Vector3.new(v.X, root.AssemblyLinearVelocity.Y, v.Z)
				else
					W.slideCd = now + 0.25
				end
			else
				local p = left / (W.slideDur or 0.9)
				local endS = CFG.BASE_WALK * 0.85
				W.slideSpd = endS + ((W.slideS0 or endS * 2) - endS) * (p ^ 1.7)
				spd = W.slideSpd
				if look.Magnitude > 0.01 then slideDir = slideDir:Lerp(look.Unit, math.min(1, dt * 1.4)).Unit end -- you can bend the slide a little
				hum:Move(slideDir, false)
				W.mvSpd = spd
				camY = -2.3 - 0.15 * math.sin(now * 38) * (left / (W.slideDur or 1)) -- slight scrape rumble
			end
		elseif W.crouch then
			camY = -1.1
		end
		if AS.active then spd = 0 end
		-- speed eases between walk / sprint / crouch instead of snapping
		if W.sliding or AS.active then
			W.mvSpd = spd
		else
			W.mvSpd = W.mvSpd or spd
			W.mvSpd += (spd - W.mvSpd) * (1 - math.exp(-dt * ((spd > W.mvSpd) and 8 or 12)))
			spd = W.mvSpd
		end
		hum.WalkSpeed = spd
		-- landing dip + snappier jump arc (a little extra gravity on the way down / when you let go of jump)
		if W.wasAir and not inAir and (W.lastVy or 0) < -24 then W.landK = math.min(1, (-(W.lastVy) - 24) / 46) end
		W.wasAir, W.lastVy = inAir, vy
		W.landK = (W.landK or 0) * math.exp(-dt * 9)
		camY -= 0.85 * W.landK
		if inAir and not AS.active then
			local extra = 0
			if vy < 0 then extra = 0.45 elseif vy > 0 and not Held.Jump then extra = 0.6 end
			if extra > 0 then pcall(function() root:ApplyImpulse(Vector3.new(0, -root.AssemblyMass * Workspace.Gravity * extra * dt, 0)) end) end
		end
		-- camera feel: FOV kick on sprint/slide and a small lean into strafes
		W.cancelKick = (W.cancelKick or 0) * math.exp(-dt * 7)
		local tgtFov = W.sliding and 8 or (W.sprinting and (W.tact and 7 or 4.5) or 0)
		tgtFov += 5 * W.cancelKick
		W.fovK = (W.fovK or 0) + (tgtFov - (W.fovK or 0)) * (1 - math.exp(-dt * 7))
		local side = hum.MoveDirection:Dot(camera.CFrame.RightVector)
		local tgtRoll = (-side * 1.5 * math.min(1, spd / 16) - (W.sliding and (slideDir:Dot(camera.CFrame.RightVector) * 3) or 0)) * S.g.tilt
		W.roll = (W.roll or 0) + (tgtRoll - (W.roll or 0)) * (1 - math.exp(-dt * 8))
		hum.CameraOffset = hum.CameraOffset:Lerp(Vector3.new(0, camY, 0), math.min(1, dt * 12))
	end)

	RunService:BindToRenderStep("BO_View", Enum.RenderPriority.Camera.Value + 1, function(dt)
		local hum, root = myHum(), myRoot()
		local active = inMatch() and hum ~= nil and root ~= nil and hum.Health > 0
		if not active then
			if VM.model and VM.model.Parent then VM.model.Parent = nil end
			if FA.model then FA.destroy() end
			HUD.scope.Visible = false
			HUD.optic.Visible = false
			if UI.fovSet then
				UI.fovSet = false
				if not UI.camScripted then camera.FieldOfView = 70 end
			end
			return
		end
		UI.fovSet = true
		local def = curDef()
		local now = os.clock()
		-- recoil recovery + apply to the camera (also displaces real aim)
		local k = math.max(0, 1 - dt * 7)
		R.pitch *= k
		R.yaw *= k
		if not UI.blocking() then
			camera.CFrame = camera.CFrame * CFrame.Angles(math.rad(R.pitch), math.rad(R.yaw), 0)
		end
		aimCF = camera.CFrame
		camera.CFrame = camera.CFrame * CFrame.Angles(0, 0, math.rad((W.roll or 0) * (1 - (W.adsT or 0))))
		-- ADS blend + FOV
		local target = (W.ads and not UI.blocking()) and 1 or 0
		local adsRate = dt / def.adsTime * ((os.clock() < (W.fastAds or 0)) and 2.2 or 1)
		W.adsT = W.adsT + math.clamp(target - W.adsT, -adsRate, adsRate)
		local ease = W.adsT * W.adsT * (3 - 2 * W.adsT)
		camera.FieldOfView = S.g.fov + (S.g.fov * def.zoom - S.g.fov) * ease + (W.fovK or 0) * (1 - ease)
		local scoped = (def.scope or def.scopeOpt) and W.adsT > 0.85
		if scoped then HUD.layoutScope() end
		HUD.scope.Visible = scoped == true
		HUD.opticOn = HUD.setOptic(def, W.adsT)
		if VM.model then
			if VM.optFor ~= VM.model then
				VM.optFor, VM.optParts, VM.optT = VM.model, {}, {}
				for _, d in ipairs(VM.model:GetChildren()) do
					if d:IsA("BasePart") and (string.sub(d.Name, 1, 8) == "AttOptic" or string.sub(d.Name, 1, 7) == "AttLens") then table.insert(VM.optParts, d) VM.optT[d] = d.Transparency end
				end
			end
			-- the AK-47's holo housing stays on screen while aiming (it frames the reticle); everything else is hidden so it never blocks the view
			local keepHousing = false
			for _, op in ipairs(VM.optParts) do
				local hide = (HUD.opticOn or W.adsT > 0.35) and not (keepHousing and op.Name ~= "AttLens")
				op.Transparency = hide and 1 or (VM.optT[op] or 0)
			end
			-- everything behind the rear sight (stock, butt pad, cheek rest, charging handle...) would sit right on the camera when aiming: hide it
			if VM.rearFor ~= VM.model or VM.rearDef ~= def.len then
				VM.rearFor, VM.rearDef, VM.rearParts = VM.model, def.len, {}
				local limit = -def.len * 0.46 * 0.12 + 0.75
				for _, d in ipairs(VM.model:GetChildren()) do
					if d:IsA("BasePart") and (d.Name == "AttLaserBeam" or (VM.model:GetPivot():PointToObjectSpace(d.Position).Z > limit and d.Name ~= "Body")) then
						table.insert(VM.rearParts, { p = d, t = d.Transparency })
					end
				end
			end
			for _, rp in ipairs(VM.rearParts) do rp.p.Transparency = (W.adsT > 0.3) and 1 or rp.t end
		end
		-- viewmodel
		if not VM.model or VM.id ~= def.id then viewmodelRebuild() end
		local vm = VM.model
		if not vm then
			pcall(FA.update, now, nil, def, 0, 1)
			weaponStep(now)
			return
		end
		if (def.scope or def.scopeOpt) and W.adsT > 0.7 then
			if vm.Parent then vm.Parent = nil end
			pcall(FA.update, now, vm, def, 0, 1)
			VM.kick = math.max(0, VM.kick - dt * 9)
			weaponStep(now) -- the scoped view used to return before this ran, so sniper shots were never fired while zoomed in
			return
		end
		if vm.Parent ~= camera then vm.Parent = camera end
		local flat = root.AssemblyLinearVelocity * Vector3.new(1, 0, 1)
		local speed = flat.Magnitude
		local grounded = hum.FloorMaterial ~= Enum.Material.Air
		if grounded then VM.bob += dt * speed * 0.5 end
		local bobAmt = math.min(1, speed / 16) * (1 - ease * 0.85) * S.g.bob
		-- aim position: the sight line (iron sights or the optic lens) is lined up dead centre, with the camera 1.35 studs behind the rear sight
		local bl = def.len * 0.46
		local eyeY, rearZ = 0.37, -bl * 0.12
		if def.att then
			for _, aid in ipairs(def.att) do
				if aid == "RDOT" then eyeY, rearZ = 0.47, -bl * 0.1
				elseif aid == "SCOPE" then eyeY, rearZ = 0.58, -bl * 0.1
				elseif aid == "ZOOM3" then eyeY, rearZ = 0.54, -bl * 0.1
				elseif aid == "XSIGHT" then eyeY, rearZ = 0.46, -bl * 0.1 end
			end
		end
		-- measure the real sight line from the built model (optic lens, or the iron sights) so every gun and attachment lines up exactly
		if VM.sightFor ~= vm then
			VM.sightFor, VM.sEye, VM.sZ = vm, nil, nil
			local pv = vm.PrimaryPart and vm.PrimaryPart.CFrame or vm:GetPivot()
			local lens, optic = vm:FindFirstChild("AttLens"), vm:FindFirstChild("AttOptic") or vm:FindFirstChild("AttXBase") or vm:FindFirstChild("AttFrameBase")
			local fs, rs = vm:FindFirstChild("FrontSight"), vm:FindFirstChild("Sight")
			if lens and optic then
				VM.sEye, VM.sZ = pv:PointToObjectSpace(lens.Position).Y, pv:PointToObjectSpace(optic.Position).Z
			elseif fs and rs then
				VM.sEye, VM.sZ = pv:PointToObjectSpace(fs.Position).Y + fs.Size.Y / 2, pv:PointToObjectSpace(rs.Position).Z
			end
		end
		if VM.sEye and VM.sEye == VM.sEye and VM.sZ then eyeY, rearZ = VM.sEye, VM.sZ end
		local pos = Vector3.new(0.9, -0.85, -1.5):Lerp(Vector3.new(0, -eyeY, -(rearZ + 1.35)), ease)
		pos += Vector3.new(math.sin(VM.bob) * 0.035 * bobAmt, math.abs(math.cos(VM.bob)) * 0.03 * bobAmt, VM.kick * 0.3)
		local cf = CFrame.new(pos) * CFrame.Angles(math.rad(VM.kick * 3), math.rad(4 * (1 - ease)), 0)
		VM.sprintT = (VM.sprintT or 0) + ((W.sprinting and 1 or 0) - (VM.sprintT or 0)) * math.min(1, dt * 10)
		if VM.sprintT > 0.01 then
			cf = cf * CFrame.new(-0.25 * VM.sprintT, -0.3 * VM.sprintT, 0.1 * VM.sprintT) * CFrame.Angles(math.rad(-10 * VM.sprintT), math.rad(-40 * VM.sprintT), math.rad(12 * VM.sprintT))
		end
		if now < W.reloadUntil and W.reloadDur then
			local p = math.clamp((now - (W.reloadStart or now)) / W.reloadDur, 0, 1)
			local function sm(x) return x * x * (3 - 2 * x) end
			local down = sm(math.clamp(p / 0.22, 0, 1)) * (1 - sm(math.clamp((p - 0.86) / 0.14, 0, 1)))
			local tilt = sm(math.clamp((p - 0.08) / 0.2, 0, 1)) * (1 - sm(math.clamp((p - 0.82) / 0.12, 0, 1)))
			local seat = (p > 0.76 and p < 0.9) and math.sin((p - 0.76) / 0.14 * math.pi) or 0
			local rack = (p > 0.9) and math.sin((p - 0.9) / 0.1 * math.pi) or 0
			local shake = math.sin(p * 40) * 0.012 * tilt
			cf = cf * CFrame.new(0.14 * down + shake, -0.55 * down - 0.1 * seat, 0.2 * down + 0.2 * rack) * CFrame.Angles(math.rad(-26 * down + 7 * seat + 4 * rack), math.rad(-10 * tilt), math.rad(46 * tilt - 5 * seat))
		end
		local sw = math.clamp((now - VM.swapAt) / math.max(0.1, def.switch), 0, 1)
		if sw < 1 then
			local iv = (1 - sw) * (1 - sw)
			cf = cf * CFrame.new(0.3 * iv, -1.0 * iv, 0.35 * iv) * CFrame.Angles(math.rad(-55 * iv), math.rad(20 * iv), math.rad(30 * iv))
		end
		local mp = math.clamp((now - VM.meleeAt) / 0.35, 0, 1)
		if mp < 1 then
			local a = math.sin(mp * math.pi)
			cf = cf * CFrame.new(-0.4 * a, 0.1 * a, -0.8 * a) * CFrame.Angles(math.rad(18 * a), math.rad(-35 * a), 0)
		end
		VM.kick = math.max(0, VM.kick - dt * 9)
		local hide = math.max(EQ.update(now), AS.hide or 0)
		if hide > 0 then cf = cf * CFrame.new(0, -1.3 * hide, 0.5 * hide) * CFrame.Angles(math.rad(-30 * hide), 0, 0) end
		if hide >= 0.97 and vm.Parent then vm.Parent = nil end
		vm:PivotTo(camera.CFrame * cf)
		local okRL, errRL = pcall(RL.update, now, vm, def)
		if not okRL and not FA.warnedRL then FA.warnedRL = true warn("BLACKOUT reload anim: " .. tostring(errRL)) end
		local okFA, errFA = pcall(FA.update, now, vm, def, hide, ease)
		if not okFA and not FA.warned then FA.warned = true warn("BLACKOUT arms: " .. tostring(errFA)) end
		weaponStep(now)
	end)

	-- hub camera (menu background) -----------------------------------
	RunService:BindToRenderStep("BO_HubCam", Enum.RenderPriority.Camera.Value + 2, function()
		if not inMatch() and menuHolder.Visible then
			camera.CameraType = Enum.CameraType.Scriptable
			local t = os.clock() * 0.07
			local c = CFG.HUB_POS + Vector3.new(0, 10, 0)
			camera.CFrame = CFrame.lookAt(c + Vector3.new(math.cos(t) * 75, 26, math.sin(t) * 75), c)
			camera.FieldOfView = 60
			UI.camScripted = true
		elseif UI.camScripted then
			UI.camScripted = false
			camera.CameraType = Enum.CameraType.Custom
			camera.FieldOfView = 70
			local h = myHum()
			if h then camera.CameraSubject = h end
		end
	end)

	-- =================================================================
	-- MOBILE ON-SCREEN BUTTONS (positions / size / opacity from S, editable)
	-- =================================================================
	local touchGui = new("Frame", { Name = "TouchControls", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, ZIndex = 3 }, gui)
	local touchBtns = {}
	for _, def in ipairs(MOBILE_BUTTONS) do
		local b = new("TextButton", { AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = COL.panel2, Text = def.label, Font = Enum.Font.GothamBold,
			TextColor3 = COL.text, TextSize = 14, AutoButtonColor = false, ZIndex = 3 }, touchGui)
		corner(b, 100)
		stroke(b, COL.orange, 2)
		touchBtns[def.id] = b
		b.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
				dispatch(def.id, true)
				if def.id ~= "Sprint" and def.id ~= "ADS" then
					b.BackgroundColor3 = COL.orange
				end
				if def.id == "Fire" or def.id == "Slide" or def.id == "Reload" or def.id == "Swap" then
					input.Changed:Connect(function()
						if input.UserInputState == Enum.UserInputState.End or input.UserInputState == Enum.UserInputState.Cancel then
							dispatch(def.id, false)
							b.BackgroundColor3 = COL.panel2
						end
					end)
				else
					task.delay(0.15, function() b.BackgroundColor3 = COL.panel2 end)
				end
			end
		end)
	end
	local function layoutTouch()
		local vp = camera.ViewportSize
		for _, def in ipairs(MOBILE_BUTTONS) do
			local b = touchBtns[def.id]
			local p = S.mobile[def.id]
			local sz = def.size * S.g.mobScale
			b.Size = UDim2.fromOffset(sz, sz)
			b.Position = UDim2.fromOffset(p[1] * vp.X, p[2] * vp.Y)
			b.BackgroundTransparency = 1 - S.g.mobOpacity * 0.6
			b.TextTransparency = 1 - S.g.mobOpacity
		end
	end
	table.insert(settingsHooks, layoutTouch)
	camera:GetPropertyChangedSignal("ViewportSize"):Connect(layoutTouch)

	-- menu button for exploring the hub (also the only way back on touch)
	local menuToggleBtn = button(gui, "MENU", UDim2.fromOffset(90, 36), UDim2.fromOffset(12, 12), COL.panel, function()
		UI.toggleMenu()
	end, 14)
	menuToggleBtn.ZIndex = 4
	menuToggleBtn.Visible = false
	local exploreHint = label(gui, { Size = UDim2.fromOffset(400, 24), Position = UDim2.new(0.5, -200, 1, -40), Text = "", TextSize = 14, TextColor3 = COL.dim, ZIndex = 4, Visible = false })

	-- =================================================================
	-- SLOW HUD LOOP (health, timer, scores, reload bar, death timer, tags)
	-- =================================================================
	table.insert(settingsHooks, function()
		MS.hudScale.Scale = S.g.hud * 0.84
		MS.fitHud()
		HUD.updateCross(HUD.spread)
		exploreHint.Text = "PRESS [" .. prettyKey(S.keys.Menu) .. "] FOR MENU"
	end)

	local function myRow()
		for _, r in ipairs(HUD.rows) do
			if r.n == player.Name or r.n == player.DisplayName then return r end
		end
	end
	local tagAcc, lightAcc = 0, 0
	RunService.Heartbeat:Connect(function(dt)
		local match = inMatch()
		local hum = myHum()
		touchGui.Visible = match and UI.touch and not UI.blocking() and isAlive()
		menuToggleBtn.Visible = (not match and UI.explore) or (match and UI.touch and not UI.blocking())
		exploreHint.Visible = (not match) and UI.explore and not UI.touch
		-- hide our own world-gun so it never floats in first person
		local ch = player.Character
		if ch and match and os.clock() - (XT.shadowAt or 0) > 1 then
			XT.shadowAt = os.clock()
			for _, d in ipairs(ch:GetDescendants()) do
				if d:IsA("BasePart") and d.CastShadow then d.CastShadow = false end
			end
		end
		if ch and match and os.clock() - (XT.opHideAt or 0) > 0.25 then
			XT.opHideAt = os.clock()
			for _, d in ipairs(ch:GetChildren()) do
				if d:IsA("BasePart") and string.sub(d.Name, 1, 2) == "Op" then d.LocalTransparencyModifier = 1 end
			end
		end
		if ch and match then
			local g = ch:FindFirstChild("BO_Gun")
			if g then
				for _, p in ipairs(g:GetDescendants()) do
					if p:IsA("BasePart") then p.LocalTransparencyModifier = 1 end
				end
			end
		end
		if not match then return end
		-- health
		local hp = hum and math.max(0, hum.Health) or 0
		local mx = hum and hum.MaxHealth or 100
		HUD.hpSet(hp / mx, hp)
		HUD.lvl.Text = "LEVEL " .. P.level
		if HUD.updateOpCard then pcall(HUD.updateOpCard) end
		-- timer + score
		local ph = phase()
		local ends = State:GetAttribute("EndsAt")
		if ph == "MATCH" and type(ends) == "number" then
			HUD.timer.Text = fmtTime(ends - serverNow())
		elseif ph == "COUNTDOWN" and type(ends) == "number" then
			HUD.timer.Text = tostring(math.max(0, math.ceil(ends - serverNow())))
		else
			HUD.timer.Text = "--:--"
		end
		local mode = MODES[State:GetAttribute("ModeId") or ""]
		HUD.modeTxt.Text = mode and mode.name or ""
		local limit = State:GetAttribute("Limit") or 0
		if mode and mode.id == "FREE" then
			HUD.scoreA.Text = ""
			HUD.scoreB.Text = ""
			HUD.killsA.Text, HUD.killsB.Text = "", ""
		elseif mode and mode.teams then
			local sa, sb = State:GetAttribute("ScoreAlpha") or 0, State:GetAttribute("ScoreBravo") or 0
			if player:GetAttribute("Team") == "Bravo" then sa, sb = sb, sa end
			HUD.scoreA.Text = tostring(sa)
			HUD.scoreB.Text = tostring(sb)
			HUD.scoreA.TextColor3 = COL.text
			HUD.scoreB.TextColor3 = COL.red
		else
			local r = myRow()
			HUD.scoreA.Text = tostring(r and r.s or 0)
			HUD.scoreA.TextColor3 = COL.orange
			HUD.scoreB.Text = "/ " .. tostring(limit)
			HUD.scoreB.TextColor3 = COL.dim
		end
		-- reload bar
		local now = os.clock()
		if now < W.reloadUntil and W.reloadDur then
			HUD.reloadBar.Visible = true
			HUD.reloadBar.Size = UDim2.fromOffset(360 * math.clamp((now - (W.reloadStart or now)) / W.reloadDur, 0, 1), 4)
		else
			HUD.reloadBar.Visible = false
		end
		-- crosshair
		local def = curDef()
		local root = myRoot()
		if def and root then
			local sp = (root.AssemblyLinearVelocity * Vector3.new(1, 0, 1)).Magnitude
			local px = ((W.ads and def.ads or def.hip) * 4) + sp * 0.35 + R.pitch * 2.5
			if hum and hum.FloorMaterial == Enum.Material.Air then px += 8 end
			HUD.spread += (px - HUD.spread) * math.min(1, dt * 14)
			HUD.updateCross(HUD.spread)
			HUD.cross.Visible = not ((def.scope or def.scopeOpt) and W.adsT > 0.5) and not HUD.opticOn and W.adsT < 0.3 and not UI.blocking()
		end
		-- death timer
		if HUD.death.Visible then
			local left = HUD.deadUntil - serverNow()
			HUD.deathTimer.Text = left > 0 and string.format("Respawning in %.1f", left) or "Respawning..."
		end
		-- opponents are marked RED (outline + name tag); teammates get nothing
		tagAcc += dt
		if tagAcc > 0.4 then
			tagAcc = 0
			local teams = mode and mode.teams
			local myTeam = player:GetAttribute("Team")
			local RED = RGB(255, 48, 48)
			local function mark(model)
				-- no enemy outlines, glows or health bars any more: strip anything left over
				if not model then return end
				for _, nm in ipairs({ "BO_BotHL", "BO_EnemyHL", "BO_TeamHL" }) do
					local o = model:FindFirstChild(nm)
					if o then o:Destroy() end
				end
				local t = model:FindFirstChild("BO_EnemyTag")
				if t then t:Destroy() end
			end
			for _, pl in ipairs(Players:GetPlayers()) do
				if pl ~= player then
					local c = pl.Character
					if c then
						local th = c:FindFirstChild("BO_TeamHL")
						if th then th:Destroy() end
						mark(c)
					end
				end
			end
			local bf = Workspace:FindFirstChild("BO_Bots")
			if bf then
				for _, m in ipairs(bf:GetChildren()) do mark(m) end
			end
		end
	end)

	-- ---------- killstreak UAV: radar + through-wall outlines ----------
	do
		local R_PX, RANGE = 84, 150
		local radar = new("Frame", { Size = UDim2.fromOffset(R_PX * 2, R_PX * 2), Position = UDim2.fromOffset(32, 350), BackgroundColor3 = RGB(6, 14, 10),
			BackgroundTransparency = 0.25, Visible = false, ClipsDescendants = true }, hud)
		new("UICorner", { CornerRadius = UDim.new(1, 0) }, radar)
		stroke(radar, COL.green, 2, 0.2)
		new("Frame", { Size = UDim2.new(1, 0, 0, 1), Position = UDim2.fromScale(0, 0.5), BackgroundColor3 = COL.green, BackgroundTransparency = 0.75, BorderSizePixel = 0 }, radar)
		new("Frame", { Size = UDim2.new(0, 1, 1, 0), Position = UDim2.fromScale(0.5, 0), BackgroundColor3 = COL.green, BackgroundTransparency = 0.75, BorderSizePixel = 0 }, radar)
		new("Frame", { Size = UDim2.fromOffset(8, 8), Position = UDim2.new(0.5, -4, 0.5, -4), BackgroundColor3 = COL.cyan, BorderSizePixel = 0, Rotation = 45 }, radar)
		local sweep = new("Frame", { Size = UDim2.new(1, 0, 0, 2), Position = UDim2.fromScale(0, 0), BackgroundColor3 = COL.green, BackgroundTransparency = 0.6, BorderSizePixel = 0 }, radar)
		local tag = label(hud, { Size = UDim2.fromOffset(R_PX * 2, 18), Position = UDim2.fromOffset(32, 350 + R_PX * 2 + 4), Text = "", TextSize = 13, TextColor3 = COL.green, Visible = false })
		local dots = {}
		for i = 1, 20 do
			dots[i] = new("Frame", { Size = UDim2.fromOffset(7, 7), BackgroundColor3 = COL.red, BorderSizePixel = 0, Visible = false }, radar)
			new("UICorner", { CornerRadius = UDim.new(1, 0) }, dots[i])
		end
		local acc, hlAcc = 0, 0
		local function enemies()
			local out = {}
			local myTeam = player:GetAttribute("Team")
			local function consider(model, team)
				if not model then return end
				local r = model:FindFirstChild("HumanoidRootPart")
				local h = model:FindFirstChildOfClass("Humanoid")
				if r and h and h.Health > 0 and (team == "FFA" or team ~= myTeam) then table.insert(out, { model = model, pos = r.Position }) end
			end
			for _, pl in ipairs(Players:GetPlayers()) do
				if pl ~= player and pl:GetAttribute("InMatch") then consider(pl.Character, pl:GetAttribute("Team")) end
			end
			local bf = Workspace:FindFirstChild("BO_Bots")
			if bf then
				for _, m in ipairs(bf:GetChildren()) do consider(m, m:GetAttribute("Team")) end
			end
			return out
		end
		RunService.Heartbeat:Connect(function(dt)
			local left = (player:GetAttribute("UavUntil") or 0) - Workspace:GetServerTimeNow()
			local on = left > 0 and inMatch() and isAlive() and hud.Visible
			radar.Visible, tag.Visible = on, on
			if not on then
				for _, d in ipairs(dots) do d.Visible = false end
				if hlAcc > 0 then
					hlAcc = 0
					for _, pl in ipairs(Players:GetPlayers()) do
						local c = pl.Character
						local h = c and c:FindFirstChild("BO_UavHL")
						if h then h:Destroy() end
					end
					local bf = Workspace:FindFirstChild("BO_Bots")
					if bf then for _, m in ipairs(bf:GetChildren()) do local h = m:FindFirstChild("BO_UavHL"); if h then h:Destroy() end end end
				end
				return
			end
			local tier = player:GetAttribute("UavTier") or 1
			tag.Text = string.format("%s  %ds", tier >= 2 and "SUPER UAV" or "UAV", math.ceil(left))
			sweep.Position = UDim2.fromScale(0, (os.clock() % 2) / 2)
			acc += dt
			if acc < 0.08 then return end
			acc = 0
			local root = myRoot()
			if not root then return end
			local cf = camera.CFrame
			local fwd = Vector3.new(cf.LookVector.X, 0, cf.LookVector.Z)
			if fwd.Magnitude < 0.05 then return end
			fwd = fwd.Unit
			local right = Vector3.new(-fwd.Z, 0, fwd.X) * -1
			right = Vector3.new(fwd.Z, 0, -fwd.X)
			local list = enemies()
			for i, d in ipairs(dots) do
				local e = list[i]
				if e then
					local rel = e.pos - root.Position
					local x, y = rel:Dot(right), rel:Dot(fwd)
					local v = Vector2.new(x, -y) / RANGE
					if v.Magnitude > 1 then v = v.Unit end
					d.Position = UDim2.fromOffset(R_PX + v.X * (R_PX - 6) - 3, R_PX + v.Y * (R_PX - 6) - 3)
					d.Visible = true
				else
					d.Visible = false
				end
			end
			if tier >= 2 then
				hlAcc += 0.08
				if hlAcc > 0.4 then
					hlAcc = 0.0001
					for _, e in ipairs(list) do
						if not e.model:FindFirstChild("BO_UavHL") then
							new("Highlight", { Name = "BO_UavHL", FillColor = COL.red, OutlineColor = COL.red, FillTransparency = 0.7, OutlineTransparency = 0.1,
								DepthMode = Enum.HighlightDepthMode.AlwaysOnTop }, e.model)
						end
					end
				end
			end
		end)
	end

	-- lighting follows InMatch / map (once a second as a safety net)
	RunService.Heartbeat:Connect(function(dt)
		lightAcc += dt
		if lightAcc > 1 then
			lightAcc = 0
			refreshLighting(false)
			if not inMatch() and player.Character then
				local h = myHum()
				if h and h.WalkSpeed ~= CFG.BASE_WALK then h.WalkSpeed = CFG.BASE_WALK end
			end
		end
	end)

	-- character resets
	local function onChar(char)
		W.ads, W.sprinting, W.sliding, W.crouch, W.burstQ = false, false, false, false, 0
		W.reloadUntil, W.swapUntil = 0, 0
		R.pitch, R.yaw = 0, 0
		for k in pairs(Held) do Held[k] = false end
		touchToggles.ADS, touchToggles.Sprint = false, false
		HUD.scope.Visible = false
		char:WaitForChild("Humanoid", 10)
		UI.refresh()
	end
	player.CharacterAdded:Connect(onChar)
	if player.Character then task.spawn(onChar, player.Character) end

	-- =================================================================
	-- SMOOTHNESS: adaptive quality governor, hub spinners, soldier animation
	-- (Roblox itself caps rendering at 60 FPS; this keeps weak devices near it)
	-- =================================================================
	local function applyQuality()
		ensureFx()
		lightObjs.bloom.Enabled = false
		lightObjs.cc.Enabled = Q.level >= 2
	end
	RunService.RenderStepped:Connect(function(dt)
		Q.frames += 1
		Q.acc += dt
		if Q.acc < 2 then return end
		local fps = Q.frames / Q.acc
		Q.frames, Q.acc = 0, 0
		local manualQ = math.floor((S.g.quality or 0) + 0.5)
		if manualQ >= 1 and Q.level ~= manualQ then
			Q.level = manualQ
			applyQuality()
		end
		if not inMatch() or manualQ >= 1 then
			Q.low, Q.high = 0, 0
			return
		end
		if fps < 36 then
			Q.low += 1
			Q.high = 0
		elseif fps >= 57 then
			Q.high += 1
			Q.low = 0
		else
			Q.low, Q.high = 0, 0
		end
		local now = os.clock()
		if Q.low >= 2 and Q.level > 1 then
			Q.level -= 1
			Q.low = 0
			Q.changedAt = now
			applyQuality()
		elseif Q.high >= 10 and Q.level < Q.max and now - Q.changedAt > 60 then
			Q.level += 1
			Q.high = 0
			Q.changedAt = now
			applyQuality()
		end
	end)

	-- hub spinners (parts tagged with a "Spin" attribute by the server are turned locally = smooth, no network traffic)
	local spinParts = {}
	local function scanSpin()
		table.clear(spinParts)
		local world = Workspace:FindFirstChild("BO_World")
		local hubF = world and world:FindFirstChild("BO_Hub")
		if not hubF then return end
		for _, d in ipairs(hubF:GetDescendants()) do
			if d:IsA("BasePart") then
				local sp = d:GetAttribute("Spin")
				if sp then table.insert(spinParts, { p = d, s = sp }) end
			end
		end
	end
	task.spawn(function()
		while true do
			scanSpin()
			task.wait(8)
		end
	end)
	RunService.Heartbeat:Connect(function(dt)
		if inMatch() then return end
		for _, sp in ipairs(spinParts) do
			if sp.p.Parent then sp.p.CFrame = sp.p.CFrame * CFrame.Angles(0, dt * sp.s, 0) end
		end
	end)

	-- soldier animation: runs locally every frame from replicated velocity (walk cycle, strafing, lean, breathing, recoil)
	local rigs = setmetatable({}, { __mode = "k" })
	local function getRig(m)
		local r = rigs[m]
		if r then return r end
		local root = m.PrimaryPart or m:FindFirstChild("HumanoidRootPart")
		local function j(name)
			local part = m:FindFirstChild(name)
			return part and part:FindFirstChild("Joint")
		end
		local torso, lArm, rArm, lLeg, rLeg = j("Torso"), j("LeftArm"), j("RightArm"), j("LeftLeg"), j("RightLeg")
		if not (root and torso and lArm and rArm and lLeg and rLeg) then return nil end
		r = { root = root, torso = torso, lArm = lArm, rArm = rArm, lLeg = lLeg, rLeg = rLeg, ph = math.random() * 6.28, spd = 0, kick = 0,
			shot = m:GetAttribute("Shot") or 0, t = math.random() * 10 }
		rigs[m] = r
		return r
	end
	RunService.Heartbeat:Connect(function(dt)
		local folder = Workspace:FindFirstChild("BO_Bots")
		if not folder then return end
		local camPos = camera.CFrame.Position
		for _, m in ipairs(folder:GetChildren()) do
			local hum = m:FindFirstChildOfClass("Humanoid")
			if hum and hum.Health > 0 then
				local r = getRig(m)
				if r and r.root.Parent and (r.root.Position - camPos).Magnitude < 260 then
					local vel = r.root.AssemblyLinearVelocity * Vector3.new(1, 0, 1)
					local spd = vel.Magnitude
					r.spd += (spd - r.spd) * math.min(1, dt * 9)
					local n = math.clamp(r.spd / 14, 0, 1.15)
					r.t += dt
					r.ph += dt * (2.2 + r.spd * 0.78)
					local lv = r.root.CFrame:VectorToObjectSpace(vel)
					local fwd = spd > 0.8 and math.clamp(-lv.Z / spd, -1, 1) or 0
					local side = spd > 0.8 and math.clamp(lv.X / spd, -1, 1) or 0
					local sw = math.sin(r.ph)
					local amp = 0.85 * n
					local sgn = (fwd < 0) and -1 or 1
					local legX = sw * amp * math.abs(fwd) * sgn
					local legZ = sw * amp * side * 0.8
					r.lLeg.Transform = CFrame.Angles(legX, 0, legZ)
					r.rLeg.Transform = CFrame.Angles(-legX, 0, -legZ)
					local airborne = hum.FloorMaterial == Enum.Material.Air
					if airborne then
						r.lLeg.Transform = CFrame.Angles(0.5, 0, 0)
						r.rLeg.Transform = CFrame.Angles(-0.3, 0, 0)
					end
					local sh = m:GetAttribute("Shot") or 0
					if sh ~= r.shot then
						r.shot = sh
						r.kick = 1
					end
					r.kick = math.max(0, r.kick - dt * 7)
					local bob = math.abs(math.cos(r.ph)) * 0.12 * n + math.sin(r.t * 1.7) * 0.025
					r.torso.Transform = CFrame.new(0, bob, 0) * CFrame.Angles(-fwd * 0.13 * n + r.kick * 0.05, 0, -side * 0.1 * n)
					local armSway = sw * 0.045 * n + math.sin(r.t * 1.7) * 0.012
					r.rArm.Transform = CFrame.Angles(r.kick * 0.2 + armSway, 0, 0)
					r.lArm.Transform = CFrame.Angles(r.kick * 0.12 - armSway, 0, 0)
				end
			end
		end
	end)

	-- =================================================================
	-- BOOT
	-- =================================================================
	if RunService:IsStudio() then
		-- Studio-only photo mode: set workspace attribute BO_CAM (a CFrame) to take clean map screenshots
		RunService:BindToRenderStep("BO_PHOTO", Enum.RenderPriority.Last.Value + 1, function()
			local c = workspace:GetAttribute("BO_CAM")
			if typeof(c) == "CFrame" then
				camera.CameraType = Enum.CameraType.Scriptable
				camera.CFrame = c
				camera.FieldOfView = workspace:GetAttribute("BO_FOV") or 70
				gui.Enabled = false
				for _, d in ipairs(camera:GetChildren()) do if d:IsA("Model") or d:IsA("BasePart") then d:Destroy() end end
			end
		end)
	end
	applyQuality()
	settingsChanged()
	refreshProfileUI()
	showPage(nil)
	applyLighting("HUB", true)
	UI.refresh()
	updateAmmoHud()
	Net.ClientReady:FireServer()
	task.delay(4, function()
		-- if the profile has not arrived yet, ask again
		if not settingsLoaded then Net.ClientReady:FireServer() end
	end)
	print("BLACKOUT: client ready.")
end

-- =====================================================================
-- ENTRY
-- =====================================================================
if IS_SERVER then
	runServer()
else
	runClient()
end
