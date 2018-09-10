# perl-upf_gen
#perl in upf flow. This script is used to translate the excel which contains the necessary information for upf flow to upf file.

#!/usr/bin/perl
$in=$ARGV[0];
if(!defined $in) {
    die "Usage:$0 filename";
}

my $out=$in;
$out=~s/(\.\w+)?$/.upf/;

if(!open $in_fh, '<',$in) {
    die "cannot open '$in':$!";
}

if(!open $out_fh, '>',$out) {
    die "cannot open '$out':$!";
}

my @pd = ();
my @pst = ();
my $pst_tmp = ();
my @supply_net = ();
my @supply_port = ();
my $pst_name;

while(<$in_fh>)  {
    chomp;
    my @line = split(',',$_);

    if($line[0] =~ /^\s*PD_/) {
        push @pd, $line[0];
        @pd_ele = split(' ',$line[1]);
        my $ele_len = @pd_ele;
        if($ele_len > '1') {
            foreach $ele (@pd_ele) {
                push @pd_element, $ele." \\\n"; 
            }
        }
        else {
                push @pd_element, $pd_ele[0]; 
             }
        print $out_fh "create_power_domain ".$line[0]." -elements {@pd_element}\n";
        @pd_element = ();
        print $out_fh "set_domain_supply_net ".$line[0]." -primary_power_net ".$line[3]." -primary_ground_net ".$line[4]."\n";
        print $out_fh "\n";
        if($line[5] =~ /yes/) {
            print $out_fh "create_power_switch ".lc($line[0])."_sw"." -domain $line[0] \\"."\n";
            print $out_fh "-input_supply_port {switch_in $line[6]} \\"."\n";
            print $out_fh "-output_supply_port {switch_out $line[7]} \\"."\n";
            print $out_fh "-control_port {NSleep $line[8]} \\"."\n";
            print $out_fh "-ack_port {switch_ack $line[9]} \\"."\n";
            print $out_fh "-on_state {on_state switch_in $line[10]}\n";
        }
        print $out_fh "\n";
        if($line[11] ne "no") {
            my @ls = split(';',$line[11]);
            if($ls[0] =~ /I/)  {
                if($ls[1] =~ /B/)       {$rule_i = "both";}
                elsif($ls[1] =~ /HL/)   {$rule_i = "high_to_low";}
                elsif($ls[1] =~ /LH/)   {$rule_i = "low_to_high";}
                if($ls[2] =~ /P/)       {$location_i = "parent";}
                elsif($ls[2] =~ /S/)    {$location_i = "self";}
                elsif($ls[2] =~ /A/)    {$location_i = "automatic"}
                print $out_fh "set_level_shifter ".lc($line[0]."_ls_i "."-domain $line[0] "."-applies_to inputs -rule $rule_i -location $location_i\n");
            }
            if($ls[3] =~ /O/) {
                if($ls[4] =~ /B/)       {$rule_o = "both";}
                elsif($ls[4] =~ /HL/)   {$rule_o = "high_to_low";}
                elsif($ls[4] =~ /LH/)   {$rule_o = "low_to_high";}
                if($ls[5] =~ /P/)       {$location_o = "parent";}
                elsif($ls[5] =~ /S/)    {$location_o = "self";}
                elsif($ls[5] =~ /A/)    {$location_o = "automatic";}
                print $out_fh "set_level_shifter ".lc($line[0]."_ls_o "."-domain $line[0] "."-applies_to outputs -rule $rule_o -location $location_o\n");
            }
        }
        print $out_fh "\n";
        if($line[12] eq "yes")  {
            my $iso_pow = $line[13];
            my $iso_ground = $line[14];
            my $clamp_value = $line[15];
            my @iso_elem = split(' ',$line[16]);
            my $iso_len = @iso_elem;
            if($iso_len > '1') {
                foreach $ele (@iso_elem) {
                    push @iso_element,$ele." \\\n";
                }
            }
            else {
                push @iso_element,$iso_elem[0];
            }
            my @iso_ctl = split(';',$line[17]);
            my $iso_cell = $line[18];
            print $out_fh "set_isolation ".lc($line[0])."_iso "."-domain $line[0] \\\n";
            print $out_fh "-isolation_power_net $iso_pow \\\n";
            print $out_fh "-isolation_ground_net $iso_ground \\\n";
            print $out_fh "-clamp_value $clamp_value \\\n";
            print $out_fh "-elements {@iso_element}\n";
            print $out_fh "\n"; 
            print $out_fh "set_isolation_control ".lc($line[0])."_iso "."-domain $line[0] \\\n";
            print $out_fh "-isolation_signal {$iso_ctl[0]} \\\n";
            print $out_fh "-isolation_sense $iso_ctl[1] \\\n";
            print $out_fh "-location $iso_ctl[2]\n";
            print $out_fh "\n";
            print $out_fh "map_isolation_cell ".lc($line[0])."_iso "."-domain $line[0] \\\n";
            print $out_fh "-lib_cells {$iso_cell}\n"
        }
        print $out_fh "\n";
        @supply_net = split(' ',$line[19]);
        foreach $ele (@supply_net) {
            my @reuse = split('\.',$ele);
                if($reuse[1] eq 'r') {
                    print $out_fh "create_supply_net ".$reuse[0]." -domain $line[0] -reuse\n";
                }
                else {
                    print $out_fh "create_supply_net ".$reuse[0]." -domain $line[0]\n";
                }
        }
        @supply_port = split(' ',$line[20]);
        foreach $ele (@supply_port) {
            print $out_fh "create_supply_port $ele -domain $line[0]\n";
        }
        my $i = 0;
        foreach $ele (@supply_port) {
            print $out_fh "connect_supply_net $supply_net[$i] -ports $ele\n";
            $i = $i + 1;
        }
        print $out_fh "\n";
    } 

    elsif($line[0] =~ /^\s*VDD/) {
        print $out_fh "add_port_state $line[0] \\\n";
        my @pow = split(' ',$line[1]);
        foreach $ele (@pow) {
                @state = split('\.',$ele);
                print $out_fh "-state {".$line[0].$state[0]."p".$state[1]." $ele} \\\n";
            } 
        }
    elsif($line[0] =~ /^\s*VSS/) {
        print $out_fh "add_port_state $line[0] \\\n";
        print $out_fh "-state {GND 0.00}\n";
    }
    elsif($line[0] =~ /^\s*PST/) {
        print $out_fh "\n";
        $pst_name = $line[0];
        foreach $ele (@line) {
            if(!($ele =~ /PST/) && ($ele ne '')) {
                push @pst_tmp, $ele;
                push @pst, $ele." ";
            }
        }
        print $out_fh "create_pst $line[0] -supplies {@pst}\n"; 
    }
    elsif($line[0] =~ /^\s*Con/) {
        my $i = 0;
        foreach $ele (@line) {
            if(!($ele =~ /Con/) && ($ele ne '')) {
                if($pst_tmp[$i] ne "VSS") {
                @pst_vol = split('\.',$ele);
                push @pst_st, $pst_tmp[$i].$pst_vol[0]."p".$pst_vol[1]." ";
                $i = $i + 1;
                }
            else {
                push @pst_st, "GND ";
                }
            }
        } 
    print $out_fh "add_pst_state $line[0] -pst $pst_name -state {@pst_st}\n";   
    @pst_st = ();
    }
    else {
    next;
    }
}



#######################################################################################################################################

power_domain	element	power_src	power_prim	ground_prim	power_switch	sw_in_port	sw_out_port	sw_ctl_port	sw_ack_port	sw_on_state	level_shifter	iso	iso_power	iso_ground	clamp_value	iso_elem	iso_ctl	iso_cell	supply net	supply port
PD_SYS		VDD	VDD	VSS	no						no	no							VDD VDDR VDDI VDDT VDDS VDDD VSS	VDD VDDR VDDI VDDT VDDS VDDD VSS
PD_IRX	gmr_bp_mid_inst/gmr_bp_core_inst/irx_asip_inst	VDDI	VIR_VDDI	VSS	yes	VDDI	VIR_VDDI	gmr_bp_mid_inst/lpcu_inst/sw_ctrl_rx	gmr_bp_mid_inst/gmr_bp_core_inst/irx_asip_inst/sw_ack	!NSleep	I;B;P;O;B;P	yes	VDDI	VSS	0	gmr_bp_mid_inst/gmr_bp_core_inst/irx_asip_inst/sem_out gmr_bp_mid_inst/gmr_bp_core_inst/irx_asip_inst/hdi	gmr_bp_mid_inst/lpcu_inst/isolate_ctrl_rx;low;parent	A2LVLD_X4M_A9TR40 A2ISO_*	VDDI.r VIR_VDDI VSS.r	
PD_TX	gmr_bp_mid_inst/gmr_bp_core_inst/tx_asip_inst	VDDT	VIR_VDDT	VSS	yes	VDDT	VIR_VDDT	gmr_bp_mid_inst/lpcu_inst/sw_ctrl_tx	gmr_bp_mid_inst/gmr_bp_core_inst/tx_asip_inst/sw_ack	!NSleep	I;B;P;O;B;P	yes	VDDT	VSS	0	gmr_bp_mid_inst/gmr_bp_core_inst/tx_asip_inst/sem_out gmr_bp_mid_inst/gmr_bp_core_inst/tx_asip_inst/hdi	gmr_bp_mid_inst/lpcu_inst/isolate_ctrl_tx;low;parent	A2LVLD_X4M_A9TR40 A2ISO_*	VDDT.r VIR_VDDT VSS.r	
PD_KGR	gmr_bp_mid_inst/gmr_bp_core_inst/kgr_inst	VDD	VIR_VDDK	VSS	yes	VDD	VIR_VDDK	gmr_bp_mid_inst/lpcu_inst/sw_ctrl_kgr	gmr_bp_mid_inst/gmr_bp_core_inst/kgr_inst/sw_ack	!NSleep	I;HL;P;O;LH;P	yes	VDD	VSS	0	gmr_bp_mid_inst/gmr_bp_core_inst/kgr_inst/oSerial_i1 gmr_bp_mid_inst/gmr_bp_core_inst/kgr_inst/oSerial_i0	gmr_bp_mid_inst/lpcu_inst/isolate_ctrl_kgr;low;parent	A2LVLU_X4M_A9TR40 A2LVLUO_X4M_A9TR40 A2ISO_*	VDD.r VIR_VDDK VSS.r	
PD_SP	gmr_bp_mid_inst/gmr_bp_core_inst/sp_top_inst	VDDS	VDDS	VSS	no						I;B;P;O;B;P								VDDS.r VSS.r	
PD_DDR	gmr_bp_mid_inst/gmr_bp_core_inst/u_gmr_bp_lpddr2	VDDD	VDDD	VSS	no						I;HL;P;O;LH;P	yes	VDD	VSS	0	gmr_bp_mid_inst/gmr_bp_core_inst/u_gmr_bp_lpddr2/ex_gmr_lpddr2_i_axi_AXI_Master01_arready gmr_bp_mid_inst/gmr_bp_core_inst/u_gmr_bp_lpddr2/ex_gmr_lpddr2_i_axi_AXI_Master01_awready 	gmr_bp_mid_inst/lpcu_inst/isolate_ctrl_lpddr;low;parent	A2LVLU_X4M_A9TR40 A2LVLUO_X4M_A9TR40 A2ISO_*	VDDD.r VSS.r	
PD_RTC	gmr_bp_mid_inst/u_rtc pad_RSTN_RTC_IN gmr_bp_mid_inst/u_rst_gen/u_rstn_rtc_sync gmr_bp_mid_inst/u_rst_gen/u_rstn_rtc_deglitch pad_EXT_WAKEUP_IN pad_CKI_32K pad_PWR_EN pad_TEST_EN	VDDR	VDDR	VSS	no						I;HL;P;O;LH;P	yes	VDDR	VSS	0	gmr_bp_mid_inst/u_rtc/standbywfi  gmr_bp_mid_inst/u_rtc/pclk	gmr_bp_mid_inst/lpcu_inst/isolate_ctrl_rtc;low;parent	A2LVLU_X4M_A9TR40 A2LVLUO_X4M_A9TR40 A2ISO_*	VDDR.r VSS.r	
PD_AP_BP_INF	gmr_bp_mid_inst/gmr_bp_core_inst/u_gmr_usb_otg	VDD	VIR_VDDU	VSS	yes	VDD	VIR_VDDU	gmr_bp_mid_inst/lpcu_inst/sw_ctrl_usb	gmr_bp_mid_inst/gmr_bp_core_inst/u_gmr_usb_otg/sw_ack	!NSleep	I;HL;P;O;LH;P	yes	VDD	VSS	0	gmr_bp_mid_inst/gmr_bp_core_inst/u_gmr_usb_otg/i_otg_intr_irq gmr_bp_mid_inst/gmr_bp_core_inst/u_gmr_usb_otg/otg_hmx_axi_master_araddr	gmr_bp_mid_inst/lpcu_inst/isolate_ctrl_usb;low;parent	A2LVLU_X4M_A9TR40 A2LVLUO_X4M_A9TR40 A2ISO_*	VDD.r VIR_VDDU VSS.r	
#############																				
power_port	port_state																			
VDD	0.99 0																			
VDDR	0.99																			
VDDI	0.9 1.26 0																			
VIR_VDDI	0.9 1.26 0																			
VDDT	0.9 1.26 0																			
VIR_VDDT	0.9 1.26 0																			
VIR_VDDK	0.99 0																			
VDDS	0.99 1.26 0																			
VDDD	0.99 0																			
VIR_VDDU	0.99 0																			
VSS	0																			
#############																				
PST_TOP	VDD	VDDR	VDDI	VIR_VDDI	VDDT	VIR_VDDT	VIR_VDDK	VDDS	VDDD	VIR_VDDU	VSS									
Con3_LP	0.99	0.99	1.26	0	0.9	0.9	0	1.26	0.99	0	0									
Con1_HP	0.99	0.99	0.9	0	1.26	0	0.99	1.26	0.99	0.99	0									
Con2_HP	0.99	0.99	1.26	1.26	0.9	0.9	0	0.9	0.99	0.99	0									
Con3_HP	0.99	0.99	0.9	0.9	1.26	1.26	0.99	0.9	0.99	0.99	0									
Con1_OFF	0	0.99	0	0	0	0	0	0	0	0	0									
Con1_LP	0.99	0.99	1.26	1.26	0.9	0	0.99	0.9	0	0.99	0									
Con2_LP	0.99	0.99	0.9	0.9	1.26	1.26	0.99	1.26	0	0	0									






