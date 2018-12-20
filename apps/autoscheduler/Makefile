include ../support/Makefile.inc

BIN ?= bin
HL_TARGET ?= host

WEIGHT_OBJECTS=\
$(BIN)/weights_head1_conv1_weight.o \
$(BIN)/weights_head1_conv1_bias.o \
$(BIN)/weights_head2_conv1_weight.o \
$(BIN)/weights_head2_conv1_bias.o \
$(BIN)/weights_trunk_conv1_weight.o \
$(BIN)/weights_trunk_conv1_bias.o \
$(BIN)/weights_schedule_mean.o \
$(BIN)/weights_schedule_std.o \
$(BIN)/weights_pipeline_mean.o \
$(BIN)/weights_pipeline_std.o \

$(BIN)/weights_%.cpp:
	@mkdir -p $(@D)
	../../bin/binary2cpp weights_$* < weights/$*.data > $@

$(BIN)/weights_%.o: $(BIN)/weights_%.cpp
	$(CXX) -c $< -o $@

COST_MODEL_LIBS=\
$(BIN)/cost_model/cost_model.a \
$(BIN)/cost_model/train_cost_model.a \

$(BIN)/cost_model.generator: cost_model_generator.cpp Derivative.cpp Derivative.h DerivativeUtils.cpp DerivativeUtils.h cost_model_schedule.h $(GENERATOR_DEPS)
	@mkdir -p $(@D)
	$(CXX) $(CXXFLAGS) -g $(filter-out %.h,$^) -o $@ $(HALIDE_SYSTEM_LIBS)

$(BIN)/runtime.a: $(BIN)/cost_model.generator
	@mkdir -p $(@D)
	$^ -r runtime -o $(BIN) target=$(HL_TARGET)

$(BIN)/cost_model/%.a: $(BIN)/cost_model.generator
	@mkdir -p $(@D)
	HL_PERMIT_FAILED_UNROLL=1 $^ -g $* -o $(BIN)/cost_model -f $* -e static_library,h target=$(HL_TARGET)-no_runtime auto_schedule=false

# It's important to use dynamic lookups for undefined symbols here: all of libHalide
# is expected to be present (in the loading binary), so we explicitly make the symbols
# undefined rather than dependent on libHalide.so.
$(BIN)/libauto_schedule.so: AutoSchedule.cpp DefaultCostModel.cpp FunctionDAG.h Featurization.h CostModel.h PerfectHashMap.h $(WEIGHT_OBJECTS) $(COST_MODEL_LIBS) $(GENERATOR_DEPS) $(BIN)/runtime.a
	@mkdir -p $(@D)
	$(CXX) -shared $(USE_EXPORT_DYNAMIC) -fPIC $(CXXFLAGS) -g -I $(BIN)/cost_model AutoSchedule.cpp DefaultCostModel.cpp $(WEIGHT_OBJECTS) $(COST_MODEL_LIBS) $(BIN)/runtime.a -O3 -o $@ $(HALIDE_SYSTEM_LIBS)

$(BIN)/train_cost_model: train_cost_model.cpp DefaultCostModel.cpp CostModel.h $(COST_MODEL_LIBS) $(WEIGHT_OBJECTS) $(BIN)/runtime.a
	$(CXX) $(CXXFLAGS) -I $(BIN)/cost_model -O3 $(filter-out %.h,$^) -o $@ $(LDFLAGS) $(USE_OPEN_MP)

$(BIN)/augment_sample: augment_sample.cpp
	$(CXX) $< -O3 -o $@

# A sample generator to autoschedule. Note that if it statically links
# to libHalide, then it must be build with $(USE_EXPORT_DYNAMIC), or the
# autoscheduler can't find the libHalide symbols that it needs.
$(BIN)/demo.generator: demo_generator.cpp $(GENERATOR_DEPS)
	@mkdir -p $(@D)
	$(CXX) $(CXXFLAGS) $(USE_EXPORT_DYNAMIC) -g $(filter-out %.h,$^) -o $@ $(LDFLAGS) $(HALIDE_SYSTEM_LIBS)

# To use the autoscheduler, set a few environment variables and use the -p flag to the generator to load the autoscheduler as a plugin
$(BIN)/demo.a: $(BIN)/demo.generator $(BIN)/libauto_schedule.so
	HL_MACHINE_PARAMS=32,1,1 HL_PERMIT_FAILED_UNROLL=1 HL_WEIGHTS_DIR=weights \
	$(BIN)/demo.generator -g demo -o $(BIN) -f demo -e static_library,h target=$(HL_TARGET) auto_schedule=true -p $(BIN)/libauto_schedule.so

$(BIN)/demo.rungen: $(BIN)/RunGenMain.o $(BIN)/demo.a $(HALIDE_DISTRIB_PATH)/tools/RunGenStubs.cpp
	$(CXX) -I $(BIN) -DHL_RUNGEN_FILTER_HEADER="\"demo.h\"" $^ -o $@ $(HALIDE_SYSTEM_LIBS) $(IMAGE_IO_FLAGS)

# demonstrates single-shot use of the autoscheduler
demo: $(BIN)/demo.rungen
	$(BIN)/demo.rungen --benchmarks=all --benchmark_min_time=1 --default_input_buffers=random:0:auto --output_extents=estimate --default_input_scalars=estimate

# demonstrates an autotuning loop
autotune: $(BIN)/demo.generator $(BIN)/augment_sample $(BIN)/train_cost_model autotune_loop.sh $(BIN)/libauto_schedule.so
	bash autotune_loop.sh $(BIN)/demo.generator demo x86-64-avx2-disable_llvm_loop_unroll-disable_llvm_loop_vectorize

# Simple jit-based test
$(BIN)/test: test.cpp $(BIN)/libauto_schedule.so
	$(CXX) $(CXXFLAGS) -rdynamic $< -o $@ $(LDFLAGS) $(LIB_HALIDE) $(HALIDE_SYSTEM_LIBS)

test: $(BIN)/test
	HL_WEIGHTS_DIR=weights LD_LIBRARY_PATH=$(BIN) $(BIN)/test

clean:
	rm -rf $(BIN)

