.PHONY: all clean sim_counter

SERV_SRC ?= ./serv
LIBS_74 ?= ../74ac.lib ../74_models.v ../74_adder.v ../74_dffe.v ../74_mux.v ../synth_74.ys
#VPR = ../../vtr-verilog-to-routing/vpr/vpr
VPR ?= vpr

all: out/serv.place out/serv_vpr_generated.kicad_pcb

out/%.json: ../benchmarks/%.v $(LIBS_74)
	yosys -q -s ../synth_74.ys -p "write_json $@" $<

out/serv.json out/serv.eblif: $(SERV_SRC)/rtl/*.v serv_rf_top_no_ext.v $(LIBS_74)
	# We omit -cname and -iname because the output nets actually have more useful names.
	yosys -q -r serv_rf_top_no_ext -s ../synth_74.ys -p "write_json $@; write_blif -conn out/serv.eblif" $(SERV_SRC)/rtl/*.v serv_rf_top_no_ext.v

out/%.net: out/%.json generate_netlist.py parts.py
	python3 generate_netlist.py $<

clean:
	rm -f out/ *.erc *.log *.rpt

out/serv.place: out/serv.eblif 74xx_fpga.xml
	sed -E <out/serv.eblif >out/serv2.eblif 's/( \S+=|^[.]conn +)[$$](false|true|undef)/\1unconn/g'
	rm -f out/serv2.place out/serv.place out/serv_vtr_place*.png
	mkdir -p out
	#FIXME If we don't let vpr route, will that make the placement worse? Auto-router seems to have a harder time, now.
	# --pack --place
	# --clustering_pin_feasibility_filter off  # not necessary anymore
	# --pack_verbosity 40
	$(VPR) 74xx_fpga.xml out/serv2.eblif \
		--device hans4 \
		--netlist_verbosity 20 --pack_verbosity 4 \
		--alpha_clustering 0.0 --beta_clustering 0.0 --allow_unrelated_clustering on \
		--balance_block_type_utilization off \
		--graphics_commands "set_draw_block_internals 1; set_nets 2; set_macros 1; save_graphics out/serv_vtr_place{i}.png" \
		--timing_analysis off \
		--outfile_prefix out/ \
		|& tee out/serv_vpr.log
	mv out/serv2.place out/serv.place

out/serv_vpr_generated.kicad_net out/serv_vpr_generated.kicad_pcb: out/serv.json out/serv.place out/serv2.net generate_netlist2.py generate_netlist.py parts.py serv_vpr_template.kicad_*
	cp serv_vpr_template.kicad_pro out/serv_vpr_generated.kicad_pro
	python3 generate_netlist2.py out/serv_vpr_generated out/serv.json out/serv2.net out/serv.place serv_vpr_template.kicad_pcb 27x19+1+0

.PHONY: serv-freerouting
serv-freerouting: out/serv_vpr_generated.kicad_pcb
	freerouting -de out/serv_vpr_generated.dsn
