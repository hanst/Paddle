# Copyright (c) 2016 PaddlePaddle Authors. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error=pessimizing-move")
endif(APPLE)


set(inference_deps paddle_inference_api paddle_fluid_api analysis pass ir_pass_manager ${GLOB_PASS_LIB})

if(WITH_GPU AND TENSORRT_FOUND)
    set(inference_deps ${inference_deps} paddle_inference_tensorrt_subgraph_engine)
endif()

function(inference_api_test TARGET_NAME)
    if (WITH_TESTING)
        set(options "")
        set(oneValueArgs SRC)
        set(multiValueArgs ARGS)
        cmake_parse_arguments(inference_test "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

        set(PYTHON_TESTS_DIR ${PADDLE_BINARY_DIR}/python/paddle/fluid/tests)
        cc_test(${TARGET_NAME}
                SRCS ${inference_test_SRC}
                DEPS "${inference_deps}"
                ARGS --dirname=${PYTHON_TESTS_DIR}/book/)
        if(inference_test_ARGS)
            set_tests_properties(${TARGET_NAME}
                    PROPERTIES DEPENDS "${inference_test_ARGS}")
        endif()
    endif(WITH_TESTING)
endfunction(inference_api_test)

cc_library(paddle_inference_api SRCS api.cc api_impl.cc helper.cc DEPS lod_tensor)
cc_library(analysis_predictor SRCS analysis_predictor.cc DEPS paddle_inference_api analysis)
cc_test(test_paddle_inference_api
        SRCS api_tester.cc
        DEPS paddle_inference_api)

inference_api_test(test_api_impl SRC api_impl_tester.cc
                    ARGS test_word2vec test_image_classification)

if(WITH_GPU AND TENSORRT_FOUND)
cc_library(paddle_inference_tensorrt_subgraph_engine
        SRCS api_tensorrt_subgraph_engine.cc
        DEPS paddle_inference_api analysis tensorrt_engine paddle_inference_api paddle_fluid_api tensorrt_converter)

inference_api_test(test_api_tensorrt_subgraph_engine SRC api_tensorrt_subgraph_engine_tester.cc ARGS test_word2vec)
endif()

if (WITH_ANAKIN AND WITH_MKL) # only needed in CI
    # compile the libinference_anakin_api.a and anakin.so.
    cc_library(inference_anakin_api SRCS api.cc api_anakin_engine.cc DEPS anakin_shared anakin_saber mklml)
    cc_library(inference_anakin_api_shared SHARED SRCS api.cc api_anakin_engine.cc DEPS anakin_shared anakin_saber)
    function(anakin_target target_name)
      target_compile_options(${target_name} BEFORE PUBLIC ${ANAKIN_COMPILE_EXTRA_FLAGS})
    endfunction()
    anakin_target(inference_anakin_api)
    anakin_target(inference_anakin_api_shared)
    if (WITH_TESTING)
        # TODO(luotao): ANAKIN_MODLE_URL etc will move to demo ci later.
        set(INFERENCE_URL "http://paddle-inference-dist.bj.bcebos.com")
        set(ANAKIN_RNN_MODLE_URL "${INFERENCE_URL}/anakin_test%2Fditu_rnn.anakin2.model.bin")
        set(ANAKIN_RNN_DATA_URL "${INFERENCE_URL}/anakin_test%2Fditu_rnn_data.txt")
        execute_process(COMMAND bash -c "mkdir -p ${ANAKIN_SOURCE_DIR}")
        execute_process(COMMAND bash -c "cd ${ANAKIN_SOURCE_DIR}; wget -q --no-check-certificate ${ANAKIN_RNN_MODLE_URL} -N")
        execute_process(COMMAND bash -c "cd ${ANAKIN_SOURCE_DIR}; wget -q --no-check-certificate ${ANAKIN_RNN_DATA_URL} -N")
        if(WITH_GPU)
            set(anakin_test_extra_deps dynload_cuda)
            set(ANAKIN_MODLE_URL "${INFERENCE_URL}/mobilenet_v2.anakin.bin")
            execute_process(COMMAND bash -c "cd ${ANAKIN_SOURCE_DIR}; wget -q --no-check-certificate ${ANAKIN_MODLE_URL} -N")
            cc_test(api_anakin_engine_tester SRCS api_anakin_engine_tester.cc 
                    ARGS --model=${ANAKIN_SOURCE_DIR}/mobilenet_v2.anakin.bin
                    DEPS inference_anakin_api_shared ${anakin_test_extra_deps} SERIAL)
        endif()
        cc_test(api_anakin_engine_rnn_tester SRCS api_anakin_engine_rnn_tester.cc 
                ARGS --model=${ANAKIN_SOURCE_DIR}/anakin_test%2Fditu_rnn.anakin2.model.bin 
                     --datapath=${ANAKIN_SOURCE_DIR}/anakin_test%2Fditu_rnn_data.txt
                DEPS inference_anakin_api_shared ${anakin_test_extra_deps} SERIAL)
    endif(WITH_TESTING)
endif()
