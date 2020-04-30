class SwayHandler : StaticEventHandler
{
	SwayManager sway;
	
	override void WorldLoaded(WorldEvent e)
	{
		ThinkerIterator it = ThinkerIterator.Create("SwayManager", Thinker.MAX_STATNUM);
		sway = SwayManager(it.Next());
		
		if (!sway)
		{
			sway = new("SwayManager");
			sway.ChangeStatNum(Thinker.MAX_STATNUM);
			
			for (uint i = 0; i < MAXPLAYERS; ++i)
			{
				if (!playerInGame[i] || !players[i].mo)
					continue;
				
				sway.prevAngle[i] = players[i].mo.angle;
				sway.prevPitch[i] = players[i].mo.pitch;
				if (!players[i].ReadyWeapon)
				{
					sway.prevY[i] = WEAPONBOTTOM;
					continue;
				}
				
				let psp = players[i].GetPSPrite(PSP_WEAPON);
				sway.prevX[i] = psp.x;
				sway.prevY[i] = psp.y;
			}
		}
		
		sway.CheckCVars();
	}
}

struct LayerManager
{
	Array<PSprite> layers;
	Array<double> prevX;
	Array<double> prevY;
}

class SwayManager : Thinker
{
	double prevAngle[MAXPLAYERS];
	double prevPitch[MAXPLAYERS];
	double prevX[MAXPLAYERS];
	double prevY[MAXPLAYERS];
	
	LayerManager lm[MAXPLAYERS];
	
	double xOffset[MAXPLAYERS];
	double yOffset[MAXPLAYERS];
	double vOffset[MAXPLAYERS];
	
	// General
	
	transient CVar fireSway[MAXPLAYERS];
	transient CVar bobSway[MAXPLAYERS];
	transient CVar noSwayBob[MAXPLAYERS];
	
	// Vertical offset
	
	transient CVar vExtend[MAXPLAYERS];
	transient CVar rExtend[MAXPLAYERS];
	transient CVar vInverse[MAXPLAYERS];
	transient CVar crouchVInverse[MAXPLAYERS];
	transient CVar vertScale[MAXPLAYERS];
	transient CVar crouchVertScale[MAXPLAYERS];
	
	// Horizontal sway
	
	transient CVar swayHInverse[MAXPLAYERS];
	transient CVar hCumulative[MAXPLAYERS];
	transient CVar swayHCrouch[MAXPLAYERS];
	transient CVar swayHScalar[MAXPLAYERS];
	transient CVar swayHSpeed[MAXPLAYERS];
	transient CVar swayHAccuracy[MAXPLAYERS];
	
	// Vertical sway
	
	transient CVar swayVInverse[MAXPLAYERS];
	transient CVar vCumulative[MAXPLAYERS];
	transient CVar swayVCrouch[MAXPLAYERS];
	transient CVar swayVScalar[MAXPLAYERS];
	transient CVar swayVSpeed[MAXPLAYERS];
	transient CVar swayVAccuracy[MAXPLAYERS];
	
	// Movement
	
	transient CVar vMoveSway[MAXPLAYERS];
	transient CVar hMoveSway[MAXPLAYERS];
	transient CVar fMoveSway[MAXPLAYERS];
	
	override void Tick()
	{
		for (uint i = 0; i < MAXPLAYERS; ++i)
		{
			if (!playerInGame[i] || !players[i].mo)
				continue;
			
			if (!players[i].ReadyWeapon)
			{
				prevAngle[i] = players[i].mo.angle;
				prevPitch[i] = players[i].mo.pitch;
				xOffset[i] = 0;
				yOffset[i] = 0;
				vOffset[i] = 0;
				prevX[i] = 0;
				prevY[i] = WEAPONBOTTOM;
				continue;
			}
			
			// This is a best guesstimate on reversing the effects
			let psp = players[i].GetPSPrite(PSP_WEAPON);
			
			psp.oldx = prevX[i];
			if (psp.x ~== prevX[i])
				psp.x += xOffset[i];
			
			psp.oldy = prevY[i];
			if ((psp.y ~== prevY[i] && prevY[i] != WEAPONTOP) || psp.y > WEAPONTOP || (vExtend[i].GetBool() && psp.y < WEAPONTOP))
				psp.y -= (yOffset[i] + vOffset[i]);
			
			for (uint j = 0; j < lm[i].layers.Size(); ++j)
			{
				let layer = lm[i].layers[j];
				if (layer)
				{
					layer.oldx = lm[i].prevX[j];
					layer.oldy = lm[i].prevY[j];
					if (layer.bMirror && layer.x ~== lm[i].prevX[j])
						layer.x -= xOffset[i]*2;
				}
			}
			
			lm[i].layers.Clear();
			lm[i].prevX.Clear();
			lm[i].prevY.Clear();
			
			for (let pspr = players[i].psprites; pspr; pspr = pspr.next)
			{
				if (pspr.id == PSP_WEAPON)
					continue;
				
				if (pspr.bAddWeapon)
					lm[i].layers.Push(pspr);
			}
			
			bool bob = (players[i].WeaponState & WF_WEAPONBOBBING);
			bool doReset = ((bobSway[i].GetBool() && !bob) ||
							(fireSway[i].GetBool() && players[i].attackdown) ||
							(noSwayBob[i].GetBool() && bob && !(players[i].vel ~== (0,0))));
			
			// Horizontal sway
			
			double hScalar = swayHScalar[i].GetFloat();
			if (players[i].crouchFactor < 1)
				hScalar *= players[i].crouchFactor * swayHCrouch[i].GetFloat();
			
			bool hReset;
			if (hScalar ~== 0 && !(xOffset[i] ~== 0))
			{
				hReset = true;
				hScalar = 1;
			}
			
			double hRange = 32 * hScalar;
			if (abs(xOffset[i]) > hRange)
				hReset = true;
			
			double deltaAng = Actor.DeltaAngle(players[i].mo.angle, prevAngle[i]) * 2 * hScalar;
			if (!(deltaAng ~== 0) && players[i].cmd.yaw && !players[i].turnticks)
				deltaAng = -players[i].cmd.yaw * (360/65536.) * 2 * hScalar;
			
			if (!hMoveSway[i].GetBool())
			{
				Vector2 right = Actor.AngleToVector(players[i].mo.angle-90);
				deltaAng += (players[i].mo.vel.xy dot right) * 1.5 * hScalar;
			}
			
			if (swayHInverse[i].GetBool())
				deltaAng *= -1;
			
			if (hReset || doReset)
				deltaAng = 0;
			
			double hSpeed = swayHSpeed[i].GetFloat() * hScalar;
			double hDiff = abs(deltaAng - xOffset[i]);
			if (hCumulative[i].GetBool() && !(deltaAng ~== 0))
			{
				if (hDiff >= swayHAccuracy[i].GetFloat()*hScalar)
				{
					if (deltaAng > 0)
						xOffset[i] += hSpeed;
					else
						xOffset[i] -= hSpeed;
				}
			}
			else
			{
				if (hDiff >= 8*hScalar)
					hSpeed *= 2;
				
				if (deltaAng ~== 0 || hDiff >= swayHAccuracy[i].GetFloat()*hScalar)
				{
					if (xOffset[i] > deltaAng)
					{
						xOffset[i] -= hSpeed;
							
						if (xOffset[i] < deltaAng)
							xOffset[i] = deltaAng;
					}
					else if (xOffset[i] < deltaAng)
					{
						xOffset[i] += hSpeed;
							
						if (xOffset[i] > deltaAng)
							xOffset[i] = deltaAng;
					}
				}
			}
			
			if (!hReset)
				xOffset[i] = clamp(xOffset[i], -hRange, hRange);
			
			psp.x -= xOffset[i];
			
			for (uint j = 0; j < lm[i].layers.Size(); ++j)
			{
				let layer = lm[i].layers[j];
				if (layer)
				{
					if (layer.bMirror)
						layer.x += xOffset[i]*2;
					
					lm[i].prevX.Push(layer.x);
					lm[i].prevY.Push(layer.y);
				}
				else
				{
					lm[i].prevX.Push(0);
					lm[i].prevY.Push(0);
				}
			}
			
			// Vertical sway
			
			double vScalar = swayVScalar[i].GetFloat();
			if (players[i].crouchFactor < 1)
				vScalar *= players[i].crouchFactor * swayVCrouch[i].GetFloat();
			
			bool vReset;
			if (vScalar ~== 0 && !(yOffset[i] ~== 0))
			{
				vReset = true;
				vScalar = 1;
			}
			
			double vRange = 16 * vScalar;
			if (abs(yOffset[i]) > vRange)
				vReset = true;
			
			double deltaPch = Actor.DeltaAngle(players[i].mo.pitch, prevPitch[i]) * 2 * vScalar;
			if (!(deltaPch ~== 0) && players[i].cmd.pitch && !players[i].centering)
				deltaPch = players[i].cmd.pitch * (360/65536.) * 2 * vScalar;
			
			if (!vMoveSway[i].GetBool() || !fMoveSway[i].GetBool())
			{
				double amnt;
				
				if (!vMoveSway[i].GetBool())
				{
					Vector2 forward = Actor.AngleToVector(players[i].mo.angle);
					amnt += (players[i].mo.vel.xy dot forward) * 0.5;
				}
				
				if (!fMoveSway[i].GetBool())
					amnt += (players[i].mo.vel dot (0,0,1));
				
				deltaPch += amnt * vScalar;
			}
			
			if (swayVInverse[i].GetBool())
				deltaPch *= -1;
			
			if (vReset || doReset || psp.y > WEAPONTOP)
				deltaPch = 0;
			
			double vSpeed = swayVSpeed[i].GetFloat() * vScalar;
			double vDiff = abs(deltaPch - yOffset[i]);
			if (vCumulative[i].GetBool() && !(deltaPch ~== 0))
			{
				if (vDiff >= swayVAccuracy[i].GetFloat()*vScalar)
				{
					if (deltaPch > 0)
						yOffset[i] += vSpeed;
					else
						yOffset[i] -= vSpeed;
				}
			}
			else
			{
				if (vDiff >= 8*vScalar)
					vSpeed *= 2;
				
				if (deltaPch ~== 0 || vDiff >= swayVAccuracy[i].GetFloat()*vScalar)
				{
					if (yOffset[i] > deltaPch)
					{
						yOffset[i] -= vSpeed;
						
						if (yOffset[i] < deltaPch)
							yOffset[i] = deltaPch;
					}
					else if (yOffset[i] < deltaPch)
					{
						yOffset[i] += vSpeed;
						
						if (yOffset[i] > deltaPch)
							yOffset[i] = deltaPch;
					}
				}
			}
			
			if (!vReset)
				yOffset[i] = clamp(yOffset[i], -vRange, vRange);
			
			// Vertical offset
			
			bool ext = vExtend[i].GetBool();
			double pchOfs = players[i].mo.pitch;
			if (!ext)
				pchOfs += 90;
			
			if (vInverse[i].GetBool())
				pchOfs = ext ? pchOfs*-1 : 180 - pchOfs;
			
			if (players[i].crouchFactor < 1)
			{
				double convert = 45 + (ext ? pchOfs+90 : pchOfs) / 2;
				if (ext)
					convert -= 90;
				
				double ofs = (pchOfs - convert)*(1 - players[i].crouchFactor) * 2 * crouchVertScale[i].GetFloat();
				if (crouchVInverse[i].GetBool())
					ofs *= -1;
				
				if (ext)
					pchOfs = clamp(pchOfs - ofs, -90, 90);
				else
					pchOfs = clamp(pchOfs - ofs, 0, 180);
			}
			
			vOffset[i] = pchOfs / 9 * vertScale[i].GetFloat();
			
			if (ext && rExtend[i].GetBool() &&
			(!(players[i].WeaponState & (WF_WEAPONREADY|WF_WEAPONREADYALT)) || players[i].attackdown) && vOffset[i] < 0)
			{
				vOffset[i] = 0;
			}
			
			psp.y += vOffset[i];
			
			// Add y offset after vertical scaling
			
			if (!ext && psp.y + yOffset[i] < WEAPONTOP)
				yOffset[i] = min(0, WEAPONTOP - psp.y);
			
			psp.y += yOffset[i];
			
			prevAngle[i] = players[i].mo.angle;
			prevPitch[i] = players[i].mo.pitch;
			prevX[i] = psp.x;
			prevY[i] = psp.y;
		}
	}
	
	void CheckCVars()
	{
		for (uint i = 0; i < MAXPLAYERS; ++i)
		{
			if (!playerInGame[i] || !players[i].mo)
				continue;
			
			if (!fireSway[i])
				fireSway[i] = CVar.GetCVar("ws_disablefire", players[i]);
			if (!bobSway[i])
				bobSway[i] = CVar.GetCVar("ws_bobonly", players[i]);
			if (!noSwayBob[i])
				noSwayBob[i] = CVar.GetCVar("ws_nobobsway", players[i]);
			
			if (!vExtend[i])
				vExtend[i] = CVar.GetCVar("ws_vertextend", players[i]);
			if (!rExtend[i])
				rExtend[i] = CVar.GetCVar("ws_extendready", players[i]);
			if (!vInverse[i])
				vInverse[i] = CVar.GetCVar("ws_inversevert", players[i]);
			if (!crouchVInverse[i])
				crouchVInverse[i] = CVar.GetCVar("ws_crouchinversevert", players[i]);
			if (!vertScale[i])
				vertScale[i] = CVar.GetCVar("ws_vertscale", players[i]);
			if (!crouchVertScale[i])
				crouchVertScale[i] = CVar.GetCVar("ws_crouchvertscale", players[i]);
			
			if (!swayHInverse[i])
				swayHInverse[i] = CVar.GetCVar("ws_swayhinverse", players[i]);
			if (!hCumulative[i])
				hCumulative[i] = CVar.GetCVar("ws_swayhcumulative", players[i]);
			if (!swayHScalar[i])
				swayHScalar[i] = CVar.GetCVar("ws_swayhscale", players[i]);
			if (!swayHCrouch[i])
				swayHCrouch[i] = CVar.GetCVar("ws_swayhcrouchscale", players[i]);
			if (!swayHSpeed[i])
				swayHSpeed[i] = CVar.GetCVar("ws_swayhspeed", players[i]);
			if (!swayHAccuracy[i])
				swayHAccuracy[i] = CVar.GetCVar("ws_swayhaccuracy", players[i]);
			
			if (!swayVInverse[i])
				swayVInverse[i] = CVar.GetCVar("ws_swayvinverse", players[i]);
			if (!vCumulative[i])
				vCumulative[i] = CVar.GetCVar("ws_swayvcumulative", players[i]);
			if (!swayVScalar[i])
				swayVScalar[i] = CVar.GetCVar("ws_swayvscale", players[i]);
			if (!swayVCrouch[i])
				swayVCrouch[i] = CVar.GetCVar("ws_swayvcrouchscale", players[i]);
			if (!swayVSpeed[i])
				swayVSpeed[i] = CVar.GetCVar("ws_swayvspeed", players[i]);
			if (!swayVAccuracy[i])
				swayVAccuracy[i] = CVar.GetCVar("ws_swayvaccuracy", players[i]);
			
			if (!vMoveSway[i])
				vMoveSway[i] = CVar.GetCVar("ws_novmove", players[i]);
			if (!hMoveSway[i])
				hMoveSway[i] = CVar.GetCVar("ws_nohmove", players[i]);
			if (!fMoveSway[i])
				fMoveSway[i] = CVar.GetCVar("ws_nofmove", players[i]);
		}
	}
}