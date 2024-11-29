add_library(pico_fractional_pll INTERFACE)

target_sources(pico_fractional_pll INTERFACE
    ${CMAKE_CURRENT_LIST_DIR}/pico_fractional_pll.c
    )

target_include_directories(pico_fractional_pll INTERFACE ${CMAKE_CURRENT_LIST_DIR})
target_link_libraries(pico_fractional_pll INTERFACE
    hardware_exception
    hardware_pwm
    hardware_irq
    pico_multicore)
