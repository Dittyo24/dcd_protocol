file(GLOB HEADERS "include/dcd/txn_test_gen_plugin/*.hpp")
add_library( txn_test_gen_plugin
             txn_test_gen_plugin.cpp
             ${HEADERS} )
             
target_link_libraries( txn_test_gen_plugin appbase fc http_plugin chain_plugin dcd_testing_contracts )
target_include_directories( txn_test_gen_plugin PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
                            txn_test_gen_plugin PUBLIC ${CMAKE_SOURCE_DIR}/libraries/testing/include
                            txn_test_gen_plugin PUBLIC ${CMAKE_BINARY_DIR}/unittests/include )
