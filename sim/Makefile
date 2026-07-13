# floo_axi_chimney DV 仿真入口（Synopsys VCS/VCS_MX + Verdi 2018，UVM-1.2）
# 常用:
#   make smoke                          冒烟（M0=上游 SV tb 排雷；M1 起切换为 UVM smoke）
#   make run TEST=<test> SEED=<n>       单测（FSDB=1 抓波形，COV=1 收覆盖率）
#   make regress [COV=1]                一键回归 → sim/result_summary.txt
#   make cov                            urg 覆盖率报告（out/urgReport）
#   make lint                           轻量 lint（VCS +lint，判定范围限 ../tb/）
#   make verdi                          看波形
.PHONY: compile run smoke sanity regress cov lint verdi clean

# EDA 环境兜底（非交互 shell 不 source ~/.bashrc；此处与 VM 部署一致，可被外部覆盖）
export VCS_HOME     ?= /home/synopsys/vcs-mx/O-2018.09-SP2
export VCS_MX_HOME  ?= $(VCS_HOME)
export VERDI_HOME   ?= /home/synopsys/verdi/Verdi_O-2018.09-SP2
export SCL_HOME     ?= /home/synopsys/scl/2018.06
export LM_LICENSE_FILE ?= 27000@icarray-virtual-machine
export VCS_ARCH_OVERRIDE ?= linux
export LD_LIBRARY_PATH := $(VERDI_HOME)/share/PLI/VCS/LINUX64:$(LD_LIBRARY_PATH)
export PATH := $(VCS_HOME)/bin:$(VERDI_HOME)/bin:$(SCL_HOME)/linux64/bin:$(PATH)

TEST ?= chimney_smoke_test
SEED ?= 1
VERB ?= UVM_MEDIUM
COV  ?= 0
FSDB ?= 0
OUT  ?= out

VCS      := vcs
# M0 阶段仅编 vendor+dut+上游 tb；M1 起追加 -f flist/tb.f（UVM 环境）
FLISTS   := -f flist/vendor.f -f flist/dut.f -f flist/tb_upstream.f
# 本 VM 适配：Ubuntu 22.04 的 g++ 默认 -Wl,--as-needed 会剔除 libvcsnew.so 依赖的
# snps*/vfs_* 库导致链接失败（VCS 2018 官方支持的老发行版无此默认），必须显式关闭
LD_FIX   := -LDFLAGS "-Wl,--no-as-needed"
# Verdi PLI（novas.tab + pli.a）：注册 $fsdbDump* 系统任务，FSDB 抓波形必需
NOVAS    := -P $(VERDI_HOME)/share/PLI/VCS/LINUX64/novas.tab \
            $(VERDI_HOME)/share/PLI/VCS/LINUX64/pli.a
# -assert svaext：pulp 库大量使用 elaboration system task（$error/$fatal 于 generate 内）
VCS_OPTS := -full64 -sverilog -timescale=1ns/1ps -ntb_opts uvm-1.2 \
            -assert svaext \
            -debug_access+all -kdb +vcs+lic+wait $(LD_FIX) $(NOVAS) \
            -o $(OUT)/simv -l $(OUT)/comp.log
SIM_OPTS := +UVM_TESTNAME=$(TEST) +ntb_random_seed=$(SEED) +UVM_VERBOSITY=$(VERB) \
            -l $(OUT)/$(TEST)_$(SEED).log

# 覆盖率口径：六类 line+cond+fsm+tgl+branch+assert，≥90% 合格
ifeq ($(COV),1)
CM       := -cm line+cond+fsm+tgl+branch+assert -cm_dir $(OUT)/cov.vdb
VCS_OPTS += $(CM)
SIM_OPTS += $(CM) -cm_name $(TEST)_$(SEED)
endif

ifeq ($(FSDB),1)
VCS_OPTS += +define+DUMP_FSDB
endif

compile:
	@mkdir -p $(OUT)
	$(VCS) $(VCS_OPTS) $(FLISTS) -top tb_floo_axi_chimney

run: compile
	$(OUT)/simv $(SIM_OPTS)

# M0 sanity：上游 SV tb（背靠背双 chimney，axi_test 随机读写 + 保序比对）。
# 上游 tb 以 $stop 结束，批处理下用 ucli 脚本收尾避免挂在交互提示符。
sanity: compile
	echo "run; exit" > $(OUT)/sanity.ucli
	$(OUT)/simv -ucli -i $(OUT)/sanity.ucli -l $(OUT)/upstream_sanity_1.log

smoke: sanity

regress:
	@python3 ../scripts/regress.py regress/regress.list $(if $(filter 1,$(COV)),COV=1)

cov:
	urg -full64 -dir $(OUT)/cov.vdb -report $(OUT)/urgReport
	@echo "覆盖率报告: sim/$(OUT)/urgReport"

# 轻量 lint：判定范围限 ../tb/（本仓库交付物）。vendor RTL 与 UVM 库的告警不受
# 本仓库控制、不计入判定（原始完整 log 仍写入 $(OUT)/lint.log 供查阅）。
lint:
	@mkdir -p $(OUT)
	$(VCS) -full64 -sverilog -timescale=1ns/1ps -ntb_opts uvm-1.2 +vcs+lic+wait $(LD_FIX) \
	       +lint=all,noVCDE $(FLISTS) -o $(OUT)/simv_lint -l $(OUT)/lint.log
	@if grep -A1 '^Lint-\[' $(OUT)/lint.log | grep -qE '^\.\./tb/'; then \
	    grep -A1 '^Lint-\[' $(OUT)/lint.log | grep -B1 -E '^\.\./tb/'; \
	    echo "lint 有告警（../tb/ 范围内）：修复，或登记 doc/lint-waivers.md 后由 rev 复核"; exit 1; \
	else echo "lint 干净（../tb/ 范围内 0 告警）: sim/$(OUT)/lint.log"; fi

verdi:
	verdi -ssf $(OUT)/wave.fsdb &

clean:
	rm -rf $(OUT) csrc simv* *.daidir ucli.key vc_hdrs.h novas.* verdiLog nWaveLog
