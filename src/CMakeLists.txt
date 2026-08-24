add_library(allocators INTERFACE)

target_include_directories(allocators INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}
)

target_sources(allocators INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/pool_allocator.hpp
    ${CMAKE_CURRENT_SOURCE_DIR}/arena_allocator.hpp
    ${CMAKE_CURRENT_SOURCE_DIR}/stack_allocator.hpp
    ${CMAKE_CURRENT_SOURCE_DIR}/slab_allocator.hpp
    ${CMAKE_CURRENT_SOURCE_DIR}/buddy_allocator.hpp
    ${CMAKE_CURRENT_SOURCE_DIR}/freelist_allocator.hpp
    ${CMAKE_CURRENT_SOURCE_DIR}/ring_buffer.hpp
)