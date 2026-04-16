// ─────────── inputs ───────────
// ADC pin mapping: ADC0 = Mode pin, ADC1 = XP1 pin, ADC2 = YP pin, ADC3 = SS pin
modeV, xp1V, ypV, ssV : real (volts, 0..5)

// ─────────── shared helpers ───────────
function VoltageToCode(v):          return clamp(round(v * 256 / 5), 0, 255)
function IsFixH(v):                 return v > 4.8
function IsFixL(v):                 return v < 0.195
function ClassifyRail(v):
    if IsFixH(v):                   return FIX_H
    if IsFixL(v):                   return FIX_L
    if code(v) in {10..19, 120..135, 236..245}:
                                    return FORBIDDEN
    if v < 0.39:                    return BELOW_LINEAR
    if v > 4.59:                    return ABOVE_LINEAR
    return LINEAR

function VoltageToMode(v):
    code = VoltageToCode(v)
    for (lo, hi, m) in MODE_BANDS:      // [(0,31,0),(32,63,1),...,(224,255,7)]
        if lo ≤ code ≤ hi:              return m
    return null                          // forbidden: step 120..136

// ─────────── entry point ───────────
function Calculate(modeV, xp1V, ypV, ssV):

    mode  ← VoltageToMode(modeV)
    if mode is null:                     return empty_result      // forbidden mode

    adc1  ← VoltageToCode(xp1V)      // ADC1 = XP1 pin
    adc2  ← VoltageToCode(ypV)       // ADC2 = YP  pin
    adc3  ← VoltageToCode(ssV)       // ADC3 = SS  pin

    xp1Fix ← IsFixH(xp1V) or IsFixL(xp1V)
    ypFix  ← IsFixH(ypV)  or IsFixL(ypV)
    ssFix  ← IsFixH(ssV)  or IsFixL(ssV)

    xp1Rail ← ClassifyRail(xp1V)
    ypRail  ← ClassifyRail(ypV)
    ssRail  ← ClassifyRail(ssV)

    // ── Field 1: cmd_inv  (reg_01[7], 1 bit) ──
    cmd_inv ← 0
    if mode == 0 and ypV > 2.5:				cmd_inv ← 1
    else if mode == 0 and xp1V > 2.5 and ypV < 2.5:	cmd_inv ← 1
    else if mode == 7 and xp1V < 2.5 and ypV < 2.5:	cmd_inv ← 1
    else						cmd_inv ← reg_data_01[7]

    // ── Field 2: fan_curve[1:0]  (reg_01[4:3], 2 bits) ──
    if mode in {0,1,2,3,7}:				fan_curve ← 2
    else if mode in {4,5,6}:				fan_curve ← 1      // mode ∈ {4,5,6}
    else						fan_curve ← reg_data_01[4:3]

    // ── Field 3: rpm_ratio[2:0]  (reg_01[2:0], 3 bits) ──
    // ADC3 = SS pin
    if mode in {0,1,2,3,7}:				rpm_ratio ← 7
    else if:                                                    // mode ∈ {4,5,6}
        adc3_top ← (adc3 >> 5) and 0x7                       // ADC3[7:5]
        if xp1Fix:					rpm_ratio ← adc3_top + 4
        else if xp1Rail == LINEAR:			rpm_ratio ← adc3_top
        else:						rpm_ratio ← 0    // reset value (xp1 non-linear)
    else:						rpm_ratio ← reg_data_01[2:0]

    // ── Field 4: min_sd  (reg_02[7], 1 bit) ──
    if mode in {0,1,4,7}:				min_sd ← 0
    else if mode in {2,3,5,6}:                                                    // mode ∈ {2,3,5,6}      
	if xp1V > 2.5:					min_sd ← 0  
	else:						min_sd ← 1
    else:						min_sd ← reg_data_02[7]

    // ── Field 5: xp1[4:0]  (reg_02[4:0], 5 bits) ──
    if mode in {0,1,2,3,7}:
        if xp1Fix:					xp1 ← 3
        else if xp1Rail == BELOW_LINEAR:		xp1 ← 0
        else if xp1Rail == ABOVE_LINEAR:		xp1 ← 0
        else:                                               // LINEAR
            adc1_low ← adc1 and 0x7F                         // ADC1[6:0]
	    adc1_low_inv ← ~adc1 and 0x7F                    // ~ADC1[6:0]
	    if xp1V > 2.5:				xp1 ← round((adc1_low_inv-20)/4)
	    else:					xp1 ← round((adc1_low-20)/4)    
    else if mode in {4,5,6}:				xp1 ← 26       // mode ∈ {4,5,6}
    else:						xp1 ← reg_data_02[4:0] x 16

    // ── Field 6: soft_sw[1:0]  (reg_03[7:6], 2 bits) ──
    if mode == 1					soft_sw ← 1
    else if mode in {0,2,3,4,5,6,7}			soft_sw ← 0
    else						soft_sw ← reg_data_03[7:6]

    // ── Field 7: yp1[6:0]  (reg_04[6:0], 7 bits) ──
    if mode in {0,1,2,3,7}:
        if ypFix:					yp1 ← 12
        else if ypRail == BELOW_LINEAR:			yp1 ← 0
        else if ypRail == ABOVE_LINEAR:			yp1 ← 0
        else:                                               // LINEAR
		adc2_low ← adc2 and 0x7F 
		adc2_low_inv ← ~adc2 and 0x7F 
		if ypV > 2.5:				yp1 ← adc2_low_inv-20
		else:					yp1 ← adc2_low-20
	    
    else if mode in {4,5,6}:				yp1 ← 26       // mode ∈ {4,5,6}
    else						yp1 ← reg_data_04[6:0] x 4

    // ── Field 8: yp2[6:0]  (reg_05[6:0], 7 bits) ──
    // yp2 永遠是常數，不依 ADC2；pptx slide 5 對齊。
							yp2 ← 127      

    // ── Field 9: icmd_sw  (reg_06[7], 1 bit) ──
    if mode in {0,1,2,3,4,5,6,7}			icmd_sw ← 0   // all modes
    else						icmd_sw ← reg_data_06[7]

    // ── Field 10: rp1[6:0]  (reg_06[6:0], 7 bits) ──
    // ADC1 = XP1 pin
    if mode in {4,5,6}:
        if xp1Fix:			rp1 ← 1    
	else:
		adc1_low ← adc1 and 0x7F                         // ADC1[6:0]
		adc1_low_inv ← ~adc1 and 0x7F                    // ~ADC1[6:0]
		if xp1V > 2.5:				rp1 ← adc1_low_inv-20 
		else:					rp1 ← adc1_low-20 
    else:						rp1 ← (reg_data_06[6:0]   // reset value

    // ── Field 11: rp2[7:0]  (reg_07[7:0], 8 bits) ──
    if mode in {4,5,6}:
	if ypFix:					rp2 ← 226	
        else:						rp2 ← adc2
  else:							rp2 ← reg_data_07 x 4

    // ── Field 12: ss_time[2:0]  (reg_08[6:5], 2 bits) ──
    if mode in {0,1,2,3,7}:
        if ssFix:					ss_time ← 0
        else if ssRail == BELOW_LINEAR:			ss_time ← 0
        else if ssRail == ABOVE_LINEAR:			ss_time ← 0
        else:                                               // LINEAR
		adc3_low ← adc3 and 0x7F 
		adc3_low_inv ← ~adc3 and 0x7F 
		if ssV > 2.5:				ss_time ← floor((adc3_low_inv-20)/8)
		else:					ss_time ← floor((adc3_low-20)/8)
    else if mode in {4,5,6}:				ss_time ← 7   // mode ∈ {4,5,6}
    else						ss_time ← reg_data_08[7:5]

    // ── Field 13: so_set[1:0]  (reg_09[6:5], 2 bits) ──
    so_set ← 0
    switch mode:
        case 0, 2, 7:   so_set ← 3  if adc3 ≥ code(2.5) else 1
        case 1:         so_set ← 3  if adc3 ≥ code(2.5) else 2
        case 3:         so_set ← 3  if adc3 ≥ code(2.5) else 0
        case 4:         so_set ← 0  if adc1 ≥ code(2.5) else 1
        case 5, 6:      so_set ← 3
	default : 	so_set ← reg_data_09[6:5]

    // ── Field 14: auto_soft_sw  (reg_0a[7], 1 bit) ──
    auto_soft_sw ← 1                                         // all modes
	default : 	auto_soft_sw ← reg_data_0a[7]
    // ── Field 15: duty_deb_sw  (reg_0d[7], 1 bit) ──
    if mode in {4,5,6,7}		duty_deb_sw ← 1  
    else				duty_deb_sw ← reg_data_0d[7]

    // ── Field 16: source_duty  (reg_0d[6], 1 bit) ──
    // ADC2 = YP pin. ADC2[7] < 2.5V 等價於 adc2 < 128。
    source_duty ← 0
    if (mode == 0 or mode == 7) and adc2 < 128:		source_duty ← 1
   else							source_duty ← reg_data_0d[6]

    // ── Field 17: align_sw  (reg_0e[7], 1 bit) ──
    align_sw ← 0 
	if mode in {0,1, 2, 3, 4, 5, 6,7}			align_sw ← 1
	else							align_sw ← reg_data_0e[7]

    // ── Field 18: low_freq_rev  (reg_0e[5], 1 bit) ──
    low_freq_rev ← 1                                         // all modes
	else							low_freq_rev ← reg_data_0e[5]

    // ── Field 19: fr_pin_sw  (reg_0f[1], 1 bit) ──
    // ADC1 = XP1 pin. ADC1[7] > 2.5V 等價於 adc1 >= 128。
    if mode in {2,4,6}:						fr_pin_sw ← 1
    else if mode in {3,5}:					fr_pin_sw ← 0
    else if mode in {0,1,7}:					fr_pin_sw ← 0  if adc1 >= 128  else 1// mode ∈ {0,1,7}
    else 							fr_pin_sw ← reg_data_0f[1]
        
	
    return {
        mode, adc1, adc2, adc3,
        cmd_inv, fan_curve, rpm_ratio, min_sd, xp1, soft_sw,
        yp1, yp2, icmd_sw, rp1, rp2, ss_time, so_set,
        auto_soft_sw, duty_deb_sw, source_duty,
        align_sw, low_freq_rev, fr_pin_sw,
    }