include(GoogleTest)

set(ALLOCATOR_TESTS
    pool_allocator_test
    arena_allocator_test
    stack_allocator_test
    slab_allocator_test
    buddy_allocator_test
    freelist_allocator_test
    ring_buffer_test
)

foreach(test_name ${ALLOCATOR_TESTS})
    add_executable(${test_name} ${test_name}.cpp)
    target_link_libraries(${test_name} PRIVATE allocators GTest::gtest_main)
    gtest_discover_tests(${test_name})
endforeach()